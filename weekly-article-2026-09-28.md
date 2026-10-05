# `SessionManager`: the keeper of the hotel’s keys

## 1. Where it is, what it is for, who uses it

In ARA an agent does not carry state on itself: `AgentInstance` is **stateless with respect to execution**. Conversations, memory, state machine, and history live inside an **`AgentSession`** — a separate room for each `SessionId`. It is `SessionManager` (`ara-runtime`, package `io.ara.runtime.agent`) that decides **who enters which room, and when the room is closed**.

**Who uses it?** Not the user directly: it is `AgentInstance` that calls it automatically, **on every single task**. The user only touches it indirectly, via `AgentConfig.sessionTtl()` (30 minutes by default) and `listActive()` for monitoring.

**Why is it needed?** An occupied room costs twice: memory and **resources**. Each session “pins” an `AgentWiring` — an already-resolved LLM client and tool registry, plus any MCP connections. Without a custodian, rooms accumulate and connections stay open.

## 2. A metaphor to keep track

A **hotel**. Each room has a key with a tag listing the utilities already hooked up. The `SessionManager` is the **receptionist**: it hands out the key and notes on a sheet which room is occupied and for how long. It walks the corridors and **takes back the keys of departed guests** (the TTL). When the hotel closes, it returns all the keys and disconnects the utilities.

## 3. The flow: what goes in, what happens, what comes out

**Input.** A `SessionId` plus the configuration **snapshot**, read only once by the caller. And two implicit inputs: the TTL, and the **sweeper tick** (the periodic task that calls `evictStale()`).

**Process.** `getOrCreate` puts an **empty skeleton** in the map, and inserts an `AgentSession` into it only if one already exists. On the first pass it builds the wiring *outside* the map, under its own lock, and resets the inactivity clock. The sweeper reads the clocks, collects expired keys, and **starts each eviction on a dedicated virtual thread**, rechecking that the room is still free before snatching it from the guest’s hands.

**Output.** The `AgentSession` — memory, state machine, history, wiring frozen at birth — and for observability `listActive()`: `session_id`, `turn_count`, `ttl_remaining_ms`, `state`.

## 4. The choices, and the reasons why

**Construction stays outside `computeIfAbsent`.** It was in there and it worked, but `ConcurrentHashMap` holds its internal lock for the entire function, and a virtual thread blocked on real I/O inside that lock *blocks its own carrier* on JDK 21 (JEP 491 removes this in JDK 24). And that lock belongs to the JDK, not to us: there is nothing to replace. The solution is structural — a trivial skeleton in the map, construction afterward, under its own `ReentrantLock`: it parks instead of pinning, multiple callers on the same id construct only once, and two different ids never contend with each other.

**`nanoTime` for the TTL, the wall clock only to display it.** This way an NTP adjustment cannot make a live session look dead. And the TTL measures **idle** time, not total lifetime.

**The sweeper cannot block.** Closing the wiring is real I/O (an MCP connection closing): a stuck closure would have blocked all the other rooms **and the next tick**, because `scheduleAtFixedRate` runs the task serially on a single thread — the sweeper would remain lame for the rest of the JVM’s life. For this reason each eviction starts on a dedicated virtual thread, and `sweep()` has a `catch-all`: if the task throws, the scheduler **silently cancels all future executions**.

**Removal from the map is the only ownership transfer.** Whoever sees `remove` return non-null closes the wiring, the others do not — two teardowns cannot close twice. For the same reason, a safety `clear()` was **removed** from `shutdown()`: it could delete the entry from under the caller’s `invalidate`, which found nothing and closed nothing — a leak *caused* by the `clear` instead of avoided.

---

*Class: `io.ara.runtime.agent.SessionManager` — `ara-runtime`. Idle TTL: `AgentConfig.sessionTtl()`, default 30 minutes.*
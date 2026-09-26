# `AgentDelegationTool`: when one agent asks a colleague for help

## 1. Where it lives, what it does, and who uses it

Imagine ARA as an office full of agents. Every agent has its own workstation: a system prompt, a set of tools, and its own memory. Up to a certain point, however, every agent worked alone, isolated in its own room.

`AgentDelegationTool` (`io.ara.runtime.bus`, module `ara-runtime`) is the office's internal phone system.

It is a regular `AraTool`—the same contract used for a web search or a calculator—with the tool ID `delegate_task`. If you include this string in an agent's `enabledTools` list, the language model driving that agent can decide on its own to call a colleague instead of solving the task by itself.

Who uses it? Not the user. It is used by the LLM, in the middle of its reasoning process.

You do not have to wire it manually. `AraRuntime` automatically installs it for every agent it creates by wrapping your tool registry with `DelegatingToolRegistry` and injecting the `LocalMessageBus`.

If you are building your own registry from scratch, the wiring is a single line:

Java

```
new DelegatingToolRegistry(base, messageBus, agentId)
```

## 2. The flow: what goes in, what happens, what comes out

Input. There are two inputs, not one.

The explicit input is a JSON payload with two fields:

JSON

```
{
  "agent_id": "kb-agent",
  "task": "How does ARA handle concurrency?"
}
```

The implicit input is the caller's `AgentTask`, which carries the caller's execution context and session. That hidden context is what makes delegation safe, as explained below.

Process. The actual work is a pipeline of four filters, executed in order:

1. Parsing and validation.

   * Malformed JSON → failure.

   * Delegating to yourself → explicit failure, preventing infinite self-delegation loops.

2. Recipient existence. If an `AgentView` is available, the tool first checks whether the target agent is visible and authorized for this caller. There is no point sending a letter if the recipient does not exist.

   (In the default `AraRuntime` wiring, this pre-check is disabled because the message bus always performs the authoritative validation.)

3. Permission attenuation. This is the core of the design (see §4). The tool prepares the execution context that the recipient will inherit.

4. Delivery. The tool builds an `AgentMessage` and calls `bus.request(...)`.

   The caller blocks on a virtual thread until the colleague replies or the request times out (120 seconds by default).

Output. The tool returns a `ToolResult`.

The ReAct execution loop does not treat that result as a terminal event. Instead, it records it as an observation and feeds it into the next reasoning step.

A successful delegation is tagged with the responding agent's identity:

```
[kb-agent] ...
```

If the delegated task fails, the delegation itself is still considered successful: the `ToolResult` contains the error text returned by the colleague. The LLM can read that observation and decide to retry or choose another strategy instead of giving up.

If the delegation itself fails, the observation contains the delegation failure reason. In strategies such as ReflAct, a failed tool observation can trigger an explicit reflection step.

## 3. The mental model

* `MessageBus` is the internal courier. It knows every office and delivers messages between agents.

* Scope intersection is like photocopying someone's key after filing down some of its teeth. The copy can open no more doors than the original key.

* `DelegateStateAccess` decides which whiteboard the recipient receives:

  * `SHARED`: the exact same whiteboard, with shared reads and writes.

  * `OVERLAY` (default): the recipient sees the shared whiteboard plus a private notebook where its own writes remain private.

  * `ISOLATED`: a sealed envelope with no shared mutable state.

* The timeout means you do not stay on the phone forever. If the colleague does not answer, the caller moves on.

## 4. The design decisions — and why they look this way

### Delegation is a tool, not a new API

Because the ReAct loop already knows how to invoke tools, wait for their result, and consume their observations.

Delegation introduces no new control flow. It simply reuses the execution model that already exists.

### Failures are values, not exceptions

Because ReAct is designed to consume tool failures as observations and continue reasoning.

The three `catch` blocks inside `dispatch` are intentionally distinct:

* the target agent does not exist,

* the target agent is busy,

* the delegation itself failed.

Each tells the model something different about what it should try next.

### Permissions shrink by construction, not by runtime checks

The recipient receives:

effectiveScopes=incomingScopes∩recipientScopes\text{effectiveScopes} = \text{incomingScopes} \cap \text{recipientScopes}effectiveScopes=incomingScopes∩recipientScopes

An intersection can never be larger than either operand.

That means a delegation chain A → B → C can only lose permissions, never gain them. There is no runtime `if` that could accidentally be skipped.

The same rule applies to the `ExecutionContext`:

* the subject (the original user identity) remains unchanged,

* only the actor becomes more restricted at each hop.

And the caller makes the delegation decision—not the callee. A subordinate agent can never grant itself permissions it did not already receive.

### Delegated sessions are derived, not random

The delegated session ID is built as:

```
<caller-session>::deleg::<recipient-id>
```

As a result:

* repeated delegations to the same colleague reuse the same delegated session,

* the colleague's direct sessions remain separate,

* delegations to different colleagues never collide.

This naming scheme composes naturally across delegation chains.

### There are two visibility checks, not one

The optional `AgentView` pre-check is a cheap front door that avoids attempting deliveries that are obviously invalid.

The message bus still performs the authoritative authorization check when the message is actually delivered.

### `attenuateForHops` is pure and isolated

The entire security surface of delegation lives inside one dedicated method.

A reviewer can read `attenuateForHops` from top to bottom and understand exactly how permissions, execution context, and session identity are transformed, instead of chasing security logic scattered throughout the execution path.

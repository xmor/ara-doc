# `AraRuntime`: The Hub That Keeps Every Agent Running

## 1. Where it lives, what it does, and who uses it

In ARA, every agent needs a central hub to coordinate its pieces: the model (LLM), the tools it can call, memory, scheduling, permissions, and how tasks flow between them. **`AraRuntime`** (`ara-runtime`, package `io.ara.runtime`) is that hub: the top-level entry point for running ARA in standalone mode. You build it once with a fluent **Builder**, call `start()`, create agents from it, run tasks, and eventually `stop()` or `close()` it. Everything the agents need is wired together behind the scenes.

**Who uses it?** Application code — your programs, demos, services — bootstraps the system through it. The agents themselves never construct it: they *receive* their dependencies through the wiring it provides. It is the engine room powering every agent created from it.

**Why is it needed?** Without it you would have to connect LLM clients, tool registries, memory, schedulers, telemetry and security by hand. `AraRuntime` does that wiring in a consistent, tested way, so agents can focus on their work.

## 2. An analogy to keep things clear

Think of **a power plant control room**. The control room does not generate electricity itself: it connects the generators, fuel lines, safety systems, monitoring equipment and the grid, making sure the right resources flow to the right places at the right time — and it has emergency shutoffs and monitoring dashboards.

`AraRuntime` is the control room for AI agents: it connects the "brain" (the LLM), the "hands" (the tools), the "memory" (conversations and knowledge), the "calendar" (the scheduler) and the "security guard" (permissions and HITL approvals). You would not run a power plant by plugging wires at random; do not ad-hoc wire an agent system either — build one central control room.

## 3. The flow: what goes in, what happens, and what comes out

**Input.** Configuration, provided through `AraRuntime.builder()`: an LLM client, a tool registry, a memory manager, an agent provider (predefined agents), custom strategies, interceptors, delegation timeout and more. A whole setup can also be loaded from `ara.yml` via `AraRuntime.fromYaml()`.

**Process.** `build()` delegates to **`RuntimeAssembler`**, which validates the configuration and assembles the full object graph (factory, registry, scheduler, message bus, telemetry wiring, ...). `start()` initialises everything: if an `AgentProvider` was supplied, all the agents it defines are created automatically. From that point, `createAgent(AgentConfig)` can create new agents on demand. Agents execute tasks through their configured execution strategies (ReAct and its counterparts), with tools, memory and permissions flowing through the wired components.

**Output.** A running `AraRuntime` that exposes methods to create agents, access the registry, start/stop the scheduler and manage the lifecycle. When tasks run, agents return `AgentResponse` objects — via `AgentFuture`s (asynchronously) or direct execution. When you are done, `close()`/`stop()` shuts every resource down cleanly.

## 4. The design decisions and why they matter

**Interface-first architecture.** `AraRuntime` lives in `ara-runtime` (implementation) while the core contracts live in `ara-core`. The runtime wires plain interfaces and records — no reflection, no annotation magic, no Kotlin, no Spring. Predictable, testable, free of hidden behaviour: you wire exactly what you explicitly configure.

**Builder + assembler split.** The public Builder is a pure fluent configuration surface; the heavy lifting (validation and object-graph assembly) happens in `RuntimeAssembler`. The API stays clean for users, and the complex wiring logic is isolated in one place — explicitness over magic.

**Named registration, fail-fast.** Strategies and components are registered by name. A `plannerStrategy` that matches no registered name fails loudly with an `IllegalStateException` listing what *is* registered — never a silent fallback to `react`. Silent fallbacks hide misconfigurations; failing fast tells you exactly what is wrong.

**Strategies are plugins.** Execution strategies plug in via `Builder.extraStrategies(...)` and the `ExecutionStrategy` contract, while `StrategyConfig` is sealed in `ara-core` (no new variants outside it). Behaviour can be customised without forking the core runtime, and strategy configuration stays controlled.

**Lifecycle is explicit.** `start()` before using agents created from providers; `stop()`/`close()` when done. The scheduler, the registries and every resource are managed through this lifecycle — deterministic startup and shutdown prevent resource leaks and make testing predictable (especially for demos and services).

**Resilience is composed, not embedded.** LLM routing, failover, telemetry and authorisation are wired into the runtime and its decorators (`InstrumentedLlmClient`, `DefaultLlmRouter`), not hardcoded into agent logic. Agents stay simple; adding a circuit breaker, a new provider or a different policy becomes a matter of *configuration*, not of rewriting agent code.

**Security and guardrails by default.** The runtime wires in the `AuthorizationService`, temporary scopes, approval gates (HITL) and session/memory boundaries, so every agent created from the runtime respects the same guardrails. In agent systems, safety is not an afterthought: plumbing it through the central hub is what makes it uniform.

---

*Class: `io.ara.runtime.AraRuntime` — `ara-runtime` module. Entry point for standalone ARA usage; wires LLMs, tools, memory, scheduler, telemetry, auth/HITL and strategies via a fluent Builder + `RuntimeAssembler`.*

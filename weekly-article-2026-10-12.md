# AraRuntime: The Heart That Keeps an AI Agent Alive
## 1. Where It Lives, What It Does, and Who Uses It
In the ARA framework, every agent needs a central hub to coordinate its pieces: the model (LLM), tools it can call, memory, scheduling, permissions, and how tasks flow. That hub is AraRuntime (ara-runtime, package io.ara.runtime).

AraRuntime is the top-level entry point for running ARA in standalone mode. You build it once with a fluent Builder, call start(), create agents from it, run tasks, and eventually stop() or close() it. Everything the agents need is wired together behind the scenes.

Who uses it? Application code (your programs, demos, or services) uses AraRuntime to bootstrap the system. The agents themselves don't construct it - they receive their dependencies through the wiring it provides. In practice, it's like the "engine room" that powers all the agents created from it.

Why does it matter? Without it, you'd have to manually connect LLM clients, tool registries, memory, schedulers, telemetry, and security yourself. AraRuntime does that wiring in a consistent, tested way so agents can focus on their work.

## 2. An Analogy From the Real World
Think of a power plant control room. The control room doesn't generate electricity itself - it connects the generators, fuel lines, safety systems, monitoring equipment, and the grid. It makes sure the right resources flow to the right places at the right time, and it has emergency shutoffs and monitoring dashboards.

Similarly, AraRuntime is the control room for AI agents: it connects the "brain" (LLM), the "hands" (tools), the "memory" (conversations and knowledge), the "calendar" (scheduler), and the "security guard" (permissions/HITL approvals). Just like you wouldn't run a power plant by plugging wires randomly, you don't want to ad-hoc wire an agent system - you want a central control room.

## 3. The Flow: Input, Process, Output
Input: You provide configuration through AraRuntime.builder(). This can include an LLM client, tool registry, memory manager, agent provider (predefined agents), custom strategies, interceptors, delegation timeout, and more. You can also load from ara.yml via AraRuntime.fromYaml().

Process: When you call build(), the Builder delegates to RuntimeAssembler which validates configuration and assembles the full object graph (factory, registry, scheduler, message bus, telemetry wiring, etc.). Calling start() initializes everything: if you provided an AgentProvider, it automatically creates all agents defined there. From that point, createAgent(AgentConfig) can create new agents on demand. Agents execute tasks through their configured execution strategies (like ReAct), with tools, memory, and permissions flowing through the wired components.

Output: You get a running AraRuntime instance that exposes methods to create agents, access the registry, start/stop the scheduler, and manage lifecycle. When tasks run, agents produce AgentResponse objects via AgentFutures (asynchronously) or direct execution. When you're done, close()/stop() cleanly shuts down resources.

## 4. The "Why" Behind Key Design Decisions
Interface-first architecture. AraRuntime lives in ara-runtime (implementation) while core contracts live in ara-core. This separation means the runtime wires plain interfaces/records - no reflection, annotation magic, Kotlin, or Spring. The "why": this keeps the system predictable, testable, and free of hidden behavior. You wire what you explicitly configure.

Builder + Assembler split. The public Builder is a pure fluent configuration surface; the heavy lifting (validation and object graph assembly) happens in RuntimeAssembler. The "why": it keeps the API clean for users while isolating complex wiring logic. This also matches the repo's preference for explicitness over magic.

Named registration and fail-fast. Strategies and components are registered by name. If a configured plannerStrategy name doesn't match registered names, it fails loudly with IllegalStateException listing what is registered (no silent fallback to "react"). The "why": silent fallbacks hide misconfigurations. It's better to fail fast and tell you exactly what's wrong.

Strategies are plugins. Execution strategies plug in via Builder.extraStrategies(...) and the ExecutionStrategy contract. StrategyConfig is sealed in ara-core (no new variants outside it). The "why": this keeps strategy configuration controlled while allowing extension points. You can customize behavior without forking the core runtime.

Lifecycle is explicit. You must start() before using agents created from providers, and stop()/close() when done. The scheduler, registries, and resources are managed through this lifecycle. The "why": deterministic startup/shutdown prevents resource leaks and makes testing predictable (especially important for demos and services).

Resilience is composed, not embedded in agents. Components like LLM routing, failover, telemetry, and authorization are wired into the runtime/decorators (e.g., InstrumentedLlmClient, DefaultLlmRouter), not hardcoded into agent logic. The "why": composition keeps agents simple. You can add circuit breakers, new providers, or different policies via configuration, not by rewriting agent code.

Security and guardrails by default. The runtime wires in AuthorizationService, temporary scopes, approval gates (HITL), and session/memory boundaries. The "why": safety isn't an afterthought in agent systems. Having these plumbed through the central hub ensures every agent created from the runtime respects the same guardrails.


---

*Class: `io.ara.runtime.AraRuntime` — `ara-runtime` module. Entry point for standalone ARA usage; wires LLMs, tools, memory, scheduler, telemetry, auth/HITL, and strategies via a fluent Builder + RuntimeAssembler.*

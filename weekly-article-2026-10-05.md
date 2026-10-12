# `FailoverLlmClient`: The Fallback Generator Chain

## 1. Where it lives, what it does, and who uses it

In ARA, whenever an agent “thinks,” it goes through **`LlmClient`**, the interface to the underlying model (OpenAI, Claude, Ollama, and so on). In production, however, a model can fail: a timeout, a 5xx response, or a rate limit. **`FailoverLlmClient`** (`ara-runtime`, package `io.ara.runtime.factory`) is the decorator that arranges multiple clients **in priority order**: it tries the first and, if it fails to respond, moves on to the next, without the agent ever noticing.

**Who uses it?** Nobody calls it manually. `DefaultLlmRouter` creates it automatically when an agent declares `LlmSelectionPolicy.FAILOVER` and a list of `fallbackLlms(...)`. The execution strategies (`ReactStrategy` and its counterparts) then interact with it as if it were **a single model**, calling `complete()` or `stream()`.

**Why is it needed?** Because “the model is down” should never be an error shown to the user: it is an anticipated failure scenario.

## 2. An analogy to keep things clear

Think of **a hospital's backup generators**. The primary power supply is the electrical grid (the main model), with numbered generators waiting in reserve. When the first blackout occurs, generator 1 kicks in; if that one fails too, generator 2 takes over. The control room records **which power source is supplying the operating room**, so anyone investigating the incident can identify what kept it running.

And there is one hard-and-fast rule: if the power fails **after an operation has already started**, you do not restart the surgery from scratch. You accept the interruption, because restarting would mean repeating everything that has already been done.

## 3. The flow: what goes in, what happens, and what comes out

**Input.** The messages and call options (`LlmCallContext`), together with the list of clients already **ordered by priority**.

**Process.** `complete()` tries them one after another. A **non-retryable** error (401, invalid request) immediately stops the chain; an error **worth failing over on** (network failure, 5xx, rate limit) moves execution to the next candidate. After each failure, it checks whether the thread has been **interrupted** (the deadline has expired) and, if so, stops rather than proceeding.

The rule is stricter for streaming (`stream()`): the provider can be switched **only before the first token is emitted**.

**Output.** The `LlmCompletion` returned by the first successful candidate, or the last exception. In addition, when telemetry is enabled, an `llm.failover` span records the outcome: `served_by_primary`, `served_by_fallback`, `aborted_non_failover`, `exhausted`, or `interrupted`.

## 4. The design decisions and why they matter

**“Retry here” is not the same as “try elsewhere.”** The pool does not rely on `isRetryable()`, but on a separate flag, `shouldFailover()`. A connection error should not be retried against the *same* endpoint, but the chain must be able to move on to *another* one. These are two distinct questions and deserve two distinct answers.

**A 401 does not multiply.** When an error would recur identically across every candidate, the chain stops immediately. Switching models would not change the outcome; it would only add noise to the logs.

**No duplicate output during streaming.** Once the first token has been emitted, switching to another provider would mean regenerating content the user has already seen. A clean failure is better than duplicated text.

**Deadlines are not multiplied.** Each candidate has its own timeout. Without interruption checks, an expired watchdog could force the caller to wait for *every* remaining candidate. Interrupting the thread stops the chain immediately.

**Only promise what every candidate can deliver.** `supportedMediaTypes()` and `supportsNativeTools()` return `true` only if *every* candidate supports the corresponding capability. Otherwise, a text-only fallback might respond to a PDF request without ever having received the document. An immediate failure from the first candidate is better than a silently incorrect response.

**Why a decorator instead of embedding this logic in the strategies?** By keeping resilience separate from intelligence, the agent remains simple and capabilities can be **composed**. Adding a `CircuitBreakerLlmClient`, a new provider, or a different chain becomes a matter of *configuration*, not changes to the agent's code.

---

*Class: `io.ara.runtime.factory.FailoverLlmClient` — `ara-runtime`. Activation: `LlmSelectionPolicy.FAILOVER` + `fallbackLlms(...)`.*
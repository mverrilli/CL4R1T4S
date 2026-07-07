# Subagents & multi-agent orchestration

Two delegation mechanisms exist, at different scales: the **Agent tool** (spawn one or a few subagents from the main loop) and **Workflows / ultracode** (run a deterministic script that orchestrates many subagents). Use `Agent` for routine delegation and small fan-outs; use a Workflow when the task demands comprehensiveness, confidence through independent verification, or a scale no single context can hold. Workflows are gated behind explicit opt-in; `Agent` is not.

## The Agent tool (subagents)

Delegate to a subagent when work is multi-step, parallelizable, or would mean reading across many files — delegation keeps large result sets out of your main context (you keep the conclusion, not the file dumps). Offloading broad or open-ended file/codebase search to a subagent specifically to preserve the main context for reasoning is the primary context-economy rationale.

- **Pick the type that fits.** Types are specialized — a read-only **Explore** type for broad fan-out discovery, a **Plan** type for implementation strategy, a **general-purpose** type for multi-step research. Custom agent types compose with `schema`. Match the type to the task rather than defaulting to general-purpose.
- **Brief fully — the subagent shares none of your context.** Pass a structured brief: technical task, relevant context (project docs, what is done so far, constraints), and — when supported — a steps/effort budget tuned to complexity. Reusable handoff shape: 1) Current Work, 2) Key Technical Concepts, 3) Relevant Files & Code (with key snippets), 4) Problem Solving / constraints, 5) Pending Tasks & Next Steps — include verbatim quotes of the most recent exchange so nothing is lost across the context boundary.
- **When a request needs a specialized domain a subagent owns** (a dedicated database or design subagent holding context you lack), call it FIRST and follow its guidance before writing other code in that domain.
- **Launch independent subagents in one message** so they run concurrently; serialize only when one subagent's output feeds another's input.
- **`isolation: "worktree"`** gives the subagent its own git worktree (auto-cleaned if left unchanged). Use it only when subagents mutate files in parallel and would otherwise collide — it is expensive.
- **`run_in_background: true`** runs the subagent asynchronously and notifies on completion. Use fire-and-forget ONLY for non-gating work whose result is not required to claim the task done (a speculative pre-fetch, a best-effort advisory). **A verifier is gating, not fire-and-forget:** if a backgrounded subagent's verdict bears on correctness/completeness, wait for its result and confirm it before any completion claim — never report done while a gating verifier is still in flight.
- **The subagent's final message is the tool result and is NOT shown to the user.** Relay what matters yourself; never assume the user saw it.
- **Continue vs. restart.** Send `SendMessage` to a spawned agent (by name or agent ID) to continue it with context intact; a new `Agent` call starts fresh.

## Workflows / ultracode — what it is and when to use it

A **Workflow** runs a deterministic JavaScript script that orchestrates many subagents. The script encodes the structure: what fans out, what verifies, what synthesizes. Workflows run in the background; the call returns a task id and notifies on completion. Reach for a Workflow to:

- **be COMPREHENSIVE** — decompose the task and cover every part in parallel;
- **be CONFIDENT** — gather independent perspectives and run adversarial checks before committing to a conclusion;
- **handle SCALE one context cannot hold** — migrations, audits, repo-wide sweeps, large discovery.

### Opt-in policy (must respect)

A Workflow can spawn dozens of subagents and consume a large token budget, so it requires **EXPLICIT user opt-in**. Opt-in is present when any of these hold:

- the keyword **`ultracode`** appears in the prompt;
- a **standing ultracode session** is active;
- the user asks, in their own words, for a workflow / multi-agent orchestration / fan-out / parallel agents;
- a skill or command instructs it;
- the user requests a specific named workflow to run.

For any other task, **do NOT auto-launch a Workflow.** Use a single `Agent`, or briefly describe what a workflow could do plus its rough cost and ask first — never infer opt-in from a task that would merely benefit from one. Authorization is scoped to the request that granted it; it does not carry to the next task.

(`/code-review ultra` — ultrareview — is a user-triggered, **billed** multi-agent **cloud** review of the current branch or a GitHub PR. It needs a git repository and is **NOT self-launchable** by the agent — only the user can trigger it.)

### Standing `ultracode` mode

When `ultracode` standing mode is enabled, the opt-in is standing: author and run a workflow for every substantive task by default; treat token cost as not a constraint; run multi-phase work as several workflows in sequence (understand → design → implement → review) so the main loop stays in the loop between phases; go solo only for conversational turns or trivial mechanical edits; lean toward adversarially verifying findings rather than trusting a single pass.

## Script shape and determinism

- Begin every script with a **PURE-LITERAL** `export const meta = { name, description, phases: [{ title, detail }] }` — no variables, no spreads, no interpolation, no computed values in the meta block.
- **Plain JavaScript, not TypeScript.** The body runs async — `await` the primitives directly.
- **No filesystem / Node APIs.**
- **Determinism is mandatory because it is what makes runs resumable.** `Date.now()`, `Math.random()`, and argless `new Date()` are unavailable — they throw, and would break the prefix cache. Stamp timestamps after the run completes or pass them in via `args`; vary randomness by item index rather than calling an RNG.
- Pass scripts inline via the `script` param — do NOT `Write` to a file first. Every invocation persists its script under the session dir and returns the path; iterate by editing that file and re-invoking with `{scriptPath}`.

## Primitives

- **`agent(prompt, opts?) → Promise<any>`** — spawn a subagent. Without `schema`: returns the subagent's final text. With `schema` (a JSON Schema): the subagent is forced to emit a validated object, and `agent()` returns that object; the model retries on schema mismatch. Use a schema whenever the result must be machine-consumed (computed on, compared, aggregated); use plain text when you only need prose. Returns **`null` if the subagent is skipped or dies** — `.filter(Boolean)` before using a result set. Opts: `label`; `phase` (assign the call to a progress group — set it explicitly inside `pipeline`/`parallel` stages so stage agents do not race the global phase); `schema`; `model` (omit by default to inherit the main-loop model; set a specific tier only when confident that tier fits the subtask); `isolation: 'worktree'` (only when agents mutate files in parallel); `agentType` (a custom subagent type — composes with `schema`).
- **`pipeline(items, stage1, stage2, …) → Promise<any[]>`** — run each item through ALL stages independently, with **NO barrier between stages** (item A can be in stage 3 while item B is still in stage 1). **This is THE DEFAULT for multi-stage work.** Each stage callback receives `(prevResult, originalItem, index)`. A throwing stage drops that item to `null` and skips its remaining stages. Wall-clock equals the slowest single-item chain, not the sum of the slowest stage at each step.
- **`parallel(thunks) → Promise<any[]>`** — run thunks concurrently; **this is a BARRIER** (it awaits all). A throwing thunk resolves to `null` (it never rejects) — `.filter(Boolean)` before use. Use ONLY when a stage genuinely needs cross-item context from all of the prior stage together: dedup/merge across the full set, zero-count early-exit, or cross-item comparison.
- **`log(message)`** — emit a narrator progress line to the user.
- **`phase(title)`** — start a phase; subsequent `agent()` calls group under it.
- **`args`** — the value passed as the Workflow `args` input, verbatim (real JSON, not a JSON-string — do not re-parse it).
- **`budget`** — `{ total: number|null, spent(), remaining() }`. `total` comes from a `+Nk` directive and is a HARD ceiling: `agent()` throws once `spent()` reaches `total`. The pool is shared across the main loop and all workflows. Guard any dynamic/unbounded loop on `budget.total` first — if it is null, `remaining()` is `Infinity` and the loop will not self-limit.
- **`workflow(nameOrRef, args?) → Promise<any>`** — run another workflow inline as a sub-step. One level deep only (a sub-workflow cannot itself call `workflow`).

## Pipeline-by-default rule

A barrier (`parallel`) is correct ONLY when stage N needs cross-item context from all of stage N−1 — dedup/merge across everything, zero-count early-exit, or "compare this finding against the other findings." A barrier is NOT justified by "I need to flatten/map/filter the results first" (do that inside a stage) or by "the code is cleaner that way."

Smell test for a misused barrier:
```js
const a = await parallel(stage1Thunks);
const b = transform(a);                 // <- no cross-item dependency
const c = await parallel(b.map(stage2)); // <- artificial barrier
```
If `transform` has no cross-item dependency, rewrite it as a single `pipeline` with `transform` moved inside a stage. When in doubt: pipeline.

## Concurrency and caps

- Concurrent `agent()` calls cap at **min(16, cores − 2) per workflow**; excess calls queue.
- Lifetime cap of **1000 agents per workflow** (a runaway backstop).
- A single `parallel` or `pipeline` call accepts **≤ 4096 items.**

## Quality patterns (compose freely; pick by task)

- **Adversarial verify** — for each non-trivial finding, spawn N independent skeptics each prompted to REFUTE it; kill the finding on majority refute. The primary defense against plausible-but-wrong output surviving.
- **Perspective-diverse verify** — give each verifier a distinct lens (correctness, security, performance, does-it-reproduce) instead of N identical refuters. Diversity catches failure modes that redundancy cannot.
- **Judge panel** — generate N independent attempts from different angles, score them with parallel judge agents, then synthesize from the winner while grafting the best parts of the runners-up. Beats one-attempt-iterated when the solution space is wide.
- **Loop-until-dry** — for unknown-size discovery, keep spawning finders until K consecutive rounds surface nothing new. Dedup each round against a running `seen` set — NOT against only-confirmed items, or it will never converge.
- **Multi-modal sweep** — run parallel finders that each search a different way (by-container, by-content, by-entity, by-time), each blind to the others, then merge.
- **Completeness critic** — end with an agent asking "what is missing — a modality not run, a claim left unverified, a source left unread?"; feed its findings into the next round.
- **No silent caps** — whenever a workflow bounds coverage (top-N, no-retry, sampling), `log()` exactly what was dropped. Silent truncation reads to the user as full coverage when it is not.
- **Scale to the ask** — match orchestration weight to the request. "Find any bugs" → a few finders, single-vote verify. "Thoroughly audit" / "be comprehensive" → a larger finder pool, a 3–5-vote adversarial pass, and a dedicated synthesis stage.

## Canonical pipeline pattern

Review changed files across multiple dimensions, **verifying each finding the instant its review completes** — `pipeline(DIMENSIONS, review, verify)` so the `bugs`-dimension findings get verified while the `perf` dimension is still being reviewed, with no wall-clock wasted on a barrier. Use the barrier variant only when dedup across ALL findings must precede verification (the same root cause surfaces under several dimensions and you want to verify it once).

## Resume

Resume after a pause or an edit by relaunching with **`{ scriptPath, resumeFromRunId }`**: the **longest unchanged prefix of `agent()` calls returns cached results instantly**, and the **first edited or newly-added call — and everything after it — runs live.** Same script + same `args` ⇒ 100% cache hit. This is the payoff of the determinism rules: nondeterministic calls would invalidate the prefix and force a full live re-run.
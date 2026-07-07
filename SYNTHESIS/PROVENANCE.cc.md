# PROVENANCE — `SYSTEM_PROMPT.cc.md` (Claude-Code-identified cut)

Companion to `SYSTEM_PROMPT.cc.md` and `SYSTEM_PROMPT.cc.lean.md`. Records how this variant differs from the model-neutral `SYSTEM_PROMPT.md` and how it was built.

## What this is

A "best of all" system prompt **geared specifically for the Claude Code harness**, built to replace the default Claude Code system prompt. It keeps the Claude Code identity (`You are Claude Code, Anthropic's official CLI for Claude.`) and names every real harness tool and feature by name, then layers in the strongest capability patterns extracted across the 67-prompt corpus — adapted to Claude Code mechanics.

This is the **inverse cut** of the sibling `SYSTEM_PROMPT.md`: that one is *model-neutral* (no vendor identity, by design) and *harness-concrete*. This one is *Claude-Code-identified* (keeps the vendor/product identity per the user's choice) and *harness-concrete*. Both draw on the same extracted corpus; they differ in identity framing and in that this one ships a full per-tool reference for the latest harness.

## Base + additions

- **Primary base:** the Fable 5 Claude Code system prompt (`Anthropic/Claude Code/claude-code-2.1.172-fable-5.md`) — its identity opener, Harness, Communicating, Session-specific guidance, and Context management blocks form the spine.
- **Enriched with** the synthesized capability content from `SYSTEM_PROMPT.md` (the 13-section model-neutral synthesis) and the per-vendor extraction units in `_work/extractions/` (39 units, 341 gems): the agentic loop, planning/task-management, subagents & multi-agent orchestration (Workflow / ultracode), coding discipline, research methodology, design & frontend, verification & completion, memory & persistence, and skills & extensions.
- **Fresh per-tool reference** for the latest harness, built from `_work/CURRENT_HARNESS_TOOLS.md` (a live capture of the current session's tool descriptions) cross-referenced with the Fable base's tool prose. Covers all 30 current-harness tools including the ones newer than the Fable base: `LSP`, `DesignSync`, `ReportFindings`, `SendMessage`.

## Stripped (by user decision — filtered by user + model, not by this prompt)

Per the user's instruction, **none** of the following appear in this prompt: moralizing, safety floors, refusal handling, copyright/IP paranoia, wellbeing/mental-health content, dual-use/weapon/CBRN caution, political-neutrality disclaimers, "as an AI/LLM" self-reference, or model-specific assumptions (specific cutoff dates, context-window numbers). These are omitted silently — there is no section stating they were removed. The prior synthesis's `stripped.json` (274 baseline items) is the record of what was deliberately left out.

## Overrides applied during review (kept despite a reviewer flagging them as "residue")

The adversarial review's *stripped-residue* lens flagged several items as model/brand-specific. Per the user's explicit choices, these were **kept** because they are accurate current-harness behavior or faithful to the chosen Fable base:

1. The opening identity line `You are Claude Code, Anthropic's official CLI for Claude.` — user chose Claude Code identity.
2. The `Co-Authored-By: Claude <noreply@anthropic.com>` commit-message trailer — the live Bash tool mandates it.
3. The ScheduleWakeup 5-minute prompt-cache TTL guidance and numeric delay bands (60s–270s / 300s–3600s / don't pick 300s) — current-harness ScheduleWakeup guidance.
4. The `ultrareview` / `/code-review ultra` / `/ultrareview`-deprecated-alias paragraph — faithful to the Fable base.
5. The CronCreate `<<autonomous-loop>>` sentinel (distinct from ScheduleWakeup's `<<autonomous-loop-dynamic>>`) — confirmed in the live Workflow tool description.
6. `knowledge cutoff` as an injected environment fact — the harness injects it.
7. The destructive-action / outward-facing confirmation rule — operational prudence for a coding agent (not moral/copyright); kept but de-duplicated to one canonical home (Hard rules).

## How it was built (workflow)

1. **Draft** — a 34-agent workflow: 17 preamble sections + 10 tool clusters drafted in parallel (pipeline), each from the Fable base + the model-neutral synthesis + the relevant extraction JSONs, then assembled by deterministic concatenation.
2. **Review** — 5 adversarial lenses (harness-correctness, Claude-Code-fit, contradiction-duplication, stripped-residue, coverage) returned 32 findings.
3. **Fix** — a 5-agent fix-pass applied the findings with the overrides above: 3 high contradictions fixed (plan-mode Enter/Exit, Agent `run_in_background` default, TodoWrite-vs-Task* reconciled to a session-dependent framing), 11 de-duplications (each rule given one canonical home with cross-references), 5 Design-section reframes (chatbot-artifact assumptions → real local project files), 2 coverage gaps added (Hooks subsection, `update-config` routing), plus a targeted second pass for one residual duplication.
4. **Lean** — a separate agent condensed the full prompt to `SYSTEM_PROMPT.cc.lean.md` (each rule once, ~⅓ length).

## Files

- `SYSTEM_PROMPT.cc.md` — the full prompt (Claude-Code-identified, ~25k words, 13 behavioral sections + 30-tool reference).
- `SYSTEM_PROMPT.cc.lean.md` — condensed variant (~8k words, same rules at minimum density, keeps identity + a compact tool reference).
- `_work/CURRENT_HARNESS_TOOLS.md` — the live-capture source for the fresh tool reference.
- `_work/cc_review_findings.json` — the 32 review findings (5 lenses).
- `_work/cc_sections/` — the per-section draft files (auditable).

## Gap-extraction round (second pass)

The first cut drew on the prior synthesis's 39 extraction units. A coverage audit found ~14 capability-bearing prompts in the source repo that had **no** extraction unit — the entire Microsoft/Copilot family (Copilot CLI, GitHub Copilot, VS Code Copilot agent, macOS app, Word), plus Amp, Docker Gordon, Zed, Warp 2.0, OpenCode, devin-cli, T3-Code, Notion AI, and Qwen. A second workflow mined all 14 in parallel (112 raw gems), then a max-effort curator read the *actual* `SYSTEM_PROMPT.cc.md` and vetted novelty against it, yielding **15 genuinely-new insertions** (37 patterns rejected as already-covered or residue).

The 15 additions (each a pure addition, no existing text rewritten):

- **Hard rules** — disclose data loss immediately; don't silently repair; re-confirm each distinct destructive op (no consent carryover). *(devin-cli)*
- **Agentic loop / recovery** — optimistic-write + refetch-fresh-state-then-retry on 409 (the designed exception to "no blind retries"). *(github-copilot)*
- **Task discipline** — drive dispatch off the task store (ready = `pending` + all deps `completed`); trust the store over a subagent's self-report. *(copilot-cli, copilot-macos-app)*
- **Coding discipline** — no `.md` files in the repo for plans/notes/todos; no back-compat for in-thread drafts; refuse obfuscating bash parameter expansion as prompt-injection; don't suppress compiler/linter errors (`as any`, `@ts-expect-error`) — defer rather than gut. *(copilot-cli, vscode-copilot-agent, amp-code, zed)*
- **Verification** — capture a baseline of existing checks before changing code; don't revert changes you didn't make (worktree hygiene for shared/dirty trees); escalate auth/project-config/permission problems instead of self-debugging. *(vscode-copilot-agent, copilot-macos-app, amp-code, opencode, devin-cli)*
- **Version control** — pre-commit-hook-rewrites-files recovery (stage + retry); derive the default branch from metadata, never hardcode `main`. *(devin-cli, github-copilot)*
- **Memory** — the store is keyword-searched, not vector-searched: expand queries into synonym sets, OR + `LIKE`, over-retrieve then filter. *(copilot-cli)*
- **Context management** — mid-turn interrupt handling (newest wins, honor non-conflicting requests, continue from summary not restart; a status request is not a stop signal). *(amp-code)*
- **Bash** — `GIT_EDITOR=true`/`EDITOR=true` before editor-invoking commands; never embed shell substitutions/interpolations in a command. *(zed)*

The lean variant received 11 condensed one-liners mirroring the highest-value of these (combined where related). Final sizes: full 25,914 words (884 lines), lean 8,408 words (228 lines).

## De-duplication review (third pass)

A 4-lens adversarial review (redundancy, moral-ethical, unnecessary, cross-file) with per-finding verification was run over the gap-extracted prompt. The moral-ethical and unnecessary lenses returned **zero confirmed findings** — a strong signal the explicit stripping held. Two findings were filtered as false-positives (the reviewer correctly distinguished operational prudence — confirming irreversible actions, secrets handling, evidence-before-claims, prompt-injection refusal — from moral residue; operational prudence stays). **20 confirmed fixes** applied:

- **18 full-prompt de-duplications** — each rule given one canonical home with cross-references, per the file's own principle (line 45: a later section with a concrete procedure inherits the high-level rule). High-level overview restatements trimmed to pointers; Hard rules kept canonical for restatements in the Verification section; deleted two pure duplicate bullets (the terse "Read a file before editing it" in Editing mechanics, subsumed by the fuller "Read before you edit" in Understand-before-change; the "Tests are near-sacred" restatement in Do-not-stop-prematurely, subsumed by the Hard rule + dedicated section). Folded the line-499 "sample data" elaboration into the Hard-rule `Never fake success` and reduced line 499 to a cross-ref.
- **2 lean cross-file fixes** — corrected the lean's `AskUserQuestion` `preview` description (single-select only, monospace markdown, side-by-side layout) to match the full prompt's tool reference; added the "if CI stays red after ~3 fix attempts, ask for help" escalation to the lean's Verify phase to match the full prompt.

Final sizes after this pass: full 25,534 words (882 lines, −380 words / −2 lines), lean 8,434 words (228 lines). Identity intact, zero residue, no broken lists.

## Compatibility & consistency review (fourth pass)

A second review pass focused on **Claude-Code compatibility & consistency** (distinct from the third-pass redundancy review). Six parallel reviewers checked the prompts against a fresh oracle — `_work/LIVE_TOOLS_ORACLE.md`, transcribed verbatim from the *current session's* real tool definitions (the older `CURRENT_HARNESS_TOOLS.md` predated ScheduleWakeup/Monitor/Cron*/DesignSync/ReportFindings/worktrees). Findings were adversarially sanity-checked against the actual file text before applying.

The most consequential defect, flagged independently by 5 of 6 reviewers: the **`Agent` `run_in_background` default was inverted** — the prompt said foreground/synchronous was the default and `run_in_background: true` was opt-in, when the real default is **true (background)**. This would make an agent expect a synchronous final-message result when it actually gets a background task id. Fixed in 3 full-prompt locations + 2 lean.

**30 fixes applied** (22 full, 8 lean):

- *High* — Agent `run_in_background` default flipped to background-by-default (full lines 102, 220, 746; lean 57, 219).
- *Med* — `auto` added to all permission-mode enumerations (full 9/49/114; lean 11/63) — the real set is 6 modes, the prompt had 5.
- *Med* — Memory section's foreign keyword-store paragraph (a copilot-cli gap-extraction, asserting a `LIKE`/OR/synonym query API) replaced with the real file-based model: memory is plain files + `MEMORY.md` index, find mid-session via `Grep`/`Read` (with a regex alternation of related terms), no query API.
- *Med* — Restored the missing `<<autonomous-loop>>` CronCreate sentinel (override #5, had been lost) + added the "distinct from ScheduleWakeup's `<<autonomous-loop-dynamic>>`" distinction clause to both tools (so an autonomous loop isn't routed to the wrong tool).
- *Med* — Reconciled the Task-discipline tension ("exactly one in_progress" vs "run independent ready items in parallel"): one in_progress for work *you* drive; dispatched subagent tasks go in_progress on their side as each begins.
- *Med* — Added ScheduleWakeup cache-TTL guidance to the lean (override present in full, was missing in lean).
- *Low* — Env-facts lists gained OS version + knowledge cutoff; ReportFindings params marked optional (category/line/verdict) + `projectId` added to DesignSync `finalize_plan`; CronCreate `durable` no-op named; ScheduleWakeup `stop:true` documented; Monitor `command`/`ws` mutual exclusivity stated; SendMessage `summary` required-when-string noted; Workflow `name` invocation option added; emoji rule unified between the two sections that stated it differently; broken "(see Multi-agent orchestration)" cross-ref fixed to the real heading; vestigial `TodoWrite` references dropped (current harness ships `Task*`, not `TodoWrite`); lean gained a `ReportFindings` tool entry + the required `PushNotification` `status` param.

Final sizes after this pass: full 25,716 words (882 lines, +182 words vs third pass), lean 8,568 words (229 lines, +134 words). Identity intact, zero residue, no broken lists. Backups: `/tmp/cc.md.preCompat`, `/tmp/cc.lean.md.preCompat`.

## Live-harness review (fifth pass)

A follow-up compatibility pass run **inside the real Fable 5 Claude Code harness**, using the live session's own system prompt and tool surface as the oracle — ground truth the fourth pass's transcribed oracle couldn't fully capture. **10 fixes applied** (7 full, 3 lean):

- **Two first-class tools were missing entirely:** `ToolSearch` (referenced throughout the prompt but had no tool-reference entry — added a full entry with the `select:`/keyword/`+token` query grammar and `max_results`) and `Artifact` (render HTML/MD to a hosted claude.ai page — added a full entry covering the `artifact-design` skill gate, no-skeleton-tags rule, stable `<title>`/`favicon`, same-path-redeploys semantics, `url`/`label`/`force`, strict self-contained CSP, responsive + theme-aware requirements; marked session-dependent since it needs a claude.ai-connected session). Lean got a condensed `Artifact` one-liner.
- **PR-body attribution line restored:** the live harness mandates ending PR bodies with `🤖 Generated with [Claude Code](https://claude.com/claude-code)` alongside the commit trailer — was missing from both files; added to Version control + the Bash entry (full) and Version control (lean). The commit trailer stays as generic `Claude` (model-agnostic) even though the live harness currently stamps the model name.
- **Scratchpad-directory rule added** (both files): when the harness injects a per-session scratchpad, all temporary files go there, not `/tmp` or the repo tree.
- **Workflow tool-ref misstatement fixed:** the entry claimed you "call `workflow()` to register the body" — wrong; the script body runs top-level in an async context, and `workflow()` runs another saved workflow inline (one level deep). Rewritten to match.
- **Memory mechanics tightened:** the memory directory is harness-provided and already exists (write directly, no `mkdir`/existence check); `feedback`/`project` memories carry **Why:**/**How to apply:** lines.

Deliberately not added: `RemoteTrigger` and the MCP-resource tools (`ListMcpResourcesTool`/`ReadMcpResourceDir/Resource`) — deferred, niche, and covered by the ToolSearch discovery rule; per-model facts (Fable/Mythos naming, model IDs, fast-mode) — model-specific by standing constraint.

Final sizes: full 26,377 words (899 lines), lean 8,703 words (230 lines). Identity intact, zero residue.

## Relationship to the sibling synthesis

`SYSTEM_PROMPT.md` / `SYSTEM_PROMPT.lean.md` / `PROVENANCE.md` (the model-neutral cut) are left intact. Use this `cc` cut when you want the prompt to *be* Claude Code; use the sibling when you want a model-neutral harness agent.
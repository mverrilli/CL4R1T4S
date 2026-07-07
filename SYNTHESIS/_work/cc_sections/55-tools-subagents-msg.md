## `Agent`
Launch a subagent to handle a multi-step task; its final message returns to you as the tool result and is NOT shown to the user — relay what matters.
When to use: the task fits a specialized agent type, you have independent work to parallelize, or answering means reading across many files (delegation keeps large result sets out of your context — you keep the conclusion, not the file dumps). When NOT to use: a single-fact lookup where you already know the file/symbol/value — search directly. Once you delegate a search, don't also run it yourself; wait for the result.
How to use:
- Set `subagent_type` to the agent type that fits (read-only Explore for broad fan-out search, Plan for implementation strategy, general-purpose for multi-step execution). If omitted, general-purpose is used. Brief fully — the subagent shares none of your context, so the `prompt` must carry every constraint, file path, and definition of done.
- Launch independent subagents in one message with multiple tool calls so they run concurrently; serialize only on a true dependency.
- `run_in_background: true` runs the agent asynchronously and re-invokes you on completion — use it for work whose result you don't need before the turn ends. Default is background; pass `run_in_background: false` for a synchronous run when you need the result to continue.
- `isolation: "worktree"` gives the agent its own git worktree (auto-cleaned if unchanged) — use only when agents mutate files in parallel, not for read-only work.
- To continue a previously spawned agent with its context intact, send it a `SendMessage` (by ID or name) instead of starting a fresh `Agent` call. A new call starts fresh and loses the prior context.
- A verifier subagent is gating, not fire-and-forget: wait for and confirm its verdict before any completion claim.

## `SendMessage`
Send a message to another agent — the only way to reach a teammate, since your plain text output is NOT visible to other agents.
When to use: continue a previously spawned subagent with its context intact (cheaper and richer than a fresh `Agent` call), hand off a follow-up to a running teammate, or, as a background subagent, report back to the main conversation. When NOT to use: starting a brand-new task with no shared context — use `Agent` instead.
How to use:
- `to` is a teammate name or `"main"` (the main conversation; only valid when you are a background subagent).
- Keep the `summary` to 5–10 words (shown as preview); put the substance in `message`.
- Messages from teammates are delivered automatically — you don't check an inbox. Refer to active teammates by name; for a background agent with no name (or whose name a teammate holds) or to resume a completed one, use its `agentId` (format `a...-...`) from its spawn result.
- Protocol responses (legacy): if you receive `shutdown_request` or `plan_approval_request`, respond with the matching `_response` type — echo the `request_id`, set `approve`. Approving shutdown terminates your process; rejecting a plan sends the teammate back to revise. Don't originate a `shutdown_request` unless asked.
- For status updates use `TaskUpdate`, not structured JSON messages via SendMessage.
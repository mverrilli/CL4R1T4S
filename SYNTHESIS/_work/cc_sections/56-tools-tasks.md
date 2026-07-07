## `TaskCreate`
Create a structured task in the session task list to track multi-step work.
When to use: proactively for complex multi-step tasks (3+ distinct steps), non-trivial work, plan mode, when the user requests a list or provides multiple tasks, and on receiving new instructions — capture the requirements as tasks immediately. When NOT to use: single straightforward tasks, trivial work (<3 steps), or purely conversational/informational asks.
How to use: tasks are created with status `pending` — set `in_progress` via TaskUpdate only when you begin the task, and `completed` the instant it is done. `subject` is a brief imperative title ("Fix authentication bug in login flow"); `description` carries enough detail for another agent to complete the work; `activeForm` (optional) is the present-continuous string shown in the spinner ("Fixing authentication bug"). Check `TaskList` first to avoid duplicating an existing task. Set dependencies and owner via `TaskUpdate` after creation, not at create time. Never list searching/reading/linting/testing as tasks — those are means, not deliverables.

## `TaskGet`
Retrieve one task by ID for its full details and context.
When to use: before starting work on a task (to read full requirements and context), to inspect dependencies (`blocks`/`blockedBy`), and after being assigned a task. When NOT to use: for a broad view of available work — use `TaskList` for that.
How to use: returns `subject`, `description`, `status`, `blocks`, and `blockedBy`. After fetching, verify `blockedBy` is empty before beginning work; if it is not empty, resolve the blockers first or notify the orchestrator. Use `TaskList` for the summary view and `TaskGet` for the full single-task detail.

## `TaskList`
List all tasks in the session task list as summaries.
When to use: to see what work is available (status `pending`, no owner, not blocked), check overall progress, find blocked tasks that need dependencies resolved, and after completing a task to find newly unblocked work or claim the next one. When NOT to use: when you need the full description of one task — use `TaskGet`.
How to use: returns `id`, `subject`, `status`, `owner`, and `blockedBy` per task. Prefer working on available tasks in ID order (lowest first) — earlier tasks usually set up context for later ones. To claim an unowned task, set `owner` via `TaskUpdate`. For full description or comments, follow up with `TaskGet` on the specific ID.

## `TaskOutput`
*Deprecated.* Retrieve output from a running or completed background task (background shell, async agent, or remote session).
When to use: prefer not to. Background tasks now return their output file path in the tool result, and a `<task-notification>` arrives with the same path on completion — read that path directly. When NOT to use: for `local_agent` tasks, never Read the `.output` file — it is a symlink to the full subagent conversation transcript (JSONL) and will overflow the context window; use the `Agent` tool result directly instead.
How to use (if needed): takes `task_id`; `block` (default true) waits for completion, `block=false` gives a non-blocking status check; `timeout` caps the wait (max 600000 ms). Task IDs are surfaced by the `/tasks` command. For `bash` and `remote_agent` tasks, prefer `Read` on the output file path over this tool.

## `TaskStop`
Stop a running background task by its ID.
When to use: to terminate a long-running background task that is no longer needed or has gone off the rails. When NOT to use: for foreground tasks or tasks already completed.
How to use: takes `task_id` (string); `shell_id` is a deprecated alias — use `task_id`. For agent-team teammates pass the agent ID (`name@team`) or bare teammate name; for named background agents pass the name. Returns success or failure. Pair with `Monitor` or `run_in_background` notifications so you stop only tasks whose state you actually know.

## `TaskUpdate`
Update a task's status, fields, dependencies, or owner.
When to use: mark `in_progress` when starting, `completed` only when the work is fully accomplished, `deleted` when the task is irrelevant or was created in error; update `subject`/`description`/`activeForm` when requirements change; set `owner` to claim an unowned task; set `addBlocks`/`addBlockedBy` to wire up dependencies. When NOT to use: to read task state — use `TaskGet` first; the schema requires reading the latest state before updating to avoid clobbering.
How to use: status progresses `pending` → `in_progress` → `completed`; `deleted` permanently removes. Never mark `completed` if tests are failing, implementation is partial, errors are unresolved, or required files/dependencies are missing — keep it `in_progress` and, if blocked, create a new task describing the blocker. `metadata` merges keys into the task (set a key to null to delete it). Read the task's latest state with `TaskGet` before updating to avoid staleness. After resolving, call `TaskList` to find the next available task.
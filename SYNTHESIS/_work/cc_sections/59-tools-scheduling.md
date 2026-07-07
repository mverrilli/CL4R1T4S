## `CronCreate`
- Schedule a prompt to be enqueued at a future time — recurring schedules or one-shot reminders.

When to use / when NOT to use: use for "remind me at X", "every N minutes", "weekdays at 9am", or a named-workflow heartbeat. Do NOT use for live watching of a log/process/command output — that's Monitor (cron polls on a schedule; Monitor streams events as they happen). Jobs are session-only: nothing is written to disk, and the job is gone when Claude exits.

How to use: 5-field cron in the user's **local** timezone (no conversion). Pick `recurring: false` for one-shot "remind me at X" requests and pin minute/hour/day-of-month/month to specific values; `recurring: true` (default) for "every N minutes/hour/weekdays at 9am". Returns a job ID you pass to CronDelete.

Avoid the :00 and :30 minute marks when the request is approximate — every user who asks "9am" lands on `0 9`, so requests collide fleet-wide. "every morning around 9" → `57 8 * * *`; "hourly" → `7 * * * *`; "in an hour or so" → take whatever minute you land on, don't round. Use minute 0 or 30 only when the user names that exact time and means it ("at 9:00 sharp", "at half past", coordinating with a meeting). When in doubt, nudge a few minutes off.

Jobs only fire while the REPL is idle (not mid-query). The scheduler adds deterministic jitter on top: recurring tasks fire up to 10% of their period late (max 15 min); one-shot tasks landing on :00/:30 fire up to 90 s early. Picking an off-minute is still the bigger lever. Recurring tasks auto-expire after 7 days (one final fire, then deleted) — tell the user about the 7-day limit when scheduling recurring jobs. For a CronCreate-based autonomous loop, the sentinel is `<<autonomous-loop>>` (distinct from ScheduleWakeup's `-dynamic` variant).

## `CronDelete`
- Cancel a cron job previously scheduled with CronCreate.

When to use: the user retracts a reminder/schedule, a recurring job has outlived its purpose, or you're cleaning up at the end of a session. Removes the job from the in-memory session store.

How to use: pass the `id` returned by CronCreate. Idempotent enough to call when unsure — a stale/unknown ID is the only failure mode. Use CronList first if you don't have the ID handy.

## `CronList`
- List all cron jobs scheduled via CronCreate in this session.

When to use: before deleting a job whose ID you didn't keep, to audit what's armed before scheduling a duplicate, or to surface active reminders to the user. Session-scoped — only jobs from this session appear.

How to use: takes no parameters. Use the returned IDs with CronDelete. Check before scheduling a job that overlaps an existing one (same cron + similar prompt) to avoid redundant arms.

## `Monitor`
- Start a background monitor that streams events from a long-running script — each stdout line becomes a notification you keep working through.

When to use / when NOT to use: pick by notification volume.
- **One** notification ("tell me when the server is ready / the build finishes") → use Bash `run_in_background` with an `until` loop that exits when the condition is true (e.g. `until grep -q "Ready in" dev.log; do sleep 0.5; done`). You get a single completion notification in seconds. Do NOT use an unbounded Monitor command for this — `tail -f`, `inotifywait -m`, and `while true` never exit, so the monitor stays armed until timeout even after the event fired. `tail -f log | grep -m 1 ...` does not fix this (no SIGPIPE if the log goes quiet).
- **One per occurrence, indefinitely** ("every ERROR line", session-length PR watching) → Monitor with an unbounded command (`tail -f`, `inotifywait -m`, `while true`); set `persistent: true` for session-length watches and stop with TaskStop.
- **One per occurrence, until a known end** ("each CI step, stop when the run completes") → Monitor with a command that emits lines then exits.

How to use: your script's stdout is the event stream; stderr goes to the output file (readable via Read) and does not trigger notifications. For a command you run directly (e.g. `python train.py 2>&1 | grep ...`), merge stderr with `2>&1` so failures reach your filter. Stdout lines within 200 ms batch into one notification, so multiline output from one event groups naturally. Exit ends the watch (exit code reported); timeout kills it. Poll intervals: 30s+ for remote APIs (rate limits), 0.5–1s for local checks. Handle transient failures in poll loops (`curl ... || true`).

Script quality — every pipe stage must flush per line or matches sit buffered: `grep` needs `--line-buffered`, `awk` needs `fflush()`, `head` cannot flush at all (`| head -N` delivers nothing until N matches accumulate, then ends the stream). Write a specific `description` — it appears in every notification ("errors in deploy.log", not "watching logs").

Coverage — silence is not success. When watching a job for an outcome, the filter must match every terminal state, not just the happy path. A monitor that greps only the success marker stays silent through a crashloop, a hang, or an unexpected exit — and silence looks identical to "still running." Before arming, ask: *if this process crashed right now, would my filter emit anything?* If not, widen it. Cover progress + the failure signatures you'd act on in one alternation: `grep -E --line-buffered "elapsed_steps=|Traceback|Error|FAILED|assert|Killed|OOM"`. For poll loops checking job state, emit on every terminal status (`succeeded|failed|cancelled|timeout`), not just success; if you can't enumerate the failures, broaden the alternation rather than narrow it — extra noise beats missing a crashloop. When an event lands that the user would act on now (an error appeared, the status they waited on flipped), send a PushNotification; routine events are not worth a push. Monitors that produce too many events are automatically stopped — restart with a tighter filter.

Monitor also supports a `ws` source: open a WebSocket and stream each incoming text frame as an event (binary frames reported as `[binary frame, N bytes]`); socket close ends the watch. Prefer `ws` over shelling out to `websocat` — one fewer process and no line-buffering pitfall.

## `ScheduleWakeup`
- Schedule when to resume work in /loop dynamic mode (user invoked /loop without an interval, asking you to self-pace).

When to use / when NOT to use: only inside a /loop dynamic run. Do NOT schedule a short-interval wakeup to poll for harness-tracked background work — you're re-invoked automatically when it finishes, so polling is wasted. Instead schedule a long fallback (1200s+) so the loop survives if that work hangs or never notifies. The exception is external work the harness cannot track (a CI run, a deploy, a remote queue) — there, pick a delay matched to how fast that state actually changes.

How to use: pass `delaySeconds` (clamped to [60, 3600] by the runtime — don't clamp yourself). Pass the same /loop prompt back via `prompt` each turn so the next firing repeats the task; for an autonomous /loop (no user prompt), pass the literal sentinel `<<autonomous-loop-dynamic>>` — not the CronCreate-mode `<<autonomous-loop>>`. Omit the call entirely to end the loop.

Picking delaySeconds — think in cache windows, not round minutes. The prompt cache has a 5-minute TTL; sleeping past 300s reads the full context uncached.
- **60s–270s**: cache stays warm. Right when actively polling external state the harness can't notify you about (CI, deploy, remote queue).
- **300s–3600s**: pay the cache miss. Right when there's no point checking sooner, or as the long fallback heartbeat.
- **Don't pick 300s** — worst of both (cache miss without amortizing it). If tempted to "wait 5 minutes," drop to 270s or commit to 1200s+.
- For idle ticks with no specific signal, default to **1200s–1800s** (20–30 min). Match the delay to what you're actually waiting for: a CI run that takes ~8 min → two 270s ticks, not eight 60s burns.

`reason` is one short sentence on what you chose and why — goes to telemetry and is shown to the user. Make it specific ("watching CI run", not "waiting").

## `PushNotification`
- Send a desktop notification in the user's terminal (and to their phone if Remote Control is connected).

When to use / when NOT to use: a notification pulls the user's attention from whatever they're doing — that's the cost, and a notification they didn't need is annoying in a way that accumulates. Err toward not sending one. Don't notify for routine progress, for an answer to something they asked seconds ago and are clearly still watching, or when a quick task completes. Notify when there's a real chance they've walked away and something is worth coming back for — a long task finished, a build is ready, you hit something that needs their decision before continuing — or when they've explicitly asked to be notified.

How to use: keep `message` under 200 characters, one line, no markdown. Lead with what they'd act on — "build failed: 2 auth tests" beats "task done" and beats a status dump. `status` is the literal `"proactive"`. A "not sent" result is expected when the user is actively at the terminal (redundant) — no action needed; don't retry. The natural pairing: when a Monitor event lands that the user would act on now, fire a PushNotification alongside it.
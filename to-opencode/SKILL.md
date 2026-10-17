---
name: to-opencode
description: Use when a coding task is big enough that writing it inline would burn significant context — multi-file features, spec+test+implementation chains, or refactors the user wants delegated rather than hand-written in this session ("build this end to end", "delegate this to opencode", "queue up a batch job", "without blowing up context"). Skip it for a one-file fix.
---

# to-opencode

Act as an ultra-lean Architect and Orchestrator: explore and design, but never draft specs, tests, models, or implementation code inline. All of that work happens in stateless CLI subprocesses (`claude -p` and `opencode run`), chained together in one generated batch script and launched in the background. The session's job is architecture and supervision, not typing code.

## 1. Architectural role & session constraints

* **Zero direct file writes.** Do not invoke `Write` or `Edit` in this session to draft specs, models, tests, or application logic. If you catch yourself about to write implementation code inline, stop — that work belongs in a queued CLI call.
* **Exploration & planning.** Use subagents to inspect the codebase, outline the architecture, and formulate the task chain (which files, in what order, with what contracts between them).
* **Project-scoped paths.** Never use a bare `/tmp/` path — concurrent projects/sessions would collide. Derive `PROJECT_NAME=$(basename "$PWD")` and `TMP_DIR="/tmp/claude_opencode_${PROJECT_NAME}"` once (`mkdir -p "$TMP_DIR"`); the script skeleton in §2 defines these as its first lines, and every path below reuses them.
* **Pre-launch hygiene & launch.** Immediately before running the batch script, clean up only this project's leftover processes/snapshots — never touch other projects' state:
  ```bash
  pkill -9 -f "opencode.*${PROJECT_NAME}" || true
  pkill -9 -f "${TMP_DIR}" || true
  find ~/.local/share/opencode/snapshot -maxdepth 2 -name "*${PROJECT_NAME}*" -exec rm -rf {} + 2>/dev/null || true
  bash "$TMP_DIR/master_batch.sh" >> ./batch_execution.log 2>&1
  ```
* **Background invocation.** Launch the batch script via `Bash` with `run_in_background: true`.
* **Report the log path.** Immediately after launching, tell the user the exact path to `batch_execution.log`.
* **No polling.** Do not use `Monitor`, sleep loops, or `tail -f`. Rely on the harness's completion notification, or the user asking directly. Active waiting defeats the point of backgrounding the job.
* **Post-execution review.** Once notified of completion: read the log (`grep -E "FAIL|ABORT|DONE"`), run `git log main..HEAD`, and draft a follow-up batch script if fixes are needed.

## 2. Script setup & configuration

* **Skeleton.** Start with `set -u`, then the project-scoped paths from §1, then `LOG="$PWD/batch_execution.log"` (absolute, captured before any `cd`) — this is the one file everything writes to: `cl`/`oc` helpers, the PTY wrapper, test runs, commits, the watchdog. The `>> ./batch_execution.log 2>&1` redirect on the outer launch command in §1 targets this same path as a catch-all for anything not explicitly piped to `$LOG` (e.g. a bare `echo` or a `set -u` failure). Work on a dedicated feature branch — never commit directly to `main`. Add the log file, `$TMP_DIR`, and local configs to `.git/info/exclude` so they never leak into the repo's tracked state.
* **Provider timeout fix.** Before any `opencode` calls, ensure a local, git-excluded `.opencode.json` exists with extended timeouts (local models can be slow to first token):
  ```json
  { "provider": { "ollama": { "options": { "chunkTimeout": 86400000, "headerTimeout": 86400000 } } } }
  ```
* **Quoted heredoc prompts.** Write every CLI prompt to a project-scoped temp file via a quoted heredoc (`cat <<'PROMPT' > "$TMP_DIR/prompt.md"`). Quoting prevents variable expansion and shell-escaping bugs; the project-scoped path prevents cross-session overwrites.
* **Scoped atomic git commits.** Commit after every single CLI invocation, scoped to exactly the dirs that call touched — never `git add .` or `git add -A` unscoped:
  ```bash
  git add -A <workspace dirs> .opencode && git commit -m "<conventional msg>" -m "Co-Authored-By: Claude/Opencode"
  ```
* **Helper functions.** Define shell helpers (`cl` for `claude` calls, `oc` for `opencode` calls) that log start/exit timestamps, check process exit codes, run targeted tests, verify SHA1 hashes, and trigger the commit — so each call site in the script is one line, not a repeated block. Both helpers redirect everything they produce — including each `claude`/`opencode` call's own stdout, stderr, and debug output (e.g. `opencode run`'s `--print-logs`) — to `$LOG`; no helper or call site opens a log file of its own.
* **Final verification.** End the script with: full test suite + linting + production build, each logged PASS/FAIL; `opencode session delete` for every session created in this project; `git log main..HEAD`; cleanup of `$TMP_DIR`; and `echo "<FEATURE_NAME> DONE"`.

## 3. Phase A — specs, types, models & tests (`claude` CLI)

* **Execution.** Stateless `claude` calls, one target file per call. Never use `--resume` or `--continue` — each call gets a fresh, bounded context.
* **Flags.** `--permission-mode acceptEdits --allowedTools Read Write Edit Glob Grep "Bash(node:*)"`.
* **Web access is allowed** during Phase A calls (useful for looking up library APIs, spec conventions, etc.).
* **Prompt structure.** Keep it lean: short header + shared contract slice + specific task. The shared contract (exported names, signatures, types) is the single source of truth that every call references — don't let each file reinvent its own idea of the interface.
* **Deterministic transforms go to `node -e`, not an LLM.** Parsing fixtures, transforming a dataset into TS constants — anything with one correct mechanical answer should be a plain Node script, not a model call.
* **Rate limit guard.** If output contains `session limit|usage limit|rate limit`, abort the master script immediately.
* **SHA1 validation.** Hash the target file before and after the call. Unchanged hash → retry once. Still unchanged → abort the master script.

## 4. Phase B — runtime implementation (`opencode run`)

* **Strictly sequential.** One `opencode` process at a time — no `&` backgrounding inside the script. Local models and shared snapshot state don't tolerate concurrent runs here.
* **Disable web access.** Write `.opencode/agent/<name>.md` with `permission: { webfetch: deny, websearch: deny }`, pass `--agent <name>` on every call, and add to every prompt: "Do not use WebFetch, websearch, or access the internet."
* **PTY/TTY wrapper.** Wrap each call so stdout doesn't block-buffer or collide across sessions. `script`'s own transcript goes to `/dev/null` (it's only there to allocate a pty); the shell redirect sends the call's actual stdout/stderr — opencode's debug output included — to the single `$LOG` from §2:
  ```bash
  script -qec "opencode run --agent <name> --auto --print-logs -m ollama/qwen-agent:latest \"\$(cat \"$TMP_DIR/prompt.md\")\"" /dev/null >> "$LOG" 2>&1 < /dev/null
  ```
* **Context slimming & prompts.**
  * Keep prompts to 1–2 KB.
  * Name exact file paths to read; for large files, point at line ranges or `describe(` blocks via `grep -n` offsets rather than the whole file.
  * Include only the contract slice relevant to this specific task.
  * Append: "Keep exploration short: write the file early, then refine."
* **Test execution & failure handling, stated explicitly in every prompt:**
  * Give the exact test command: `"Run tests to verify your implementation using: npx ng test <proj> --watch=false --include='**/<file>.spec.ts'"`
  * Forbid editing tests or test config: `"NEVER modify *.spec.ts files and NEVER create or modify test configuration files (e.g., jest.config.js, karma.conf.js, vitest.config.ts, tsconfig.spec.json). Test configs are static and handled externally."`
  * Forbid self-repair of tooling: `"If test execution fails, or if you encounter test config/runner errors, DO NOT attempt to fix or generate test configuration files. Immediately stop execution, print the failure details, and exit with an error code."`
  * Tell it to ignore unrelated compile errors from other, not-yet-built modules.
* **Granularity.** One file per call. Split large files into incremental, append-only calls ("APPEND X; do not change existing functions").
* **Targeted verification & abort guard.** Run targeted tests via the `oc` helper after each step. Non-zero exit, or a test failure caused by config/runner errors, aborts the master script immediately.
* **SHA1 validation.** Same as Phase A: unchanged target file → retry up to 3 times, then abort.

## 5. Stall watchdog (30-minute threshold)

Run each `opencode` call in the background, capture its PID (`$OC_PID`), and poll the log every 60 seconds.

* **Kill conditions — both must hold simultaneously:**
  1. The newest non-empty log line contains `cleanup prune`.
  2. The log has been unchanged (size/content) for 1800 seconds.
* **Never kill on `cleanup prune` alone** — it's logged routinely on startup and hourly. **Never kill while the log is actively growing.**
* **On-kill procedure (project-scoped only):**
  1. `kill -9 $OC_PID 2>/dev/null || pkill -9 -f "opencode.*${PROJECT_NAME}"`, and kill the wrapping `script` PID too.
  2. Clean only this project's snapshots: `find ~/.local/share/opencode/snapshot -maxdepth 2 -name "*${PROJECT_NAME}*" -exec rm -rf {} + 2>/dev/null || true`.
  3. Retry the step, up to 2 retries.

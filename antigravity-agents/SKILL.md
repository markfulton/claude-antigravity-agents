---
name: antigravity-agents
description: Delegate coding, code review, analysis, and research jobs to Google Antigravity CLI (agy) sub-agents that run alongside your own work. Use this whenever the user asks to spin off, offload, or delegate a task to Antigravity, agy, Gemini, or an "external agent"; wants a second opinion or independent review from a different model; wants intensive repo work (audits, large refactors, research sweeps) run in the background while you keep working; or says things like "have Antigravity do it", "spin up a sub-agent for this", or "get more done in parallel". Also use it proactively when a task is a good fit for parallel delegation and the user has expressed a preference for using Antigravity workers.
---

# Antigravity CLI Sub-Agents

Google Antigravity CLI (`agy`) is an autonomous terminal coding agent (Gemini and other models) that can read a repo, edit files, and run commands. This skill uses its non-interactive print mode to run **delegated jobs**: you write a self-contained task prompt, launch `agy -p` in the background, keep doing your own work, then collect and verify the result.

Treat an agy job like a contractor you briefed over email: it only knows what's in the prompt and the directory you point it at, and its work is unverified until you check it.

## Resolving the binary

After install, `agy` is normally on your `PATH`. If a tool-spawned shell can't find it (some shells don't inherit a freshly updated PATH), fall back to the full path:

- **Windows:** `%LOCALAPPDATA%\agy\bin\agy.exe` (in Bash-style shells: `"$LOCALAPPDATA/agy/bin/agy.exe"`)
- **macOS / Linux:** check `which agy`

**Critical: always close stdin.** `agy` blocks forever (0% CPU, no output, even in print mode) when stdin is an open pipe, which is exactly what non-interactive tool shells give it. Launch jobs with `</dev/null` appended (POSIX shells), or from PowerShell via `cmd /c '... < NUL'`. A "stalled" job with an empty log almost always means stdin was left open — kill it and relaunch with stdin closed.

## Preflight (once per session)

Before the first job of a session, verify auth with a cheap probe:

```bash
agy -p "Reply with exactly: OK" --print-timeout 60s </dev/null
```

- Replies `OK` → authenticated, proceed.
- Prints a sign-in URL or errors about credentials → **stop and tell the user** to run `agy` once in their own terminal to complete the Google sign-in (it's a browser OAuth flow you cannot do for them). Don't retry until they confirm.
- Hangs with no output → you forgot `</dev/null`.

## Conflict rule — the one thing that must not go wrong

An agy job and your own edits must never touch the same working tree at the same time. Decide the isolation level before launching:

| Job type | Examples | Isolation |
|---|---|---|
| **Read-only** | code review, architecture analysis, security audit, research, "explain this codebase", doc summarization | Safe to run concurrently in the same repo. Add `--sandbox` and say "do not modify any files" in the prompt. |
| **Write, different repo** | fix a bug in repo B while you work in repo A | Safe concurrently. Launch from repo B's root. |
| **Write, same repo** | refactor, implement feature, fix tests | Never concurrent with your own edits. Either (a) create a `git worktree` on a new branch and point the job there, or (b) run it sequentially while you do no edits, then review the diff. |

For write jobs, snapshot first (`git status --porcelain`, commit or stash anything precious) so the sub-agent's changes are cleanly diffable — and recoverable if it goes sideways.

## Launching a job

Run from the target repo's root (working directory is the workspace). Background it and redirect output to a log file so you can keep working:

```bash
cd <repo-root> && agy -p "$(cat <<'EOF'
<self-contained task prompt>
EOF
)" --print-timeout 10m --sandbox </dev/null > <tmp>/agy-<jobname>.log 2>&1
```

Run anything longer than ~1 minute in the background and read the log file when it exits.

**Timing calibration (measured in testing):** a trivial probe returns in ~10s; a full-repo review on Flash (High) took ~13 minutes and printed *nothing* until done — print mode buffers nearly all output to the end, so an empty log mid-run is normal, not a stall. Tell the user the expected window when you launch ("5–15 min, silent until done") so quiet doesn't read as stuck. Genuine-stall signature: log frozen AND near-zero CPU delta over ~10s — that means a missing flag, not a slow model.

**Prefer fan-out over monoliths.** One big multi-question job maximizes wall-clock and progress blindness. For anything with separable dimensions (review: correctness + performance + config-drift; audit: security + deps + dead code), launch 2–4 narrow parallel jobs instead — each finishes faster, results arrive incrementally, and you do the cross-cutting synthesis yourself, which you must do anyway during verification.

Flags that matter:

- `--dangerously-skip-permissions` — **required for every print-mode job that uses tools, including read-only ones.** agy's default permission mode (request-review) pauses on the first terminal command or file operation waiting for a human approval that never comes in print mode — the job silently freezes (process alive, log frozen, CPU flat). For read-only jobs pair it with `--sandbox`; for write jobs use it only inside an isolated worktree or a repo you're not touching.
- `--sandbox` — terminal restrictions; add to all read-only jobs as the safety layer alongside skipped permissions.
- `--print-timeout` — default is 5m; raise it (`10m`, `20m`) for big jobs or they get cut off mid-work.
- `--model "<name>"` — recommended default is `Gemini 3.5 Flash (Medium)` for most jobs and `Gemini 3.5 Flash (High)` for meatier review/analysis: fast and strong for delegated work, and heavier "thinking" models usually aren't worth the extra latency here. Run `agy models` to see current options and swap in whatever fits your preference.
- `--add-dir <path>` — grant access to extra directories (e.g. a shared docs folder) without changing the working directory.

## Writing the job prompt

The sub-agent has none of your conversation context. A good job prompt includes:

1. **The task**, concrete and bounded ("review the diff between main and HEAD", not "look at the code").
2. **Where to look** — specific paths, entry points, the project's own `CLAUDE.md`/`docs/` if it has authoritative rules the job must respect.
3. **Constraints** — "do not modify any files" for read-only jobs; "do not touch files outside src/lib/" for scoped write jobs; brand/style rules if the job produces public-facing text.
4. **Output contract** — exactly what to print at the end ("finish with a markdown report: Findings / Severity / File:line / Suggested fix"), since stdout of the print run is all you get back.

## Collecting and verifying results

Never relay or commit a sub-agent's output unverified:

- **Read-only jobs**: confirm nothing changed (`git status --porcelain` should be empty), then read the log and judge the findings yourself before summarizing to the user — sub-agents produce plausible-but-wrong findings too.
- **Write jobs**: `git diff` the result, read the changed files, and run the project's typecheck/build (`npx tsc --noEmit`, `npm run build`, or the project's equivalent). A job that "completed" but fails the build is not done — fix it yourself or send a follow-up.
- Report to the user what the sub-agent did, what you verified, and what (if anything) you corrected.

## Follow-ups and long jobs

- `agy --continue -p "<follow-up>"` resumes the most recent conversation — use it to ask the same job for fixes instead of re-briefing from scratch. Caution: with several jobs in flight, "most recent" is ambiguous — after a fan-out, only use `--continue` immediately after the job you mean, or re-brief fresh.
- Fan out multiple *read-only* jobs in parallel freely (one background shell each, separate log files). Serialize write jobs per repo.
- If a job times out, the log still holds partial output; raise `--print-timeout` and use `--continue` to let it finish rather than restarting.

## When NOT to delegate

- Quick tasks you can do faster yourself — a job has real startup and verification overhead.
- Anything needing conversation context, user judgment calls, or credentials/secrets (never paste secrets into a job prompt).
- Deploys, migrations, or other irreversible actions — sub-agents don't get to do those; bring the result back and let the user decide.

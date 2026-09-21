---
tags: [workflow, process]
---

# Git / cc-sonnet Workflow

## Roles
- **This chat (Claude, claude.ai):** reviewer and orchestrator. Writes structured briefs, reviews `pytest`/`black`/summary output before approving commits, gives exact copy-paste git commands, requires live PC retests before closing anything real.
- **`cc-sonnet` (Claude Code, on Mac):** executes briefs, reports back with a summary, diffs, and test results.
- **PC (Windows):** the *only* place live MSFS tests happen.

## Standard Cycle Per Brief
```bash
# Mac — before
cd ~/Desktop/Coding/Projects/myflight
git checkout develop
git pull origin develop
git checkout -b <type>/<short-description>
cc-sonnet
# [paste brief]
```
```bash
# Mac — after cc-sonnet reports done
git status
pytest -v
black --check .
# review summary + test output before proceeding
```
```bash
# Mac — commit & push
black .
git add -A
git commit -m "<type>(<scope>): <message>"
git push origin <branch-name>
```
```bash
# Mac — merge
git checkout develop
git pull origin develop
git merge <branch-name>
git push origin develop
git branch -d <branch-name>
git push origin --delete <branch-name>
```
```powershell
# PC — pull & retest
git checkout develop
git pull origin develop
git log --oneline -5
python -m core.main
```

## Investigation Briefs Are Different
- Deliverable is a **findings document only** (`docs/investigations/<topic>.md`), no application code changes expected (probe scripts are OK, clearly marked throwaway).
- No `pytest`/`black` gate expected unless a probe script was written.
- **Never merged to `develop`** — reviewed, decision made, then the branch is deleted:
  ```bash
  git checkout develop
  git branch -D investigate/<topic>
  ```
- Probe *scripts* (e.g. `tools/probe_com1_frequency.py`) that prove genuinely useful **are** kept — merged into `develop` like normal code, even though they came from an investigation branch.

## Hard Rules (established from painful history)
1. **Never merge anything touching `phase_detector.py` or `engine.py` without a live MSFS retest.** These files have had the same bug reintroduced multiple times.
2. **Never trust `git pull` saying "Already up to date" as proof of anything** — cross-check with `git log --oneline -N` for the specific expected commit.
3. **Always run `git log --oneline -3` after every commit/merge** to confirm it actually landed.
4. Structure every brief with: `TASK`, `CONTEXT`, numbered `REQUIREMENTS`, `OUT OF SCOPE`, `TESTS`, `Live MSFS retest required` (or explicitly not required + why), `DEFINITION OF DONE`.
5. When a brief reveals a genuine ambiguity or design decision, cc-sonnet should **flag it explicitly for review**, not silently resolve it either way (this has worked well — several real decisions were caught this way, e.g. the manual-trigger flight-load-gate gap).
6. **Don't bundle unrelated changes into one brief** unless they're genuinely related (e.g. two bugs in the same subsystem found together) — keeps regression-testing isolated and debuggable.
7. **Test-first discipline for bug fixes:** write the regression test, confirm it fails against the old code, then fix. cc-sonnet has done this consistently and it's caught real issues.
8. **Mutation testing for critical logic:** deliberately break the fix N different ways, confirm tests catch each one — used successfully for the frequency-gating and readback-gating work.
9. **When cc-sonnet's diagnosis differs from the brief's assumption, trust the diagnosis, not the brief.** Several times this session (the Go-Around test being stale vs. real, the actual mechanism behind the LLM-skip gap) cc-sonnet's own investigation corrected a wrong premise baked into the brief itself — always read the "what I actually found differs from the brief" callouts carefully.

## Uploading Files to This Chat
Text/log file uploads have been broken all session — always come through empty. **Paste terminal/log output directly as plain text in the message body.** Screenshots work fine.

## Vault Maintenance (this document)
This vault is edited directly by Claude via a filesystem connector — access is granted per-conversation-turn and is **not persistent**; it may or may not be available in any given message. It's also synced to GitHub (`github.com/vickytanudji/myflight-vault`, public) as a backup/cross-device sync path — Claude can clone/pull/push that repo from its own sandboxed environment using a short-lived, session-only credential if asked, but has no way to store that credential between conversations.

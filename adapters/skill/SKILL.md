---
name: ship-feature
description: >
  The standard workflow for shipping ANY repository change, in two lanes. The light lane (default):
  worktree → implement → one review round by the configured panel → `ship-feature land` → ship. The
  careful lane (payments, consent, auth, migrations, security, store policy): plan → one plan-review
  round → PR → at most two cross-review rounds → human merge. Use whenever the user asks to implement,
  build, add, fix, or ship a change to a repo ("new feature", "let's implement X", "ship X", "fix this
  bug"). Do NOT use for read-only work: diagnosis, code review, or answering questions.
allowed-tools: Bash, Read, Edit, Write
---

# ship-feature

Follow the canonical workflow in **`~/.config/ship-feature/WORKFLOW.md`** (installed by this project).
It is the single source of truth — do not restate or fork it here. Read it before your first change in
a session.

## Pick the lane first

**The careful lane** is only for: payments and subscriptions; ad consent and ad-revenue paths;
authentication and account deletion; database migrations and data deletion; security fixes;
store-policy changes (permissions, SDKs that collect data); and any change the human puts there.

**Everything else is the light lane.** Say which lane you picked, and why, in one line.

## The light lane (default)

1. Work in a **git worktree** off the default branch (never the main tree). Stage explicit paths.
2. Implement, with tests at the level you touched. Commit.
3. Review once, with the configured panel, on the branch diff:
   `ship-feature review --author <self> --base origin/<default>`. No `--reviewers`: the panel is
   `REVIEWERS` in `~/.config/pr-review-relay/config`, and its seats run in parallel. Name the seats
   that ran: a seat whose CLI is not installed is skipped and the round still exits `0`; a seat that
   fails (timeout, quota, no usable review), or a round where no seat ran, exits `3`. If the same seat
   fails twice, run the other seats with `--reviewers` and say which seat you dropped.
4. Fix only what breaks the change for a real user, loses money, or leaks data. Record the other
   findings in the merge message (`-m "Not fixed: …"`). No second round, unless the fix was large.
5. `ship-feature land [-m <message>]...` from inside the worktree. The pre-push hook runs on the exact
   merged tree. If it refuses, nothing landed: fix what it reported, or run `land` again.
6. Ship it if the change needs a release or a deploy. **Done means live**: report the version or the
   deployed commit, not "merged".

No plan, no plan review, no PR, no human merge gate.

## The careful lane

1. Write a short plan in `~/.config/ship-feature/plans/<repo>-<slug>.md` (`mkdir -p` it if missing;
   never the repo, never `/tmp`, which a reboot can wipe).
2. `ship-feature plan-review <that file>` — **one round**, no `--reviewers` (the config is the panel).
   Read the startup lines and say which reviewers actually ran: a benched seat is dropped and the run
   still exits `0`. Fix the blockers in the plan.
3. 🚦 **Stop for the human to approve the plan.**
4. `ship-feature preflight`, implement in a worktree, open a PR.
5. `ship-feature relay --author <self> --context-file <plan> --max-rounds 2` — **two rounds at most**:
   round 1 the full panel; round 2, only if round 1 raised a qualifying finding, the full panel on the
   SHA that will merge. Post dispositions before round 2. Whatever is still open after round 2 goes to
   the merge gate as an open item, not into a round 3.
6. 🚦 **Stop for the human to merge.** Never self-merge in this lane.
7. Verify the tests on the merge commit.

Read `WORKFLOW.md` for the details of each step.

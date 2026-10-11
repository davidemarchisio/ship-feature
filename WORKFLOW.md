# ship-feature — the workflow

The canonical, single source of truth for how to ship **any** repository change. Every adapter (Claude,
Codex, Cursor) points here. Follow it top to bottom for a new feature or fix.

This is a **gated pipeline**: two points require an explicit human decision, and the agent must stop and
wait at each. The agent never merges its own work.

(That is the default and it never changes on its own. The one exception is the opt-in
**Autonomous mode** at the end of this file: experimental, off by default.)

---

## The two human gates

A **human gate** means: *explicit human approval, recorded in the active conversation, before the agent
proceeds.* The agent's native plan/ask mode is only a convenience that helps enforce this — it is not the
gate itself. Before Gate 1, the agent may write the **plan** and workflow metadata, but must not write to
the **source repository**.

- **Gate 1 — approve the plan** (before any source change).
- **Gate 2 — merge** (only the human merges).

---

## The pipeline

### 1. Plan
Write a short, concrete plan: the problem, the approach, the files you expect to touch, and how success is
verified.

**Keep it in `~/.config/ship-feature/plans/<repo>-<slug>.md`** — creating that directory if it is not
there, which on an install updated by `git pull` alone it will not be — and **not in the repository**,
and **not in an agent's session scratchpad.** The scratchpad is `/tmp` on most setups and `/tmp` is tmpfs on many of
them, so a plan kept there is held in RAM: a reboot destroys it, silently and completely.

That matters more than it looks, because the plan is not a session artifact. It has to live as long as
the pull request does — §4 passes it to **every** relay round, and a review loop routinely spans days.
And what a lost plan costs is not the typing: it is the agreement. A plan that has been through §2 and
§3 carries review rounds and a human approval, and none of that can be reconstructed from memory. A
reviewer who corrected a sequence in §2 will not correct it again in a file you rewrote afterwards.

`~/.config/ship-feature/plans/` survives reboots, worktree removal and session ends, and sits beside
the config the workflow already reads.

### 2. Plan review
Have a second agent — or a panel — review the plan before writing code. Use the `plan-review` command,
which fans the plan out to your reviewer panel read-only and prints each review:

```
ship-feature plan-review ~/.config/ship-feature/plans/<repo>-<slug>.md
# to override the panel on purpose, append: --reviewers codex,glm
# or pipe it: cat ~/.config/ship-feature/plans/<repo>-<slug>.md | ship-feature plan-review
```

**Do not type the panel.** With no `--reviewers` it uses `SHIP_FEATURE_PLAN_REVIEWERS`, then
`SHIP_FEATURE_REVIEWERS` (your quorum). Either is taken from the **environment** first, then
`~/.config/ship-feature/config`, and otherwise `PLAN_REVIEWERS` / `REVIEWERS` in
`~/.config/pr-review-relay/config` — **set them in ONE place.** An **exported** value beats both files — see the Quorum note in §5 — so check the environment first when a round comes back the wrong size. The shared file is the usual home, because it is where seats are
added and swapped for both tools; a value in ship-feature's own config wins over it silently, which
is the same stale-copy problem one level up. A list typed into a command is a copy of that file that stops
being true the day it changes, and nothing announces the difference: the round simply comes back
smaller and reads as complete. Pass the flag to override deliberately, not to restate the config.

**Read the startup lines and say which reviewers actually ran.** Omitting the flag fixes a stale
list; it cannot tell you a seat dropped out. Here, bare `opencode` is a relay-only name — if your
panel falls back to `SHIP_FEATURE_REVIEWERS` and contains it, it is skipped with a warning and the
round still exits `0`. Each seat's dispatch line also shows the model and effort it resolved to.

With no file and no stdin it reads
`./plan.md` when nothing is named — convenient, and NOT where a plan should live: see §1. Reviewers run **read-only** and nothing is written or posted — supported: `claude`
(`--permission-mode plan --safe-mode`), `codex` (`--sandbox read-only`), `cursor` (ask mode, pinned to
`$CURSOR_REVIEW_MODEL` — default `composer-2.5`, Cursor's own model — so Cursor's `Auto` cannot
quietly route the review to a Claude model and have Claude grade a plan Claude wrote, and so the
seat stays out of `grok`'s family too), `glm`
(the opencode runner, model pinned to `$GLM_REVIEW_MODEL` — default `opencode-go/glm-5.3`, the
bundled OpenCode Go tier, overridable to a pay-as-you-go model — pinned read-only by
`OPENCODE_CONFIG_CONTENT` — the highest-precedence config
layer — denying `edit`+`bash`, with inherited `OPENCODE_CONFIG*` unset, plus `--pure`
and `--agent plan`), `grok` (model and effort from `MODEL_grok` / `EFFORT_grok` in the shared panel
file — the same keys the relay's grok seat reads — or `$GROK_REVIEW_MODEL` / `$GROK_REVIEW_EFFORT`;
defaults `grok-4.6` / `medium` — via `grok --prompt-file`, running in your
checkout with a read-only tool allowlist `--tools read_file,list_dir,grep`, the MCP bridge removed
(`--disallowed-tools search_tool,use_tool`), `--permission-mode plan`, `--sandbox read-only`), and `antigravity` (aliases `agy`, `gemini`) via the
`gemini` CLI. The gemini run is fail-closed: an isolated `GEMINI_CLI_HOME` **and** working dir with a
locked `.gemini/settings.json` that **allowlists only the read-only tools** via `tools.core`, names
today's write tools in `tools.exclude` as defence-in-depth, disables hooks, and declares no MCP — so
neither the user's real `~/.gemini` nor a reviewed checkout's `.gemini/` contributes any `mcpServers`,
hooks, or `tools.allowed` (the gemini analog of claude's `--safe-mode`; default non-interactive mode
plus `-e none` on top). Because `tools.core` does **not** filter every tool gemini registers, the seat
also refuses to run against a gemini-cli whose tool registry has not been audited: only the exact
versions in `SHIP_FEATURE_GEMINI_TESTED_VERSIONS` (default `0.26.0`) are accepted, and anything else fails
the seat rather than trusting an unknown registry. Exact, not major.minor — a patch release can add a
tool as easily as a minor one. The version is probed inside the same isolation the review runs in. Model pinned to `gemini-3.1-pro-preview`; override
with `SHIP_FEATURE_GEMINI_MODEL` or `MODEL_gemini` in the shared panel file. **Tradeoff:** because the
run is fully isolated, gemini reviews the plan text only — it has no access to the checkout's files
(deep codebase fact-checking is the PR cross-review's job). The `antigravity` name maps to the `gemini`
CLI **here** but to `agy` in `relay` — only `gemini` has a read-only mode. Bare `opencode` is
relay-only and skipped with a warning. `grok45high` is the old name of the `grok` seat: it still runs
`grok`, with a warning, and its old pins (`MODEL_grok45high`, `GROK45HIGH_REVIEW_MODEL`) are ignored.

`claude` and `codex` take their pins from the same place: `MODEL_claude` / `EFFORT_claude` and
`MODEL_codex` / `EFFORT_codex` in the shared panel file, or `$CLAUDE_REVIEW_MODEL`,
`$CLAUDE_REVIEW_EFFORT`, `$CODEX_REVIEW_MODEL`, `$CODEX_REVIEW_EFFORT` in the environment (which
wins). Unpinned, they get no extra argument and run whatever their CLI is configured for. These six
pins are read from the shared file and the environment only — set in ship-feature's own config they
are ignored, with a warning.

Reviewers that run in the checkout — `claude`, `codex`, `cursor`, `glm`, `grok` — can read the
tree, so they check the plan against the code (a stale line number, a test the change would turn red).
`glm` has three states, in this order: it runs in the checkout; it falls back to an isolated empty
directory when the checkout carries its own opencode config (`opencode.json`, `opencode.jsonc`,
`.opencode`), because that config could otherwise reconfigure the reviewer, and it says so above its
output; and it is **skipped entirely** if that isolated directory cannot be created, rather than
running against a tree it cannot vouch for. `grok` needs a working OS sandbox to read the tree:
where one cannot be applied (Linux without bubblewrap) it degrades to an isolated, text-only review and
says so above its output.

What "read-only" means here, precisely: each reviewer is pinned to its CLI's read-only mode and none of
them is given a way to write your checkout or post anywhere. It is not a sandbox escape proof. Except for
`claude --safe-mode`, an in-checkout reviewer loads that checkout's own agent config (`CLAUDE.md`,
`AGENTS.md`, `.cursor`, `.grok`) and whatever hooks or plugins it declares — the same trust you already
extend by opening the repo in that agent. `glm` is in that group too — it reads the checkout. The one
seat that never does is `antigravity`/`gemini`, which is fully isolated and sees the plan text only, so
review a plan for a repository you do not trust with that seat, or not at all.
Exit `0` = every reviewer responded, `3` = a
reviewer failed/timed out/returned empty (re-run — see the two-consecutive-failure rule below), `1` =
usage error. The single-reviewer default still works too:
`cat ~/.config/ship-feature/plans/<repo>-<slug>.md | codex exec --sandbox read-only`.

Read the feedback and revise — but only within the cap below, not without limit. The reviewers catch
wrong assumptions and stale facts before they become code.

**Loop-termination rule.** A plan under review that keeps drawing new objections to each revision can
spin as long as an unreviewed PR can — one real round ran **13 rounds** before the panel agreed, burning
roughly an hour and a quarter on convergence instead of on the plan. Plan-review gets the same discipline
as step 5's cross-review, in its own terms: this is a **plan** under read-only review before any code
exists, not a **PR** under cross-review, so "plan-qualifying" below is a distinct term from step 5's
"qualifying Should-fix" (correctness / safety / deployability / verification) — do not conflate the two.

- **A round** is one `plan-review` invocation where every dispatched reviewer returned a review (exit
  `0`). Exit `3` (a reviewer failed, timed out, or returned empty) does **not** count as a round —
  re-run it. If the **same** reviewer hits exit `3` on **two consecutive attempts** at the same round,
  stop retrying that reviewer, drop that reviewer for the round, and proceed with the remaining panel
  — but **tell the human about the reduced panel** before Gate 1, the same way a benched reviewer in
  step 5's initial or closing round is the human's call, not a silent substitution. **If dropping the
  reviewer leaves zero reviewers, that is not a completed round at all** — stop and take the failure
  itself to Gate 1 (a plan nobody reviewed is not "the agreed plan").
- **Cap: 2 rounds.** Round 1 is the full configured panel (`SHIP_FEATURE_PLAN_REVIEWERS`, or a
  `--reviewers` override). If round 1 raises no Blocker and no plan-qualifying Should-fix, **stop — go straight
  to Gate 1.** Do not spend a second round confirming an already-clean plan. If round 1 raised a
  plan-qualifying finding, revise **once** and run round 2 against **the same full panel — never a
  narrowed one.** A plan-review panel is 2–3 reviewers, too small to narrow the way step 5 narrows a PR
  panel; unlike step 5, there is no narrow phase here at all.
- **Plan-qualifying** means: a **Blocker**, or a **Should-fix** saying the plan is wrong about the
  existing tree, unsafe to execute as written, or **materially incomplete** — a missing edge case, an
  unhandled failure mode, or missing verification that would let a broken change ship. A **Nit**, or a
  Should-fix that only expresses a stylistic or approach preference without any of the above, does not
  reopen a round.
- **If round 2 still has an open plan-qualifying finding, stop — do not run a round 3 on your own.**
  Write a short **disagreement summary**: for each open finding, the reviewer's objection, your
  classification, and the reason. Take that summary to Gate 1 instead of "the agreed plan."
- **A human-authorized extra round is not a third autonomous round.** If the human reviews the
  disagreement summary at Gate 1 and asks for one more pass, that round runs like round 2: it either
  resolves and reaches Gate 1 clean, or it produces an updated disagreement summary and returns to
  Gate 1 again. There is no agent-initiated round beyond the cap — only a human-authorized one.

### 3. 🚦 Gate 1 — human approves the plan
Summarize the plan and **stop**. This is one of two shapes: **the agreed plan** (a clean round, or a
later round resolved every plan-qualifying finding), or **the plan plus a disagreement summary** (the
cap was hit with a plan-qualifying finding still open). In the second shape the human chooses: accept
the plan as you classified it, authorize one more round with explicit reasoning for why, or drop the
change. Do not touch the source repository until the human says go.

### 4. Implement in a worktree, open a PR
- Work in a **git worktree**, never the main working tree. Default root `.claude/worktrees/`, override
  with `SHIP_FEATURE_WORKTREE_ROOT`. Add the worktree marker dir to the repository's `info/exclude` (in a
  linked worktree that's the **common** git dir's exclude, which `ship-feature preflight` checks) so it
  never shows up in anyone's status.
- Branch off the repository's **default branch** (`origin/HEAD`), not a hardcoded name.
- **Stage explicit paths** — never `git add -A`/`git add .` (tools drop stray files).
- One coherent change per PR. Ship tests at the level you touched.
- Open the PR.

Run `ship-feature preflight` to assert the working copy is set up correctly before you start.

### 5. Cross-review + tests
Hand the PR to the **other** agents for review, and run the full test suite / CI against the exact PR
head.

```
# Do NOT name the reviewers: `relay` injects your configured quorum. See below.
# ALWAYS pass the plan: reviewers that cannot see the intent can only find bugs.
ship-feature relay --author <self> \
  --context-file ~/.config/ship-feature/plans/<repo>-<slug>.md
```

**Do not pass `--reviewers` here either.** `relay` injects `SHIP_FEATURE_REVIEWERS` when you omit
it — `SHIP_FEATURE_REVIEWERS` and nothing else: from the environment if exported there, else
`~/.config/ship-feature/config`, else your `REVIEWERS` line in `~/.config/pr-review-relay/config` —
so the config is the panel and there is nothing to keep in sync. An **exported** value beats both files — see the Quorum note in §5 — so check the environment first when a round comes back the wrong size. The failure this prevents is quiet: an agent carrying
last month's list runs a smaller panel than the one you configured, every round exits `0`, and the
reviews read as complete. Pass the flag for the **narrow** rounds described below, where a subset
is the point, and to override on purpose. And read the startup lines every round: a benched
(out-of-quota) seat is dropped and the run still exits `0`.

**Always pass the plan with `--context-file`.** The plan already exists — §2 wrote it for the plan
review — and the flag prepends it as a document every reviewer must verify the PR *against*.

This is also why §1 insists the plan lives in `~/.config/ship-feature/plans/`: this flag is needed on
every round, so the file has to outlive the session that wrote it. A relay that cannot find its context
file refuses to start, and rewriting the plan at that point is worse than running without one — a plan
authored *after* the code, to check the code against, only ever agrees with it. Without
it a reviewer can find bugs but cannot find **"this is not what we agreed to build"**: the plan
conformance check, the stale count, the rule the implementation quietly skipped. Measured on
`pr-review-relay` #23: same number of findings, different kind, and the first review to open with
"the PR matches the plan" and a conformance checklist.

**The plan file is the source, and it is immutable once approved.** Write and revise it freely before
Gate 1 — that is what step 1 and step 2 are for — and not after. The PR body is generated from it
(`gh pr edit --body-file <the plan>`), never hand-edited, so the two cannot drift. When later rounds
need dispositions (below), build a **derived** per-round file — the plan verbatim, plus a
Dispositions section — and point `--context-file` at that. Never hand-edit either one.

**Do not re-publish the derived file to the PR body.** `gh pr edit --body-file` runs for the plan and
only the plan; a Dispositions section that grows every round would bury the plan under the argument
about the plan. Dispositions belong in the round's prompt, where the reviewer reads them, and in the
PR thread, where the human reads them at Gate 2.

**Read the exit code** (`ship-feature relay` preserves it):
- `0` — every dispatched reviewer ran and posted **against a stable SHA**. This does **not** mean the
  reviews are clean — you must READ the verdicts.
- `3` — the round is not trustworthy (a reviewer failed / SHA unreadable / HEAD moved). Re-run.
- `4` — the round cap was hit. Escalate to the human.

**Quorum:** set it once — `SHIP_FEATURE_REVIEWERS` in `~/.config/ship-feature/config`, or
`REVIEWERS` in `~/.config/pr-review-relay/config`, not both — and then **do not pass a list on the
command line**.

**The environment wins over both files**, because `load_config` and `load_shared_panel_config` fill
these names only when they are not already defined. So a `SHIP_FEATURE_REVIEWERS` exported in a
shell profile months ago silently decides today's panel, and it is the first place to look when a
round comes back the wrong size. Note the two commands read different names: `relay` uses
`SHIP_FEATURE_REVIEWERS` only, while `plan-review` prefers `SHIP_FEATURE_PLAN_REVIEWERS` and falls
back to it. `relay` injects the configured quorum when you omit `--reviewers`, which is a hard
failure on a missing agent in exactly the way a typed list is, and cannot go stale the way a typed
list does. A partial pass must not read as consensus, so also read each round's startup lines: a
benched (out-of-quota) seat is dropped and the round still exits `0`.

**Loop-termination rule.** Iterating on *every* Should-fix produced eight full-quorum rounds on one
PR where the last five found no Blocker. The loop now has **three named phases**:

| phase | panel | purpose |
|---|---|---|
| **Initial** — round 1 | full quorum | find everything |
| **Narrow** — rounds 2…n | reviewers with a stake in an open finding, **plus any whose finding you downgraded** | verify targeted fixes |
| **Closing** — final round | full quorum, on the SHA that will merge | confirm the merge candidate |

**A clean initial round IS the closing round.** If round 1 completes with no Blocker and no qualifying
Should-fix, you are done — there is nothing to fix, so the SHA the panel reviewed is already the SHA
that will merge. Do not run a second full quorum to "close" a round that never opened. The closing
round exists only because *fixing* something creates a SHA nobody reviewed in full.

**If the closing round raises a new qualifying finding, it becomes the initial round of another
cycle** — fix, narrow, then a full quorum again on the new merge candidate. Not fix-narrow-stop: that
would merge a SHA that no full panel ever saw, which is the thing the closing round is for.

- **A round only counts if it completed** — every reviewer **the round was supposed to dispatch**
  produced and posted a review. For an initial or closing round that is every configured reviewer
  except the author; for a narrow round it is the narrow set, not the whole `SHIP_FEATURE_REVIEWERS`
  list. Exit `0` is *not* that statement: a **benched** reviewer (out of quota) is
  dropped deliberately and still exits `0`. A benched seat in an **initial or closing** round is a
  decision for the human **before the round**, taken when the relay announces the bench at startup:
  wait for the quota to reset, or authorise the reduced panel explicitly, recorded in the PR thread
  and repeated at Gate 2. **Narrow rounds are unaffected**: a benched seat there simply means the
  finding it raised stays open until that seat comes back — nothing to decide, nothing to wait for.
- **Iterate only for a Blocker, or a qualifying Should-fix** — one that is a material problem of
  **correctness, safety, deployability, or verification**. That covers documentation that
  misinstructs, release and deploy configuration, packaging, migrations, portability and API
  contracts; it does not cover wording, structure, or coverage of paths the PR does not touch.
- **The author classifies in writing**, naming the finding and the reason.
- **The narrow panel is defined by stake, not by size**: every reviewer that raised a qualifying
  finding, **plus** every reviewer whose finding the author downgraded — otherwise classifying
  someone out of the panel removes their only way to object. Usually that is fewer seats, which is
  where the saving is; if everyone has a stake, it is everyone.
- **A downgraded reviewer must be able to see the downgrade.** Reviewers are told not to read the PR
  discussion, so dispositions travel in the **derived context file** — each finding, its
  classification, the reason, and whether it is disputed.
- **Post the dispositions before re-running.** The relay deletes each reviewer's previous comment
  before posting the next, so the original finding and its classification are gone from the PR by the
  time the human reaches Gate 2 unless they were written down separately. The disposition comment
  **must not contain the relay's own marker text** (`<Name> review (automated cross-review)`), or the
  next round deletes it too.
- **Non-qualifying findings are recorded and left unfixed** until a follow-up PR. Fixing one creates
  a commit after the reviewed SHA, and then the merged code is not the code anyone reviewed.
- **Stop** when the closing round returns no Blocker and no undisputed qualifying Should-fix.
  Documented **disputes do not block the stop** — they travel to Gate 2 as open items.

The round cap (exit `4`) is an **escalation** threshold: stop and hand the open findings to the
human. It is never permission to merge, and never grounds for `--reset`.

**Exact-SHA rule:** capture the PR `headRefOid`; require local `HEAD` to equal it before testing; run the
tests; then re-check that the PR head has not moved. CI must check out the PR **head** SHA
(`github.event.pull_request.head.sha`), not the synthetic merge commit, so "green" maps to the reviewed
code.

### 6. 🚦 Gate 2 — human merges
Summarize the state and **stop**. The **human** merges. The agent never self-merges — merge authority
stays with the human.

The summary must state: green on the PR head; **no open Blockers and no open qualifying Should-fix**;
every remaining finding recorded with its classification; **every disputed classification listed with
both positions**; and any reduced panel that was authorised. The human rules on the disputes as part
of deciding to merge — accept the downgrade and merge, or reject it, which sends the PR back into the
narrow/closing cycle with that finding reclassified as qualifying.

This is still **two** gates, not three: a dispute does not create a new stopping point, it changes
what Gate 2 receives.

### 7. Verify on the merge commit
After the merge, re-run the tests on the merge commit. Red = stop and fix.

### (8. Release — out of scope here)
Releasing to a store / registry is a separate tool's job and is not part of this workflow.

---

## Autonomous mode (experimental, off by default)

A closed loop with no human in it: the plan panel reaches consensus (replacing Gate 1), the agent
implements, the relay panel reaches consensus, and the merge gate merges (replacing Gate 2). Nothing
here changes the default pipeline above; without the flags every command behaves as before.

**What this is, and is not.** It is a guard against accidents and a record of intent. It is **not a
security boundary against the agent**: the agent runs as the user, with the user's `gh` token, so
anything it can write (a file, an environment variable, a PR comment) it can forge. Each brake below
makes the unsafe path need a deliberate, visible, logged act and fails closed on missing evidence.
The real protection is unanimity of the full panel on the exact SHA, CI, tests and the repo's gate.

**Two keys.** The autonomous commands run only with **both** the `--autonomous` flag and
`SHIP_FEATURE_AUTONOMOUS=1` in the environment (the user sets the variable on purpose when launching
the session). One key alone is refused (exit `7`).

**Kill switch.** If `~/.config/ship-feature/AUTONOMOUS_OFF` exists, every autonomous command refuses
(exit `7`), checked at the start and again right before the merge call.

**Replacing Gate 1: `ship-feature plan-consensus <plan-file> --autonomous`.** Runs the full
`PLAN_REVIEWERS` panel (never narrowed, no `--reviewers`, a file path and never stdin). Every seat must
end its review with exactly `VERDICT: clean` as the last line, and be the only `VERDICT:` line, in the
SAME round. Cap: 2 rounds and 3 attempts per plan path; a round counts only when every seat returned a
verdict. Exit `0` consensus (a result file bound to the plan's SHA-256 is written), `2` findings: revise
and run once more, `3` a seat failed or gave no usable verdict, `5` stop and notify. There is no round 3
and no reset. The panel is `PLAN_REVIEWERS` in the relay config. A relay-only seat in it, a value that
names no seat, an unset `PLAN_REVIEWERS`, or a `SHIP_FEATURE_PLAN_REVIEWERS` override (environment or
ship-feature config) that names a different set is exit `5`, before any seat runs. A counter that cannot
be written also stops before the panel runs (exit `5`). A re-run after consensus on the same text, path
and panel exits `0` without running the panel again.
Seats that review the plan text only are named in the result as `degraded` and in the audit comment.

**The PR body carries the plan's key.** When the PR is opened from a plan that reached consensus, its
body has one line `Plan-Key: <sha256 of the plan text>`. The merge gate reads it to find the consensus
result; a PR without exactly one key is refused.

**The loop: `ship-feature relay --autonomous --context-file <plan>`.** Every round is a full-panel round
on the relay's own quorum (`--reviewers`, `--verdict-file`, `--max-rounds` and `--reset` are refused, and
so is a round without `--context-file`). Exit `0` is an `all_clean` round, exit `2` a counted round with
findings. A verdict file that is missing, names another repo or PR, or has no `all_clean=true|false` is
removed and counts as a failed attempt (exit `3`). A round state that cannot be written stops before the
relay runs (exit `4`). The verdict file is
written at a path the tool computes. One counter per PR (not per SHA) allows at most 6 rounds; the 7th
exits `4`, and so does the 4th failed attempt. The wrapper records the PR round in the verdict file as
`pr_round` (the relay's own `round` counts per SHA). The relay also gets `--max-rounds 6`, which it
applies per SHA and counting failed rounds too, so it can stop a SHA earlier (exit `4`, a safe stop). Fix every Blocker and Should-fix, post the disposition,
run the next round. The first round whose verdict is `all_clean` on the final SHA is the merge candidate.

**Disposition and dispute comment formats** (the merge gate reads whole lines only). A disposition
comment by the PR author starts with the line `Disposition, round <k> (reviewed sha <sha7>):`, lists
each finding with its classification, and ends with the line `Disputed findings: none` (or
`Disputed findings: <n>`) as its last non-empty line. A finding the author disputes is a line that starts with `- DISPUTED: <text>`;
the latest disposition must repeat every dispute settled so far as `- RESOLVED: <text>`. Dispute lines
by the author count in review comments and review bodies too. Autonomous mode never merges over
a dispute, because the human who settled disputes at Gate 2 is gone.

**Replacing Gate 2: `ship-feature merge-gate <pr> --autonomous [--merge]`.** It checks, in order, and
stops at the first failure (exit `6`); without `--merge` it only reports:
1. both keys present, kill switch absent (exit `7`);
2. plan binding: one `Plan-Key`, a consensus result for it, the plan panel unchanged, the plan unedited;
3. the PR is open, not a draft, based on the repo's default branch, and `MERGEABLE` (`UNKNOWN` is read
   up to 5 times);
4. the verdict file of the highest round is for this repo and PR, names the PR's current head SHA, has
   `all_clean=true`, round at most 6, and the panel equals the relay's configured `REVIEWERS` without
   the author;
5. no unresolved dispute and a latest disposition whose last non-empty line is `Disputed findings: none`;
6. CI on that SHA: every check `SUCCESS` or `NEUTRAL` (nothing skipped, pending or missing); an empty
   result needs the repo listed in `~/.config/ship-feature/allow-no-ci`, a file a human creates;
7. no sensitive file in the diff: the patterns in `~/.config/ship-feature/sensitive-paths`, else the
   built-in defaults (workflows, hooks, deploy and release scripts, keys, env files, anything named
   secret or credential); a match leaves the PR to a human, and a patterns file that cannot be read is
   exit `6`;
8. the head is re-read and the kill switch re-checked, then exactly one
   `gh pr merge <n> --merge --match-head-commit <verdict sha>` runs, followed by an audit comment (head
   SHA, merge commit, the verdict file, degraded seats, CI, files checked, `autonomous_env=1`). The text
   is kept in memory and also written to `audit.txt` when the state directory allows it. If the audit
   comment cannot be posted the command exits `8` and prints the text: post it.

It never fixes, never retries a failed merge call and never loops.

**Locks.** Each round and each plan-consensus run holds a lock directory, so two sessions cannot pick
the same round. A stale lock (owner gone, or no pid after a minute) is taken over inside a second
directory, `<lock>.takeover`. A takeover directory left by a crash is never removed by the tool: it
blocks only takeovers, and a human removes it.

**Stop and notify.** Exit `4`, `5`, `6`, `7` or `8`, exit `3` after the bounded retries, a failing
hook, or a dispute: report it in the conversation, send a notification if the harness has a tool for
it, and wait. Never try another route to continue.

**Everything else still binds:** worktree, tests with every change, the repo's gate and hooks (never
`--no-verify`), explicit staging, the PR body generated from the plan, no AI attribution lines,
dispositions posted before each re-run, no `--reset`.

**Residual risks.** A seat can print `VERDICT: clean` over a wrong review. Degraded seats vote. The
author can write a false `Disputed findings: none`. The panel definition is a config file the agent
can edit; the audit comment records who ran. A merge is not undone by the tool: roll back with a
revert PR.

---

## The light path

`--light` is for a trivial change (a typo, a one-line fix) and is **only** valid when the human explicitly
asks for it. It may skip the **plan review** (step 2). It may **never** skip the worktree, the relevant
tests, the cross-review, or the merge gate.

---

## Scope

Single repository, `origin/HEAD` as the base. Multi-repo changes are out of scope — split them into one
PR per repo, each run through this workflow independently.

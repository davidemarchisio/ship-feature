<!--
  This block is inserted into ~/.codex/AGENTS.md by ship-feature's install.sh, between the markers
  below. Edit it in the repo (adapters/codex/AGENTS.snippet.md), not in AGENTS.md — re-running install.sh
  replaces the marked block idempotently.
  It is a POINTER, not a copy of the rules: the lanes live in the ship-feature skill and in WORKFLOW.md.
  It used to hold a full copy of the lanes, always loaded, which drifted from the skill. Only the one
  rule that loses money when it is missed stays here: the careful lane's human merge.
-->
# >>> ship-feature >>>
## ship-feature workflow (default for repository changes)

When asked to **implement / build / add / fix / ship a change to a repository** (not a read-only task
like diagnosis, review, or a question), load the **ship-feature** skill before your first change and
follow it: `~/.agents/skills/ship-feature/SKILL.md`. It picks the lane (light by default, careful for
risky changes) and gives the steps. The details are in `~/.config/ship-feature/WORKFLOW.md`. If
neither file exists, stop and tell the human: the install is broken.

One rule holds before you read them: payments and subscriptions, ad consent and ad revenue,
authentication and account deletion, database migrations and data deletion, security fixes and
store-policy changes are the careful lane, where a human approves the plan and merges. Never merge
those yourself.
# <<< ship-feature <<<

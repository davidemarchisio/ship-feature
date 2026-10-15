---
description: Always applied. Points at the ship-feature skill, the workflow for shipping any repository change.
alwaysApply: true
---

# ship-feature workflow

When asked to **implement / build / add / fix / ship a change to this repository** (not a read-only task
like diagnosis, review, or a question), load the **ship-feature** skill before your first change and
follow it: `~/.agents/skills/ship-feature/SKILL.md`. It picks the lane (light by default, careful for
risky changes) and gives the steps. The details are in `~/.config/ship-feature/WORKFLOW.md`. If
neither file exists, stop and tell the human: the install is broken.

One rule holds before you read them: payments and subscriptions, ad consent and ad revenue,
authentication and account deletion, database migrations and data deletion, security fixes and
store-policy changes are the careful lane, where a human approves the plan and merges. Never merge
those yourself.

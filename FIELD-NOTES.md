# Field notes

Runs that produced the rules in `SKILL.md`. Kept out of the skill on purpose:
the skill states what to do, this file records why. Nothing here is loaded into
a session's context — read it only when a rule looks arbitrary and you want the
run behind it. Newest first.

## 2026-08-20 — `open-learning-exchange/myplanet` PR #15771

**Orientation cost ~15 tool calls.** The session's entire prompt was the word
`overtaking` — the slug of its own branch `claude/overtaking-cyfn4n` — with the
harness summary reading "branch claude/overtaking-cyfn4n exists; no task
description provided". It spent those calls on git log and branch inspection, a
repo-wide grep for "overtaking", `CLAUDE.md`, `docs/AGENT_SPELLBOOK.md`,
open-issue and open-PR listings, and finally asking the user what to work on —
who answered with the PR URL and the words "skill branch-overtaking".

Three signals were in plain sight:

- `get_session` → `sources[0].git_repository.revision` was
  `refs/heads/15634-unable-to-download-html-resources-download-keeps-failing`, a
  **non-default** branch (this repo's default is `master`, not `main`), while
  `outcomes[]` was the session-derived `claude/overtaking-cyfn4n`. That pair is
  the takeover signature. One call on turn one would have shown the whole
  situation.
- The prompt, and the branch slug it came from, named the skill.
- The skill was not installed: `~/.claude/plugins/installed_plugins.json` was
  `{"plugins": {}}` and myplanet's `.agents/skills/*` submodules were
  uninitialized, though its `.claude/settings.json` lists the marketplaces.
  Reading `SKILL.md` took a manual `git clone`.

→ *First move* section.

**The invoking session did not stop.** It did job 1 (spawned a session with
`source_revision` and `outcome_branch` both set to the PR's branch — verified
bound) and job 2 (`subscribe_pr_activity` on the PR, opened by another
contributor), then kept going: pulled the PR's check runs, read all 5 unresolved
CodeRabbit review threads, verified 4 of the findings against the code, and
started downloading the Android SDK to run the test suite locally — until the
interrupt "stop we are good". Every one of those tokens was waste; the spawned
session is where that work belongs.

The old wording, "Then stop — do not start working inside it", read as "don't
work inside the spawned session" and never told the invoking session to end its
turn.

→ Job 1 step 5 *Verify, report, and stop*.

**Where the subscription lives.** Stopping right after the spawn broke job 2's
"subscribe in the session that is doing the work — not in a spawned one": a
subscription in a session whose turn is over wakes it with nothing to do. Two
options — the spawned session subscribes once the user drives it, or the spawn
carries a minimal prompt that only subscribes. The second won, because this same
run shows the skill is often absent from the container: an unprompted spawned
session has no way to know it should subscribe.

→ Job 1 step 4's seed prompt, and job 2's spawned/not-spawned split.

## 2026-08-13 — `open-learning-exchange/myplanet` PR #15555

A takeover of a PR opened by another contributor sat event-silent until an
explicit `subscribe_pr_activity` call, then delivered CI failures and review
comments normally. Confirms job 2's premise: authorship does not make the
subscription impossible, only absent.

## 2026-08-12 — session bound to the wrong outcome

A session started *from* the PR's branch
(`refs/heads/claude/kotlin-flutter-dart-migration-d3gmrd`), yet its outcome was
still an auto-minted `claude/session-sof3wd`. Every commit landed on the PR's
branch; the panel stayed bound to a branch that never received one.

→ *Why the panel is not attached*, and job 1 spawning a session rather than
trying to repair the current one.

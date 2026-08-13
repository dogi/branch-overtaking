---
name: overtaking
description: 'Take over an existing branch and pull request from a Claude Code session. Invoked bare, it resolves the repo and branch this session was opened with, finds the oldest open PR on that branch, and spawns a session whose source revision *and* outcome branch are both that branch — which is what makes the web UI attach its PR info panel. Also covers putting commits on a PR someone else opened without minting a new branch or a second PR, and the stale same-named local lineage that silently makes you build on the wrong history. Use whenever asked to take over, adopt, continue, or push onto an existing PR or branch, when told "commits go onto that existing PR", when a session pushed to the wrong branch, or when someone asks why the PR panel or info sidebar will not attach to their session.'
---

# Taking over an existing branch and PR

Two different jobs wear the same name, and conflating them wastes an afternoon:

1. **Session binding** — getting the web UI's PR info panel to attach. This is
   *not* controlled by what you push, and no amount of correct pushing fixes it.
   It needs a session whose **outcome branch** is the PR's branch.
2. **Git takeover** — getting commits onto an existing branch so an existing PR
   shows them. This nearly always works first try.

Invoked bare ("take over this PR", "/over-taking"), do job 1: the user is looking
at a session that cannot show them their PR. Say which job you are doing.

## Job 1 — bind a session to this branch's PR

Nothing needs to be supplied. Resolve it all, then act.

### 1. What repo and branch was this session opened with

```
get_session(session_id: <this session>)
  → session_context.sources[0].git_repository.url        # https://github.com/<owner>/<repo>
  → session_context.sources[0].git_repository.revision   # refs/heads/<branch> — strip the prefix
  → session_context.outcomes[0].git_repository.git_info.branches   # what the panel binds to
```

The **source revision** is the branch the session was opened with — use that,
not the checked-out branch, which a user's redirect may have changed mid-session.
Cross-check with `external_metadata.current_branches` and, if `get_session` is
unavailable, fall back to `git remote get-url origin` plus
`git rev-parse --abbrev-ref HEAD`.

### 2. Is anything actually broken

Compare the source branch against `outcomes[].branches`:

- **Outcome already contains the branch** → the panel is already bound. Do
  **not** spawn anything; say so and stop. Spawning a duplicate session against
  a branch that already has one invites two sessions pushing to it.
- **Outcome is a different, session-derived name** (e.g. `claude/session-ab12cd`
  when the source was `claude/my-feature`) → this is the failure. Continue.

### 3. The oldest open PR on that branch

```
list_pull_requests(
  owner: <owner>, repo: <repo>,
  state: "open",
  head: "<owner>:<branch>",          # fork PRs use the fork owner here
  sort: "created", direction: "asc",
  perPage: 10)
```

Take the **first** result — the oldest is the canonical one; later PRs on the
same head are usually accidents. If several come back, name them all and say
which you picked. If none come back, stop and report that the branch has no open
PR: there is nothing to bind a panel to, and spawning a session would not help.

### 4. Spawn the bound session

```
create_session(
  source_url:      https://github.com/<owner>/<repo>
  source_revision: <branch>
  outcome_branch:  <branch>        ← the load-bearing argument
  title:           PR #<n> — <short description>)
```

`outcome_branch` is the field the web UI does not expose, and the whole reason
this skill exists. Send **no prompt**: the user drives it, and an unprompted
session sits idle instead of burning tokens. Then stop — do not start working
inside it.

### 5. Verify, then report

```
outcomes: branches: ["<branch>"]     ← must be the branch, not a derived name
```

Report the new session id, its title, and the PR URL. Two things to state
plainly rather than gloss:

- This created **no branch and no PR** — it points at the existing ones. Say it
  out loud to anyone who asked for no new artifacts.
- Sessions spawned this way carry `origin: claude_code_mcp_seed`, not
  `web_claude_ai`. **Whether the panel renders for that origin is unverified.**
  Ask the user what they actually see; do not promise the panel. It is reversible
  either way — `archive_session` cleans it up.

## Job 2 — putting commits on the PR

### Never trust the local branch ref

A session handed `claude/some-feature` may have a **local ref of that name that
is not the PR's head** — rebased or force-pushed from another session after this
container was provisioned. Same name, different history, often a *subset* of the
work, and it looks current.

```bash
git fetch origin <branch>
git rev-parse HEAD origin/<branch>          # equal? you are current
git status -sb | head -2                    # "[ahead N, behind M]" = diverged
git log -1 --format='%h %ci %s' <branch>
git log -1 --format='%h %ci %s' origin/<branch>
```

The PR's head SHA is the authority — read it off the PR (`pull_request_read`,
`method: "get"` → `head.sha`), not off a local ref.

**Divergence is not ambiguity.** Decide with evidence:

- **Dates.** `%ci` on both tips. A local tip a week older than the remote is
  stale leftovers, not unmerged work.
- **File sets**, because counts lie — `rev-list --count` is meaningless in a
  shallow clone:

  ```bash
  git ls-tree -r --name-only origin/<branch> -- <subdir> | sort > /tmp/remote.txt
  git ls-tree -r --name-only <branch>        -- <subdir> | sort > /tmp/local.txt
  comm -13 /tmp/remote.txt /tmp/local.txt    # files ONLY local — candidate lost work
  comm -23 /tmp/remote.txt /tmp/local.txt    # files ONLY on the PR head
  ```

  An empty first list means the local lineage is a strict subset: safe to discard.

- **Subject matching is worthless after a rebase** — every commit has a new SHA
  and often a reworded subject, so "these subjects are missing upstream" proves
  nothing.

When the remote is authoritative: `git reset --hard origin/<branch>`. If the
local branch genuinely carries work the PR head lacks, rebase it onto the PR head
instead — and say so before touching anything.

### Push, adding nothing

```bash
git push -u origin <branch>
```

- **No new branch. No second PR.** Someone asking you to take over a PR wants
  commits on *that* PR; another one is the outcome they explicitly did not want.
- Retry network failures with backoff (2s, 4s, 8s, 16s). Do not retry an auth or
  non-fast-forward rejection — diagnose it.
- **A merged PR cannot be reused.** Restart the branch from the current default
  branch and treat the follow-up as new work:
  `git fetch origin <default> && git checkout -B <branch> origin/<default>`,
  rebasing any unmerged commits onto the new base.

## Why the panel is not attached

A web session records two different things, and the panel follows the second:

```
sources:  refs/heads/claude/kotlin-flutter-dart-migration-d3gmrd   ← what you checked out
outcomes: branches: ["claude/session-sof3wd"]                      ← what the panel binds to
```

A real session (2026-08-12): started *from* the PR's branch, yet the outcome was
still an auto-minted session-derived name. Every commit landed on the PR's
branch; the panel stayed bound to a branch that never received one.

The outcome is **fixed at session creation**. `set_session_title` and
`set_session_tags` are the only session mutators available and neither touches
the binding — which is why job 1 spawns a session rather than repairing this one.

## PR events in the conversation (optional, often refused)

`subscribe_pr_activity` routes comments and CI failures into the session,
independently of the panel. On a PR you did not open it frequently fails:

```
Could not subscribe to this PR.
```

The documented cause is another watcher — a PR Steward holding a watching label.
Attempt it **once** (twice only if a transport error genuinely intervened), then
stop: a refusal is not flakiness, and repeating it is not diagnosis. The unblock
is removing that label, which is the user's call.

## Racing

Two sessions registered to the same outcome branch can both push to it. When a
second one is bound, agree which pushes and say so — otherwise a force-push or a
non-fast-forward rejection arrives with no explanation.

## Checklist

- [ ] Source repo and branch read from the session, not guessed
- [ ] `outcomes[]` checked first — already bound means stop, not spawn
- [ ] Oldest open PR on the head chosen, others named
- [ ] Spawned session's `outcome_branch` verified equal to the branch
- [ ] No prompt sent to it; no branch, no PR created
- [ ] Panel rendering confirmed **by the user**, never assumed
- [ ] For commits: divergence settled by dates and file sets before pushing

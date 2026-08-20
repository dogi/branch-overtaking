---
name: overtaking
description: 'Take over an existing branch and pull request from a Claude Code session. Invoked bare, it resolves the repo and branch this session was opened with, finds the oldest open PR on that branch, and spawns a session whose source revision *and* outcome branch are both that branch — which is what makes the web UI attach its PR info panel. It also subscribes the session to activity on that PR, without which a PR someone else opened never delivers its CI failures or review comments here. Also covers putting commits on a PR someone else opened without minting a new branch or a second PR, and the stale same-named local lineage that silently makes you build on the wrong history. Use whenever asked to take over, adopt, continue, or push onto an existing PR or branch, when told "commits go onto that existing PR", when a session pushed to the wrong branch, when CI failures or review comments on an existing PR never reach the session, when the bottom panel offers "Create PR" for a branch that already has one, or when someone asks why the PR panel or info sidebar will not attach to their session. Also the first move on a session whose prompt is empty, a single word, or its own branch slug and whose task summary says no task description was provided: call get_session before any git archaeology, issue or PR listing, or asking the user what to work on — a non-default source revision paired with a different, session-derived outcome branch is the takeover signature. Once the bound session is spawned, the invoking session reports and ends its turn.'
---

# Taking over an existing branch and PR

Three different jobs wear the same name, and conflating them wastes an afternoon:

1. **Session binding** — getting the web UI's PR info panel to attach. This is
   *not* controlled by what you push, and no amount of correct pushing fixes it.
   It needs a session whose **outcome branch** is the PR's branch.
2. **Event subscription** — getting the PR's CI failures and review comments to
   arrive in *this* conversation. Independent of the panel, and cheap: one call.
3. **Git takeover** — getting commits onto an existing branch so an existing PR
   shows them. This nearly always works first try.

Jobs 1 and 3 do not care who opened the branch or the PR, or what its name looks
like — a teammate's branch, a bot's, a fork's, `alice/redesign-nav`, a bare
`fix-typo` with no slash at all, is handled exactly like one this session
itself created. The `claude/…` names that show up below are just realistic
examples of what a Claude-Code-web-originated session happens to look like;
neither job branches on that prefix, or on there being a prefix at
all, and none of it requires the PR's own branch to have been created by
Claude Code.

Job 2 is the one exception, and it runs one way only: a PR **this** session
opened is wired up already, and a PR **someone else** opened is not — it stays
silent until you subscribe to it. The authorship does not make the subscription
impossible, only absent, so the fix is to make the call rather than to work
around it.

Jobs 1 and 2 are **Claude Code web/cloud only**: they depend on `get_session`,
`create_session`'s `outcome_branch`, and `subscribe_pr_activity`, which exist
only in that product surface. Invoked from the CLI, from OpenHands, or from
Copilot there is no session to bind or wake — skip straight to job 3, which is
plain `git`/PR mechanics and works anywhere.

Invoked bare ("take over this PR", "/branch-overtaking:overtaking"), do jobs 1
and 2 *if you're in a Claude Code web/cloud session*: the user is looking at a
session that cannot show them their PR and will not hear from it either. Say
which jobs you are doing.

## First move — orient from the session, not from the repo

On a session whose prompt is empty, a single word, or a bare branch slug, call
`get_session` **first** — before any `git log` or branch inspection, before
grepping the repo, before reading `CLAUDE.md` or an agent handbook, before
listing open issues or PRs, and before asking the user what to work on. One call
decides it:

- `sources[0].git_repository.revision` is a **non-default** branch, **and**
- `outcomes[0]…branches` is a different, session-derived name

→ this is a takeover — the exact signature described in *Why the panel is not
attached*. Do not ask what to work on: go straight to job 1 step 3, look up that
branch's oldest open PR, and proceed.

Compare against the repo's **actual** default branch. It is not always `main` —
`open-learning-exchange/myplanet` is `master`. Cheap, no guessing:

```bash
git symbolic-ref --short refs/remotes/origin/HEAD | sed 's|^origin/||'
git remote show origin | sed -n 's/.*HEAD branch: //p'   # fallback, one network call
```

The prompt is a signal too: a one-word prompt — or the branch slug it was derived
from — that names a skill *is* the instruction to run that skill.

**Expect the skill not to be installed.** `~/.claude/plugins/installed_plugins.json`
reading `{"plugins": {}}`, or `.agents/skills/*` submodules sitting
uninitialized, means nothing triggered on the word even when the repo's
`.claude/settings.json` lists the marketplaces. Fetch it and read it — never
reason from the skill's name:

```bash
git clone https://github.com/dogi/branch-overtaking /tmp/branch-overtaking
```

Receipt (2026-08-20): a session whose entire prompt was the word `overtaking` —
the slug of its own branch `claude/overtaking-cyfn4n`, with the harness summary
reading "no task description provided" — spent ~15 tool calls on git and branch
archaeology, a repo-wide grep for "overtaking", `CLAUDE.md`,
`docs/AGENT_SPELLBOOK.md`, open-issue and open-PR listings, and finally a
question to the user, before its first skill action. `get_session` would have
shown source `refs/heads/15634-unable-to-download-html-resources-…` against
outcome `claude/overtaking-cyfn4n` on turn one — the whole situation, in one
call. The skill itself was absent from the container and had to be cloned by
hand.

## Job 1 — bind a session to this branch's PR (Claude Code web/cloud only)

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
- **Outcome is a different, session-derived name** (e.g. the source was
  `alice/redesign-nav` — someone else's branch, taken over as-is — but the
  outcome auto-minted to `claude/session-ab12cd`) → this is the failure.
  Continue.

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
this skill exists.

Send **one prompt, and only job 2's**:

```
prompt: "Call subscribe_pr_activity(owner: <owner>, repo: <repo>, pullNumber: <n>),
         report the result in one line, then stop and wait for me. Do nothing
         else — no CI checks, no review-thread triage, no code changes."
```

That seed is the whole payload. The subscription has to live in the session that
will do the work, and once the spawn exists that is the spawned session, not this
one — see job 2. Give it nothing beyond that: a spawned session handed real work
starts it unprompted, which is the user's call to make, not yours.

### 5. Verify, report, and stop

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

**The report is this session's last action.** Once the bound session exists, end
the turn: no pulling the PR's check runs, no reading or verifying review threads,
no code fixes, no builds, no test runs, no toolchain downloads. That work belongs
to the spawned session, where the user will drive it — doing it here spends the
tokens twice or throws them away.

Receipt (2026-08-20): a takeover of `open-learning-exchange/myplanet` PR #15771,
opened by another contributor, did job 1 and job 2 correctly — then kept going.
It pulled the PR's check runs, read all 5 unresolved CodeRabbit review threads,
verified 4 of the findings against the code, and started downloading the Android
SDK to run the suite locally, until the user interrupted with "stop we are good".
Every one of those tokens was waste.

## Job 2 — wire the PR's events into this session (Claude Code web/cloud only)

Do this on **every** takeover, in the session that will do the work. Without it,
a PR someone else opened never speaks: CI goes red and a reviewer asks for a
change, and that conversation hears nothing.

Which session that is follows from job 1:

- **Job 1 spawned a session** → the spawned one does the work, so it subscribes
  itself from its seed prompt (job 1 step 4) and this session subscribes to
  nothing. A subscription here would wake a session whose turn is already over.
  Say plainly which session holds it.
- **Nothing was spawned** — already bound, no open PR to bind, or not a web/cloud
  surface → this session is the one doing the work. Subscribe here.

### 1. Find the branch's open PR

The same lookup job 1 does, and it stands alone — run it here even when you
skipped job 1 entirely:

```
list_pull_requests(
  owner: <owner>, repo: <repo>,
  state: "open",
  head: "<owner>:<branch>",          # fork PRs use the fork owner here
  sort: "created", direction: "asc",
  perPage: 10)
```

Oldest result wins, as in job 1. No open PR on the head → nothing to subscribe
to; skip to job 3 and say so.

### 2. Subscribe, unless this session already opened it

```
subscribe_pr_activity(owner: <owner>, repo: <repo>, pullNumber: <n>)
```

A PR this session opened is subscribed by its own creation and needs no call. A
PR anyone **else** opened needs this one explicitly — that is the whole gap this
job closes. Confirmed working (2026-08-13): a takeover of
`open-learning-exchange/myplanet` PR #15555, opened by another contributor, sat
event-silent until this exact call, then delivered CI failures and review
comments normally.

Attempt it **once** (twice only if a transport error genuinely intervened), then
stop. A refusal —

```
Could not subscribe to this PR.
```

— is not flakiness and repeating it is not diagnosis. The documented cause is
another watcher, a PR Steward holding a watching label; the unblock is removing
that label, which is the user's call. Report the refusal and carry on to job 3.

### 3. Tell the user the panel is lying

The bottom panel binds to **session-created PRs only**, so on a taken-over PR it
keeps offering **"Create PR"** no matter how well the subscription works. Once
subscribed, that button is cosmetic — events arrive in the conversation instead
of the panel — and saying so unprompted saves the user from reading it as "not
connected".

**Never press it, and never call `create_pull_request` for a branch that already
has an open PR.** GitHub refuses a second open PR for the same head/base, and
against a different base it succeeds and mints a duplicate — the one outcome
someone asking for a takeover explicitly did not want. See job 3's *Push, adding
nothing*.

## Job 3 — putting commits on the PR (any environment)

Plain `git` and PR-lookup mechanics — no `get_session`, `create_session`, or
`subscribe_pr_activity`, so this job works from the CLI, OpenHands, or Copilot
just as well as from Claude Code web/cloud. It doesn't matter who owns the branch
or the PR either.

### Never trust the local branch ref

A session handed someone else's branch — say a bare `fix-typo`, no slash, no
prefix — may have a **local ref of that name that is not the PR's head** —
rebased or force-pushed by another session after this container was
provisioned. Same name, different
history, often a *subset* of the work, and it looks current.

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
  This holds however loudly the web UI's panel offers "Create PR" — see job 2.
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

An unbound panel has a second tell worth recognising: on a branch whose PR this
session did not open, it shows a **"Create PR"** button where the CI and check
status belongs. That is the panel reporting *its own* binding, not the state of
the branch — the PR exists, and pressing the button would try to open a second
one. Job 2 is the fix for the symptom that actually matters.

## Racing

Two sessions registered to the same outcome branch can both push to it. When a
second one is bound, agree which pushes and say so — otherwise a force-push or a
non-fast-forward rejection arrives with no explanation.

## Checklist

- [ ] `get_session` called on turn one, before git archaeology, issue/PR listing,
      or asking the user what to work on
- [ ] Source repo and branch read from the session, not guessed; "non-default"
      judged against the repo's real default branch, not an assumed `main`
- [ ] `outcomes[]` checked first — already bound means stop, not spawn
- [ ] Oldest open PR on the head chosen, others named
- [ ] Spawned session's `outcome_branch` verified equal to the branch
- [ ] Its only prompt is the `subscribe_pr_activity` seed; no branch, no PR created
- [ ] Panel rendering confirmed **by the user**, never assumed
- [ ] `subscribe_pr_activity` held by the session doing the work — the spawned one
      if job 1 spawned, otherwise this one — for any open PR this session did not open
- [ ] After the spawn, the report ends the turn: no CI, review, fix, build, or
      test work here
- [ ] User told the panel's "Create PR" is cosmetic once subscribed — and that
      pressing it would open a duplicate
- [ ] For commits: divergence settled by dates and file sets before pushing

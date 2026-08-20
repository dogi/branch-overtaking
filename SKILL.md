---
name: overtaking
description: 'Take over an existing branch and pull request from a Claude Code session: find the branch''s oldest open PR, spawn a session whose source revision *and* outcome branch are both that branch — which is what makes the web UI attach its PR info panel — and subscribe that session to the PR''s activity, without which a PR someone else opened never delivers its CI failures or review comments. Also covers putting commits on a PR someone else opened without minting a new branch or a second PR, and the stale same-named local ref that silently builds on the wrong history. Use whenever asked to take over, adopt, continue, or push onto an existing PR or branch, when told "commits go onto that existing PR", when a session pushed to the wrong branch, when CI failures or review comments on an existing PR never reach the session, when the bottom panel offers "Create PR" for a branch that already has one, or when someone asks why the PR panel or info sidebar will not attach. Also the first move on a session whose prompt is empty, a single word, or its own branch slug with no task description: call get_session before any git archaeology, issue or PR listing, or asking the user what to work on — a non-default source revision paired with a different, session-derived outcome branch is the takeover signature. Once the bound session is spawned, the invoking session hands the subscription over — unsubscribing itself from the PR rather than asking whether it should — reports, and ends its turn.'
---

# Taking over an existing branch and PR

Three jobs wear the same name, and conflating them wastes an afternoon:

1. **Session binding** — getting the web UI's PR info panel to attach. *Not*
   controlled by what you push; no amount of correct pushing fixes it. It needs a
   session whose **outcome branch** is the PR's branch.
2. **Event subscription** — getting the PR's CI failures and review comments to
   arrive in the session doing the work. Independent of the panel, and cheap: one
   call.
3. **Git takeover** — getting commits onto an existing branch so an existing PR
   shows them. Nearly always works first try.

Jobs 1 and 2 are **Claude Code web/cloud only**: they need `get_session`,
`create_session`'s `outcome_branch`, and `subscribe_pr_activity`. From the CLI,
OpenHands, or Copilot there is no session to bind or wake — go straight to job 3,
which is plain `git`/PR mechanics and works anywhere.

None of this cares who owns the branch or the PR, or what the name looks like — a
teammate's, a bot's, a fork's, `alice/redesign-nav`, a bare `fix-typo` with no
slash at all, is handled exactly like one this session created. The one exception
is job 2: a PR **this** session opened is wired up already; a PR **anyone else**
opened stays silent until you subscribe to it.

Invoked bare ("take over this PR", "/branch-overtaking:overtaking"), do all three
— jobs 1 and 2 only if you're in a web/cloud session — and say which you are
doing.

## First move — orient from the session, not the repo

On a session whose prompt is empty, a single word, or a bare branch slug, call
`get_session` **first** — before any `git log` or branch inspection, before
grepping the repo, before reading `CLAUDE.md` or an agent handbook, before
listing open issues or PRs, and before asking the user what to work on. One call
decides it:

- `sources[0].git_repository.revision` is a **non-default** branch, **and**
- `outcomes[0]…branches` is a different, session-derived name

→ this is a takeover — the signature described in *Why the panel is not
attached*. Do not ask what to work on: go straight to job 1 step 3 and proceed.

Judge "non-default" against the repo's **actual** default branch. It is not
always `main`:

```bash
git symbolic-ref --short refs/remotes/origin/HEAD | sed 's|^origin/||'
git remote show origin | sed -n 's/.*HEAD branch: //p'   # fallback, one network call
```

A one-word prompt — or the branch slug it was derived from — naming a skill *is*
the instruction to run that skill. **Expect the skill not to be installed:**
`~/.claude/plugins/installed_plugins.json` reading `{"plugins": {}}`, or
`.agents/skills/*` submodules sitting uninitialized, mean nothing triggered on
the word even when `.claude/settings.json` lists the marketplaces. Fetch it and
read it — never reason from the skill's name:

```bash
git clone https://github.com/dogi/branch-overtaking /tmp/branch-overtaking
```

## Job 1 — bind a session to this branch's PR (web/cloud only)

Nothing needs to be supplied. Resolve it all, then act.

### 1. What repo and branch was this session opened with

```
get_session(session_id: <this session>)
  → session_context.sources[0].git_repository.url        # https://github.com/<owner>/<repo>
  → session_context.sources[0].git_repository.revision   # refs/heads/<branch> — strip the prefix
  → session_context.outcomes[0].git_repository.git_info.branches   # what the panel binds to
```

The **source revision** is the branch the session was opened with — use that, not
the checked-out branch, which a user's redirect may have changed mid-session.
Cross-check with `external_metadata.current_branches`; if `get_session` is
unavailable, fall back to `git remote get-url origin` plus
`git rev-parse --abbrev-ref HEAD`.

### 2. Is anything actually broken

- **Outcome already contains the branch** → the panel is bound. Do **not** spawn
  anything; say so and stop. A duplicate session against a bound branch invites
  two sessions pushing to it.
- **Outcome is a different, session-derived name** (source `alice/redesign-nav`,
  outcome auto-minted to `claude/session-ab12cd`) → this is the failure. Continue.

### 3. The oldest open PR on that branch

```
list_pull_requests(
  owner: <owner>, repo: <repo>,
  state: "open",
  head: "<owner>:<branch>",          # fork PRs use the fork owner here
  sort: "created", direction: "asc",
  perPage: 10)
```

Take the **first** result — the oldest is canonical; later PRs on the same head
are usually accidents. If several come back, name them all and say which you
picked. If none come back, stop and report that the branch has no open PR: there
is nothing to bind a panel to.

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

That seed is the whole payload: the subscription must live in the session that
will do the work, and once the spawn exists that is the spawned session, not this
one (see job 2). Give it nothing beyond that — a spawned session handed real work
starts it unprompted, which is the user's call to make, not yours.

### 5. Verify, report, and stop

```
outcomes: branches: ["<branch>"]     ← must be the branch, not a derived name
```

Report the new session id, its title, and the PR URL, plus two things worth
stating plainly:

- This created **no branch and no PR** — it points at the existing ones.
- Sessions spawned this way carry `origin: claude_code_mcp_seed`, not
  `web_claude_ai`. **Whether the panel renders for that origin is unverified** —
  ask the user what they see; do not promise it. `archive_session` reverses this
  either way.

**The report is this session's last action.** Once the bound session exists, end
the turn: no pulling check runs, no reading or verifying review threads, no code
fixes, no builds, no test runs, no toolchain downloads. That work belongs to the
spawned session, where the user will drive it; doing it here spends the tokens
twice or throws them away.

Hand the events over as part of stopping. On a clean takeover there is nothing to
hand over — you never subscribed here (job 2). When a subscription *does* predate
the report — an earlier turn, an earlier takeover, one the user or harness already
had in place — call `unsubscribe_pr_activity(owner, repo, pullNumber)` before you
report. **Do it, don't offer it:** "want me to unsubscribe?" leaves CI failures
and review comments landing in a session that will not act on them, waiting on an
answer the user should never have been asked for. A PR event that still arrives
afterwards gets the same treatment — unsubscribe, name the session that holds it,
and do not triage it.

## Job 2 — wire the PR's events into the session doing the work (web/cloud only)

Do this on **every** takeover. Without it a PR someone else opened never speaks:
CI goes red, a reviewer asks for a change, and the conversation hears nothing.

Which session holds it follows from job 1 — so **run job 1 first and make no
subscribe call until it answers.** Subscribing here and undoing it a moment later
is the same waste spread over two calls.

- **A session was spawned** → it does the work, so it subscribes itself from its
  seed prompt (step 4) and this session **never subscribes at all** — a
  subscription here would wake a session whose turn is already over. If one was
  already in place before this turn, `unsubscribe_pr_activity` it (job 1 step 5).
  Say which session ends up holding it. If the spawned session reports the
  subscribe refused, that is the user's to unblock; do not subscribe here as a
  consolation, because this session is stopping either way.
- **Nothing was spawned** — already bound, no open PR, or not web/cloud → this
  session does the work. Subscribe here.

### 1. Find the branch's open PR

The same lookup as job 1 step 3, and it stands alone — run it even when you
skipped job 1. Oldest result wins. No open PR on the head → nothing to subscribe
to; skip to job 3 and say so.

### 2. Subscribe, unless this session opened the PR

```
subscribe_pr_activity(owner: <owner>, repo: <repo>, pullNumber: <n>)
```

A PR this session opened is subscribed by its own creation. A PR anyone **else**
opened needs this call explicitly — that is the whole gap this job closes.

Attempt it **once** (twice only if a transport error genuinely intervened). A
refusal —

```
Could not subscribe to this PR.
```

— is not flakiness, and repeating it is not diagnosis. The documented cause is
another watcher, a PR Steward holding a watching label; removing that label is
the user's call. Report the refusal and carry on to job 3.

### 3. Tell the user the panel is lying

The bottom panel binds to **session-created PRs only**, so on a taken-over PR it
keeps offering **"Create PR"** however well the subscription works. Once
subscribed that button is cosmetic — events arrive in the conversation instead —
and saying so unprompted stops the user reading it as "not connected".

**Never press it, and never call `create_pull_request` for a branch that already
has an open PR.** GitHub refuses a second open PR for the same head/base; against
a different base it succeeds and mints a duplicate — the one outcome someone
asking for a takeover explicitly did not want.

## Job 3 — putting commits on the PR (any environment)

Plain `git` and PR-lookup mechanics — no session tools — so this works from the
CLI, OpenHands, or Copilot as well as web/cloud, whoever owns the branch or PR.

### Never trust the local branch ref

A session handed someone else's branch may have a **local ref of that name that
is not the PR's head** — rebased or force-pushed by another session after this
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

- **Subject matching is worthless after a rebase** — new SHAs, often reworded
  subjects, so "these subjects are missing upstream" proves nothing.

When the remote is authoritative: `git reset --hard origin/<branch>`. If the local
branch genuinely carries work the PR head lacks, rebase it onto the PR head
instead — and say so before touching anything.

### Push, adding nothing

```bash
git push -u origin <branch>
```

- **No new branch. No second PR.** Someone asking you to take over a PR wants
  commits on *that* PR. This holds however loudly the panel offers "Create PR".
- Retry network failures with backoff (2s, 4s, 8s, 16s). Do not retry an auth or
  non-fast-forward rejection — diagnose it.
- **A merged PR cannot be reused.** Restart the branch from the current default
  branch and treat the follow-up as new work:
  `git fetch origin <default> && git checkout -B <branch> origin/<default>`,
  rebasing any unmerged commits onto the new base.

## Why the panel is not attached

A web session records two different things, and the panel follows the second:

```
sources:  refs/heads/alice/redesign-nav     ← what you checked out
outcomes: branches: ["claude/session-sof3wd"]  ← what the panel binds to
```

Starting a session *from* the PR's branch does not help: the outcome still
auto-mints a session-derived name, every commit lands on the PR's branch, and the
panel stays bound to a branch that never receives one.

The outcome is **fixed at session creation**. `set_session_title` and
`set_session_tags` are the only session mutators available and neither touches the
binding — which is why job 1 spawns a session rather than repairing this one.

An unbound panel has a second tell: on a branch whose PR this session did not
open, it shows a **"Create PR"** button where the CI and check status belongs.
That is the panel reporting *its own* binding, not the state of the branch.

## Racing

Two sessions registered to the same outcome branch can both push to it. When a
second one is bound, agree which pushes and say so — otherwise a force-push or a
non-fast-forward rejection arrives with no explanation.

## Checklist

- [ ] `get_session` called on turn one, before git archaeology, issue/PR listing,
      or asking the user what to work on
- [ ] Repo and branch read from the session, not guessed; "non-default" judged
      against the repo's real default branch, not an assumed `main`
- [ ] `outcomes[]` checked first — already bound means stop, not spawn
- [ ] Oldest open PR on the head chosen, others named
- [ ] Spawned session's `outcome_branch` verified equal to the branch
- [ ] Its only prompt is the `subscribe_pr_activity` seed; no branch, no PR created
- [ ] Panel rendering confirmed **by the user**, never assumed
- [ ] After the spawn, the report ends the turn: no CI, review, fix, build, or
      test work here
- [ ] `subscribe_pr_activity` held by the session doing the work, for any open PR
      it did not open — and never called here on the spawn path
- [ ] Any subscription predating this turn cleared with `unsubscribe_pr_activity`
      before the report, without asking first
- [ ] User told the panel's "Create PR" is cosmetic once subscribed — and that
      pressing it would open a duplicate
- [ ] For commits: divergence settled by dates and file sets before pushing

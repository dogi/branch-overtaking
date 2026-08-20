# overtaking — Claude Code plugin marketplace

A personal marketplace hosting `branch-overtaking`: adopt an existing branch and pull
request instead of opening new ones, and attach a web session's PR info panel to
it. Maintain the skill here once; opt any project into it — including **Claude
Code on the web / cloud** sessions.

The skill separates two problems that arrive together and get conflated:

| Problem | Symptom | Fixable from inside the session? | Where it applies |
|---|---|---|---|
| **Git takeover** | commits need to land on someone else's PR | yes — usually first try | anywhere: CLI, OpenHands, Copilot, web/cloud |
| **Stale local lineage** | branch of the right *name* is an older pre-rebase history | yes — verify, then reset | anywhere |
| **Session binding** | PR info panel stays empty however many commits land | no — the outcome branch is fixed at session creation | Claude Code web/cloud only |
| **Event delivery** | CI failures and review comments on the PR never arrive in the session | yes — subscribe to the PR explicitly | Claude Code web/cloud only |

The third one is why this skill exists. A session's panel follows its registered
**outcome branch**, not what it pushes, and the web UI mints a session-derived
outcome even when you start it *from* the PR's branch. `create_session`'s
`outcome_branch` argument is the only lever that re-points it — a construct
that only exists on the web/cloud product surface.

Event delivery is the one place authorship matters. A PR the session itself
opened already wakes it on CI failures and review comments; a PR a teammate
opened does not, and the panel makes it worse by offering a **"Create PR"**
button for a branch that already has one — pressing it would mint a duplicate.
The skill closes that gap with an explicit `subscribe_pr_activity` call during
takeover.

Otherwise none of this cares who authored the branch or PR being taken over — a
teammate's, a bot's, a fork's, all handled the same as one of your own.

## Structure

```
.claude-plugin/marketplace.json          # marketplace catalog
SKILL.md                                 # the skill itself
NOTES.md                                 # runs behind the rules — not loaded by the skill
plugins/branch-overtaking/
├── .claude-plugin/plugin.json           # plugin manifest
└── skills/overtaking/
    └── SKILL.md                         # skill definition (symlink to the root copy)
```

## Hosting

This marketplace is hosted at `dogi/branch-overtaking`. The
`.claude-plugin/marketplace.json` catalog lives at the repo root so Claude Code
can discover it when the repo is added as a marketplace.

## Use it in the terminal (CLI)

```
/plugin marketplace add dogi/branch-overtaking
/plugin install branch-overtaking@dogi
/reload-plugins
```

Then invoke `/branch-overtaking:overtaking` — or just ask to "take over this PR" / "push
onto that existing PR" / "why won't the PR panel attach", which the description
auto-triggers.

The CLI has no `get_session`/`create_session`/outcome branch, so only the
git-takeover half applies here — there is no panel to bind.

## Use it on Claude Code web / cloud

Cloud sessions can't see your local `~/.claude`, and user-scoped `enabledPlugins`
does **not** carry over. Declare the marketplace + plugin in the target repo's
`.claude/settings.json` (this file is part of the clone, so the cloud VM installs
the plugin at session start — needs network access to GitHub, which the default
allowlist covers):

```json
{
  "extraKnownMarketplaces": {
    "dogi": {
      "source": {
        "source": "github",
        "repo": "dogi/branch-overtaking"
      }
    }
  },
  "enabledPlugins": {
    "branch-overtaking@dogi": true
  }
}
```

Commit that to each repo where you want the skill available in web sessions.

## Use it from OpenHands

OpenHands auto-loads `.agents/skills/<name>/SKILL.md`. Add this repo as a
submodule in the target repo so the file is physically present:

```bash
git submodule add -b main https://github.com/dogi/branch-overtaking.git .agents/skills/branch-overtaking
```

Bump the pin after every merge here, or OpenHands keeps seeing the old revision
while Claude Code's marketplace fetch tracks this repo's `main` tip:

```bash
git submodule update --remote -- .agents/skills/branch-overtaking
```

The skill itself stays maintained here — bump `version` in `plugin.json` on each
release so installs pick up updates.

Same caveat as the CLI: OpenHands has no session/outcome-branch concept
either, so it only gets the git-takeover half of the skill.

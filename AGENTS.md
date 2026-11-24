# AGENTS.md: rules shared by every SotA Lore repo

Every repo's `CLAUDE.md` pulls this file in with `@../hq/AGENTS.md`, so it applies to every agent working
on SotA Lore, on either machine. It holds only what is true across repos: who owns what, which files are
contracts between repos, and how agents talk to each other. Everything about one repo stays in that repo.

## Repos and who owns them

All repos live in the `sotalore` GitHub org and are checked out side by side in `~/code/sotalore/` on
gaffer (Linux) and `C:\dev\sotalore\` on peregrin (Windows).

| Repo | What it is | Usual machine | Agent name |
|---|---|---|---|
| `sotalore/sotalore` | the Rails app at sotalore.com (Heroku) | gaffer | `app` |
| `sotalore/sladmin` | `sl-admin`, the in-game Lua plugin for site admins: recipe, item, finds and game-skill exports | peregrin | `sladmin` |
| `sotalore/slskills` | `sl-skills`, the in-game Lua plugin for players: their trained skills, as a code and a file | peregrin | `slskills` |
| `sotalore/companion` | a Windows desktop app (Tauri: Rust core, web UI) beside the game: uploads exports, checks skills, files screenshots and spoken notes by scene; roadmap in hq#3 | peregrin | `companion` |
| `sotalore/hq` | this repo: coordination only, no code | gaffer | `hq` |

- **An agent changes only its own repo.** If a fix belongs in another repo, open an issue there (see
  "Messages" below); don't edit its files, even when the checkout is right next to yours.
- The agent name is also the hc terminal name (`hc run --name sladmin ...`), so `gaffer/app` or
  `peregrin/sladmin` is always the address.
- The plugins run only on peregrin, where the game is. Their agents can't launch the game; Darren tests in
  game. What every plugin shares (naming, the game's Lua rules, no shared code) is in `LUA-PLUGINS.md`, which
  plugin repos pull in too.

## Contracts

A contract is a file that one repo writes and another reads. **Each contract lives in the repo that writes
it**, and only that repo's agent changes it. Readers say, in their own repo, which versions they support.

| Contract | Lives in | Written by | Read by | Version today |
|---|---|---|---|---|
| Admin export formats (`sotalore-recipes-*.json`, `-item-catalog.json`, `-scene-finds.json`, `-game-skills.json`, and the current-scene file the companion reads) | `sladmin/EXPORT-FORMAT.md` | sladmin | app, companion | `pluginVersion` only |
| Skills formats (the paged base-36 code a player pastes, and `sotalore-skills.json`) | `slskills/EXPORT-FORMAT.md` | slskills | app | `pluginVersion` only |
| The site API (`/api/v1/...`), including uploads of the admin exports | `sotalore/docs/api.md` | app | companion | `v1` in the path |

Example files for each export go in the app as spec fixtures (`spec/fixtures/plugin-examples/`), copied from
the plugin repo when a format changes.

### Versions and breaking changes

- Every export starts with the same header: `apiVersion` (the **game's** Lua API version, not the site
  API's), `plugin`, `pluginVersion` and `type`. None of these is a format version.
- **Proposed:** add `formatVersion` (an integer per `type`) to the header. A change readers can ignore (a new
  optional field) keeps it; a change that breaks a reader (a renamed, removed or retyped field, a changed
  meaning) bumps it. Until then, readers key on `pluginVersion`, as the app does today.
- A breaking change goes in this order: the writer opens an issue in each reader's repo, readers add support
  for the new version (keeping the old one while old files are still around), then the writer ships. The
  contract file's changelog names the version and links the issue.
- A contract file changes in the same PR as the code that changes the format, never separately.

## Who does what

- **The lead** runs a plan that touches more than one repo. By default that's the `hq` agent. It opens and
  keeps the tracking issue, starts or messages the repo agents, and reports to Darren. There is one lead per
  plan.
- **Repo agents** work in their own repo, on issues in that repo. They message other agents only along
  contract edges (below) and never start agents themselves; if a job needs another agent, they tell the lead.
- **Darren** keeps what only he can do: testing in game, merging PRs, and deciding what gets built. A merge to
  the app's `main` deploys it to Heroku through GitHub Actions, so merging is deploying. Ask him only for
  those; route everything else through issues and the lead.
- **One piece of work in progress per repo.** Finish a step, get Darren's test or review, and ship it
  (committed and pushed, or the PR merged) before starting the next step in that repo. Don't build a new step
  on top of uncommitted work. If something new comes up mid-step, open an issue for it and leave it for later.
  The lead starts a repo's next step only once the last one has shipped.

## Tracking work

- **Issues are the record.** Work and the conversation about it live in GitHub issues in the repo that does
  the work. A request to another repo is an issue in that repo, answered there.
- **A cross-repo plan gets a tracking issue in `sotalore/hq`**, labelled `tracking`, with sub-issues in the
  repos doing the steps (GitHub allows sub-issues across repos in one org). The lead keeps its checklist
  current; it's where Darren looks to see where things stand.
- **One org-level GitHub Project**, [SotA Lore](https://github.com/orgs/sotalore/projects/1), shows every repo's
  work. New issues in `sotalore/sotalore` are added automatically; in every other repo, add each new issue to it
  (`gh project item-add 1 --owner sotalore --url <issue url>`). Keep each item's Status current: **Todo**,
  **Doing** (in progress, or an active plan), **Waiting on Darren** (an in-game test, a merge or a decision),
  **Done**. Waiting on Darren is the list Darren works from, so move an item there when it
  stops on him, and back when he's answered.
- **Labels:** `bug`, `enhancement` and `tracking` in every repo. Which repo an issue is in says whose side it
  is, so the old `app` and `plugin` labels go away after the split.
- **PRs say `Closes <repo>#N`** for the issue they finish. When a step of a plan lands, tick it on the
  tracking issue and add a one-line comment linking the PR.
- `gh` is signed in as Darren, so end every issue and comment you write with
  `_Opened by Claude Code for @darrenboyd._` (or `_Comment by Claude Code for @darrenboyd._`).
- **Every repo is public**, and so are its issues: no credentials, tokens, or anything about users.

## Messages between agents

Agents talk with `hc send <machine>/<agent> "..."`. Messages are signals, not the record.

1. **Point at the record; don't carry it.** A message names an issue, PR or contract file and says what
   happened to it: `sotalore/sladmin#12 is open: the finds file needs creature levels`. The detail goes in
   the issue, where Darren and later agents can find it. If you're about to write more than two lines in a
   message, write it in the issue instead.
2. **Follow the contract edges.** Agents message only the agents they share a contract with:

   | From | To | About |
   |---|---|---|
   | sladmin | app, companion | admin export formats |
   | slskills | app | skills formats |
   | app | companion | the API |
   | companion | app | the API, upload problems |
   | companion | sladmin | export files, the current-scene file |
   | readers | writers | requests, bugs in a contract |
   | anyone | hq | blocked, needs another agent, needs Darren |
   | hq | anyone | the lead starting or steering work |

   Anything else goes through the lead.
3. **One lead per cross-repo plan.** Only the lead starts agents and decides the order of steps. If two
   agents disagree about a contract, the writer decides, and the lead settles it if they can't agree.
4. **Say what reply you expect, and from whom.** End every message with one of: `Reply: none`,
   `Reply: <agent> when <condition>` (for example `Reply: sladmin when the PR is open`), or
   `Reply: Darren` (for his part only). Don't send acknowledgements, thanks or "on it"; silence means received.
5. **Darren's part goes to Darren, not round in circles.** When a step needs in-game testing, a merge or a
   decision, say so in the issue, tell the lead, and stop on that step. The lead collects these and asks Darren
   once.

Example:

```sh
hc send gaffer/app "sotalore/sladmin#12 merged: finds format 3 adds creature levels (EXPORT-FORMAT.md, 'Finds file'). Reply: app when the importer reads format 3"
```

### Starting agents (lead only)

```sh
hc run --bg --outlive-parent --name sladmin --on peregrin --cwd 'C:\dev\sotalore\sladmin' claude "<task, naming the issue>. Work only in this repo; start no agents. Reply with hc send gaffer/hq when done or blocked."
```

- Put the task in the `run` command; a send to an agent that's still starting can be lost.
- `--outlive-parent` keeps the agent running if the hq session ends. Without it, the agent stops with the lead.
- Before starting an agent, check `hc ls` for one already running under that name, and message it instead.

## The two machines

- **gaffer** (Linux): Rails work and the `hq` agent. Repos in `~/code/sotalore/`.
- **peregrin** (Windows, where the game runs): the plugin agents. Repos in `C:\dev\sotalore\`; ssh as `darre`
  (`Host peregrin`). Each game client's `Lua\sl-admin` and `Lua\sl-skills` folders are junctions into the
  plugin checkouts, so the game always runs what's checked out.
- Messages from peregrin to gaffer travel over gaffer's connection to peregrin, so they arrive only while
  gaffer's hc core is running.

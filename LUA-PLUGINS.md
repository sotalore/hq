# LUA-PLUGINS.md: rules every SotA Lore game plugin follows

Pulled into each plugin repo's `CLAUDE.md` with `@../hq/LUA-PLUGINS.md`, after `@../hq/AGENTS.md`. It holds
what is true for every plugin: how they're named, how the game runs them, and the API facts any plugin can
trip over. Facts about calls only one plugin uses (recipes, tooltips, gathering, targets, skills) stay in that
plugin's repo. Game API docs: https://catnipgames.net/lua/reference.html (download the page and search the
text; a summarising fetch loses detail, and the docs and the live API can differ).

## No shared code

Plugins share no code, and duplication is the rule. Each has its own small `core.lua` with the helpers it
needs (JSON writing, file writing, the file header), and the top of each says which other plugin has a twin of
those helpers, so a fix in one is checked in the other. Why:

- The game can't share code between packages: folders are flat, and `require` and `dofile` aren't allowed.
- There's no build step. Each game client's `Lua\<slug>` folder is a junction into the checkout, so the game
  runs exactly what's checked out.
- The copies are meant to differ: `sl-skills` goes to players and needs only API 14, `sl-admin` needs API 51.
  A shared helper could quietly use a call that older clients lack.
- The store accepts only a package's listed files, so a shared file would be copied into each package anyway.
- A player who has only `sl-skills` must never need anything from `sl-admin`.

## Naming

Every plugin is named the same way, so players can guess the command from the plugin's name. For a plugin
about `<name>`:

| What | Rule | sl-admin | sl-skills |
|------|------|----------|-----------|
| Repo | `sotalore/sl<name>` | `sladmin` | `slskills` |
| Folder and slug | `sl-<name>` | `sl-admin` | `sl-skills` |
| Display name (manifest `name`, `ScriptName`) | `SotaLore <Name>` | SotaLore Admin | SotaLore Skills |
| Slash command | `/sl<name>` (lower case, no hyphen; the game allows `[a-z][a-z0-9]{2,23}`) | `/sladmin` | `/slskills` |
| Key binding label | `SotaLore <Name>: ...` | SotaLore Admin: toggle toolbar | SotaLore Skills: toggle window |
| Window titles | start `SotaLore` | SotaLore Finds, ... | SotaLore Skills |
| Output files | `sotalore-<kind>.json`, the kind named after the site's navigation | `sotalore-scene-finds.json` | `sotalore-skills.json` |
| Global table | `SotaLore<Name>`, the only global besides its `ShroudOn...` callbacks | `SotaLoreAdmin` | `SotaLoreSkills` |

- **The global name matters most.** All add-ons share one global environment, so two plugins with the same
  global break each other. Everything else is `local`; files share things by hanging them on the global.
- Chat lines need no prefix: the game adds `[Add-on: <display name>]`.
- `/sotalore` is refused ("sota" is a reserved prefix), which is why the prefix is `sl`.
- Every file a plugin writes starts with the header in `AGENTS.md` ("Versions and breaking changes").

## How the game runs a plugin

- A package is one flat folder, `Lua/<slug>/`, with its own `manifest.json` (metadata, `min_api_version`, and
  the load order of its files). A new file must be added to the manifest. The store accepts only the listed
  files plus `icon.png`, `README.md` and a few images, sounds or data files; anything else fails the upload.
- Files run in manifest order and share one environment, so a file may use only modules from files listed
  before it.
- **Don't call the game's API at the top level of a file**: the `Shroud...` functions don't exist while files
  load. `local UI = Shroud.UI` is the one exception. Only `main.lua` defines `ShroudOn...` callbacks, and each
  just calls into a module.
- **A callback that runs over 1 second is stopped** (5 seconds in `ShroudOnStart`). Spread long work across
  frames, cut by time as well as count. A call that can itself take over a second belongs in `ShroudOnStart` or
  nowhere. Never retry a call the game may have stopped: a burst of errors makes the game switch the plugin off.
- **A stopped callback must leave the job consistent.** Keep multi-frame jobs idempotent (store each record
  under its id, advance one at a time), so a frame that runs again changes nothing.
- An add-on may own at most 8 windows. `Shroud.UI` windows start closed, and `Show()` is refused for 3 seconds
  after the player closes one. That protects players from misbehaving add-ons: leave it alone, no retry loops
  or fresh window ids to dodge it. Change callbacks can fire before `ShroudOnStart` has built the windows, so
  guard them.
- Lua can't open folders, run slash commands or reach the clipboard. `/lua path`, typed by the player, opens
  the output folder.

## Files and saved data

- Output files go to the client's `Lua/` folder, one level above the plugin folders.
- **A file can't be opened for appending** (`io.open(path, "a")` fails). Keep a growing file in memory and
  write it whole with `"w"`.
- **Saved variables** are per slug and per client (`Lua\SavedVariables\<slug>.account.json`, and per
  character). Renaming a package starts it with empty saved data. The game writes them at logout, at unload
  (`/lua reload`, every scene change) and on `ShroudFlushSavedVars`, and **skips a scope whose JSON is over
  256 KB**, with only a line in the add-on console. Keep an eye on anything that grows with play.
- **Add-on data files** (API 51) hold what's too big for saved variables: `Lua/AddonData/<slug>/`, 1 MB of JSON
  per file, 16 files, written whole. One write per file every 10 seconds, and 4 per add-on every 10 seconds.

## API versions

- `ShroudLuaApiVersion` moves (it was 40, it's 57 on 2026-11-07). Re-read the docs when it does.
- When a plugin works around an API bug, gate the workaround on the API version, so it stops when the game
  fixes the bug (bag item `weight` and `value` were stack totals up to API 40, one item's from API 41).
- Each game client has its own `Lua` folder: `%APPDATA%\Catnip Games\Shroud of the Avatar\Lua` (regular) and
  `...\Shroud of the Avatar(DEV)\Lua` (QA), each with its own saved data and output files. Check which client
  Darren is playing before reading his files.

## Testing

- The game can't be launched by an agent. Syntax-check every file after an edit with `luac -p file.lua` (Lua
  5.5 on peregrin; the game's interpreter is older, 5.2-style), and check for stray globals with
  `luac -o NUL -l -l file.lua`.
- For behaviour, write a throwaway fake-game script (stub `Shroud.UI`, the getters and saved variables, call
  the `ShroudOn...` callbacks) and compare output files with the previous version's. Say plainly what is
  untested in game; Darren tests in game (`/lua reload`, `/lua check <slug>`).
- Bump a package's version in its `core.lua` and `manifest.json` together, and note it in the repo's
  `EXPORT-FORMAT.md` when a format changed.

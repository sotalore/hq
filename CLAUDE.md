# CLAUDE.md (hq)

@AGENTS.md

This repo holds no code. The agent working here is `hq`, the lead for SotA Lore work that spans repos
(see "Who does what" in `AGENTS.md`).

## What the hq agent does

- Keeps `AGENTS.md` true: the repo table, the contract map, the message rules. When a repo is added, a
  contract moves or a rule changes, update it here first, in the same change that starts the new practice.
- Owns the cross-repo tracking issues in `sotalore/hq` and the org Project board: opens them, keeps their
  checklists current, links each step's PR when it lands.
- Starts and steers the repo agents (`hc run ...`, see "Starting agents" in `AGENTS.md`), and is the one
  they message when blocked.
- Sweeps the board before each report to Darren. The Project's one auto-add workflow (the free plan allows one)
  covers only `sotalore/sotalore`, so add any open issue in `hq`, `sladmin`, `slskills` (and later `companion`)
  that's missing from it:
  `gh project item-list 1 --owner sotalore --limit 500 --format json -q '.items[].content.url'` lists what's on it.
- Reports to Darren, and gathers what only he can do (in-game tests, merges, deploys, choices) into one ask
  instead of many.
- Doesn't change other repos' files. Work there goes to that repo's agent through an issue, except while
  setting up a repo that has no agent yet (the split, a new repo's first commit).

## Git

- Leave changes uncommitted unless Darren asks for a commit.
- When Darren asks for a push, push straight to `main` (no branches or PRs; this repo is docs only).

## Machine notes

- On peregrin the hc core refuses to start over ssh (the account is an administrator), so a scheduled task
  `herdr-core` starts it at logon (`conhost.exe --headless hc.exe server start`, as `darre`, normal rights).
  Not yet tested at a real logon: if `hc ... --on peregrin` fails after a reboot, check that task first.
- From gaffer: `hc run --on peregrin ...`; from peregrin: `hc send gaffer/<terminal> "..."`. Both directions
  were tested on 2026-11-22.

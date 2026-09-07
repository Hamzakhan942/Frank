# watcher

A headless Claude Code loop that turns a GitHub Projects board into merged pull requests.
One throwaway box, one systemd unit per project, no permission prompts.

**How it works.** `bin/spawn.sh` polls the board's **In progress** column once a minute. When
a card is claimable it creates a git worktree for it and runs one tick:
`claude -p "/watch <n>" --dangerously-skip-permissions --max-turns 400 --max-budget-usd 15`.
The tick claims the card, does the work in the worktree, opens a PR, waits for CI, runs
one review pass, merges, and moves the card to **Done**; anything it cannot decide goes to
**Stuck - Needs help** with the options written on the card. Two failed attempts park the
card. The spawner is also the stale-claim sweeper.

**Why a box.** Skipping permissions is only safe where there is nothing to protect. The box
holds a bot GitHub login, a Claude subscription token, and a checkout. If a tick misbehaves,
rebuild the box. On a developer's own machine the same loop needs allow lists, trust
probes and a classifier, and the day that was tried it cost more than the feature work.

## Layout

    bin/spawn.sh          the loop (systemd runs it); --dry-run / --once
    bin/board.py          the board CLI: list/show/move/unclaim (BOARD_OWNER, BOARD_NUMBER)
    bin/worktree.sh       one worktree per card; copies a project's private inputs in
    bin/test-spawn.sh     85 cases, no network, every external command stubbed
    commands/watch.md     the /watch contract, installed for the watcher user (all projects)
    agents/reviewer.md    the one review pass: blocking findings only, one round
    projects/<name>.env   PROJECT_NAME, PROJECT_ROOT, BOARD_OWNER, BOARD_NUMBER
    deploy/watcher@.service, deploy/install.sh

A project keeps its own `CLAUDE.md`, `.claude/agents/` and CI; the watcher brings the loop.
A project-level `.claude/commands/watch.md` overrides the generic one.

## Set up a box (Debian 12 LXC, unprivileged, 4 cores / 8 GB)

    apt install git python3 curl gh                # plus node 22 and whatever the project builds with
    npm i -g @anthropic-ai/claude-code
    git clone https://github.com/roloc/watcher /opt/watcher   # or rsync it in
    /opt/watcher/deploy/install.sh                 # user, unit, /etc/watcher/env, symlinks, `board`

Then by hand, once:

1. **Claude login:** on a machine with a browser run `claude setup-token`, and put the printed
   value in `/etc/watcher/env` as `CLAUDE_CODE_OAUTH_TOKEN=…`. That file must not set
   `ANTHROPIC_API_KEY` — the key outranks the login and would bill the API instead.
2. **GitHub:** `su - watcher`, `gh auth login` as the bot account, `gh auth setup-git`. The bot
   needs write access to the repo and access to the Projects board.
3. **Project:** `deploy/install.sh riffmap` clones it under `/home/watcher/projects/`; drop its
   `.env.local` and any private inputs in place.
4. **Start:** `systemctl enable --now watcher@riffmap`; `journalctl -u watcher@riffmap -f`.
   Pause with `touch ~watcher/.local/state/watcher/riffmap/PAUSE`.

## Add a project

Add `projects/<name>.env`, run `deploy/install.sh <name>`, `systemctl enable --now
watcher@<name>`. The project needs a board with the columns
`Backlog | Ready | In progress | In review | Done | Stuck - Needs help`, an `in-flight`
label, and CI that is a required check on `main`.

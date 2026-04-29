## Gascity gastown — Project Spec

A hands-on project for learning Gascity by standing up a working Gastown city,
then using it as the foundation for a portfolio management app.

Source docs: https://github.com/gastownhall/gascity/blob/main/docs/getting-started/quickstart.md

---

## Purpose

Learn Gascity by configuring a local city with the Gastown pack. Once the city
is running, it becomes the substrate for a portfolio management app built from
a separate spec.

---

## Phase 1 — Install Gascity

```bash
brew install gastownhall/gascity/gascity
gc version
```

Required runtime tools (Homebrew installs all automatically):
`tmux`, `jq`, `git`, `dolt`, `bd` (beads CLI), `flock`

Fix the Oh My Zsh `gc` alias conflict if present:

```zsh
# ~/.zshrc — add after Oh My Zsh loads
unalias gc 2>/dev/null
```

---

## Phase 2 — Create the City

```bash
gc init ~/projects/gascity/gastown
cd ~/projects/gascity/gastown
```

`gc init` bootstraps the city directory, registers it with the supervisor, and
starts the controller.

---

## Phase 3 — City Structure

The gastown city uses three composable packs:

```
gastown/               ← city root (gc init target)
├── city.toml          ← deployment config: rigs, agent overrides, crew
├── pack.toml          ← city pack: imports, city-specific agents
└── assets/
    └── gastown/       ← vendored gastown pack (copied from gascity examples)
        ├── pack.toml
        ├── agents/
        │   ├── mayor/
        │   ├── deacon/
        │   ├── boot/
        │   ├── witness/
        │   ├── refinery/
        │   └── polecat/
        ├── formulas/
        ├── orders/
        └── assets/scripts/
```

---

## Phase 4 — city.toml

The main deployment config. Imports the gastown pack, declares rigs, and
optionally adds crew agents.

```toml
[workspace]
name = "gastown"
provider = "claude"
global_fragments = ["command-glossary", "operational-awareness"]

[imports.gastown]
source = "./assets/gastown"

[daemon]
patrol_interval = "30s"
max_restarts = 5
restart_window = "1h"
shutdown_timeout = "5s"
formula_v2 = true

# Register a project rig — activates per-rig agents (witness, refinery, polecat)
# [[rigs]]
# name = "portfolio"
# path = "/path/to/portfolio-app"

# Crew agents: individually named, persistent, each gets an isolated worktree
# [[agent]]
# name = "wolf"
# dir = "portfolio"
# prompt_template = "assets/gastown/assets/prompts/crew.template.md"
# nudge = "Check your hook and mail, then act accordingly."
# idle_timeout = "4h"
```

---

## Phase 5 — pack.toml (city root pack)

Thin city pack that imports gastown:

```toml
[pack]
name = "my-gastown"
schema = 2

[imports.gastown]
source = "./assets/gastown"
```

---

## Phase 6 — Gastown Pack Agents

All agents are defined in `assets/gastown/agents/<role>/agent.toml`. Copy the
agent definitions from the gascity examples repo:

```bash
# Vendor the gastown pack from the gascity examples
git clone --depth=1 https://github.com/gastownhall/gascity.git /tmp/gascity
cp -r /tmp/gascity/examples/gastown/packs/gastown ./assets/gastown
cp -r /tmp/gascity/examples/gastown/packs/maintenance ./assets/maintenance
```

### City-scoped agents (one per city, always running)

| Agent   | Scope | Mode      | Purpose |
|---------|-------|-----------|---------|
| mayor   | city  | always    | Coordinator — reads mail, dispatches work |
| deacon  | city  | always    | Patrol — health checks, heartbeat cycle |
| boot    | city  | always    | Watchdog — session lifecycle |

### Rig-scoped agents (stamped per registered rig)

| Agent    | Scope | Mode      | Purpose |
|----------|-------|-----------|---------|
| witness  | rig   | always    | Monitors worker status, patrols rig health |
| refinery | rig   | on_demand | Merge queue — rebase, test, merge |
| polecat  | rig   | on_demand | Worker pool — takes tasks, creates branches, submits to refinery |

---

## Phase 7 — Register a Rig

Once the city is running:

```bash
mkdir ~/projects/portfolio && cd ~/projects/portfolio && git init
gc rig add ~/projects/portfolio --name portfolio
```

This activates the rig-scoped agents (witness, refinery, polecat) for that
project.

---

## Phase 8 — Verify

```bash
gc start                 # start/restart city
gc status                # confirm agents are running
gc doctor                # system health check
```

Send a test task to the polecat pool:

```bash
gc sling claude "Hello from gastown" --rig portfolio
```

Watch the bead:

```bash
bd show <bead-id> --watch
```

---

## Phase 9 — Common Overrides

### Add more polecats for a rig

```toml
# city.toml
[[rigs]]
name = "portfolio"

[rigs.imports.gastown]
source = "./assets/gastown"

[[rigs.patches]]
agent = "gastown.polecat"

[rigs.patches.pool]
max = 10
```

### Change provider for a rig's polecats

```toml
[[rigs.patches]]
agent = "gastown.polecat"
provider = "codex"
```

### Patch a city agent (mayor timeout)

```toml
[[patches.agent]]
name = "gastown.mayor"
idle_timeout = "2h"
```

---

## Phase 10 — Next Step: Portfolio Management App

Once the gastown city is running and `gc doctor` passes, a separate spec
(`PORTFOLIO_SPEC.md`) will describe the portfolio management app. The app will
use `gc sling` to route tasks to agents and `gc formula cook` to orchestrate
multi-step workflows against the registered portfolio rig.

---

## Key Concepts Reference

| Concept | Gas City primitive |
|---------|-------------------|
| Role (mayor, witness…) | Named agent in a pack |
| Plugin | Exec order or formula order |
| Convoy | Bead-backed grouping via `gc convoy` |
| Work item | Bead (`bd` is the CRUD tool) |
| Watchdog / dog | Exec order, or scalable session config |
| Directory identity | Explicit `dir` / `work_dir` in config |

---

# Gastown City — Execution Plan

Based on SPEC.md. Each step is a concrete action with a clear done condition.

---

## Status

| Step | Description | Status |
|------|-------------|--------|
| 1 | Check / install Gascity | ✅ done (`gc` v1.0.0, `gc doctor` pending) |
| 2 | Vendor packs | ✅ done |
| 3 | Write `pack.toml` | ✅ done |
| 4 | Write `city.toml` | ✅ done |
| 5 | Init the city | ✅ done |
| 6 | Start and verify | ✅ done |
| 7 | Register portfolio rig | ✅ done |
| 8 | Smoke-test sling | ✅ done |

---

## Step 1 — Install Gascity ✅

Check if already installed and all runtime deps are present:

```bash
command gc version                        # should print a version number
gc doctor                                 # checks tmux, jq, git, dolt, bd, flock
```

If `gc` is missing:
```bash
brew install gastownhall/gascity/gascity
```

If `gc doctor` reports missing tools:
```bash
brew install tmux jq dolt flock beads
```

If Oh My Zsh intercepts `gc` as `git commit`:
```zsh
# ~/.zshrc — add after Oh My Zsh loads
unalias gc 2>/dev/null
```

Done when: `command gc version` prints a version and `gc doctor` exits 0.

Current state: `gc` v1.0.0 present.

---

## Step 2 — Vendor the packs

Clone gascity examples and copy the two required packs into `assets/`:

```bash
git clone --depth=1 https://github.com/gastownhall/gascity.git /tmp/gascity
mkdir -p assets
cp -r /tmp/gascity/examples/gastown/packs/gastown  assets/gastown
cp -r /tmp/gascity/examples/gastown/packs/maintenance assets/maintenance
```

Done when: `assets/gastown/pack.toml` and `assets/maintenance/pack.toml` exist.

---

## Step 3 — Write `pack.toml`

Create the thin city root pack at `pack.toml`:

```toml
[pack]
name = "my-gastown"
schema = 2

[imports.gastown]
source = "./assets/gastown"
```

Done when: `pack.toml` exists in the city root.

---

## Step 4 — Write `city.toml`

Create the deployment config at `city.toml`:

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
```

Done when: `city.toml` exists and `gc config` parses it without errors.

---

## Step 5 — Init the city

```bash
gc init ~/projects/gascity/gastown
```

If the city directory already exists, `gc init` is idempotent — it registers
the city with the supervisor if not already registered.

Done when: `gc status` shows the gastown city (may show agents as stopped before start).

---

## Step 6 — Start and verify

```bash
gc start
gc status          # city-scoped agents (mayor, deacon, boot) should be running
gc doctor          # all checks green
```

Expected: mayor, deacon, boot sessions active. No doctor failures.

If `gc doctor` reports missing tools, install them:
```bash
brew install tmux jq dolt flock beads
```

Done when: `gc doctor` exits 0.

---

## Step 7 — Register portfolio rig

Create and register a project directory as a rig to activate rig-scoped agents:

```bash
mkdir -p ~/projects/portfolio
cd ~/projects/portfolio && git init
cd ~/projects/gascity/gastown
gc rig add ~/projects/portfolio --name portfolio
gc status          # witness should now appear; refinery/polecat on-demand
```

Add the rig to `city.toml` to make registration durable across restarts:

```toml
[[rigs]]
name = "portfolio"
path = "/Users/steve/projects/portfolio"

[rigs.imports.gastown]
source = "./assets/gastown"
```

Done when: `gc status` shows `witness/portfolio` as running.

---

## Step 8 — Smoke-test sling

Send a minimal task through the polecat pool:

```bash
gc sling claude "Print hello world to stdout" --rig portfolio
```

Then watch the bead:

```bash
bd show <bead-id> --watch
```

Done when: bead moves through `open → assigned → closed` without error.

---

## Next

Once Step 8 passes, the gastown city is working. The next phase is
`PORTFOLIO_SPEC.md` — defining the portfolio management app that will be
built on top of this city using `gc sling` and `gc formula cook`.

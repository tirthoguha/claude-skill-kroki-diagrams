# claude-skill-kroki-diagrams

A [Claude Code](https://claude.com/claude-code) skill that renders Mermaid (and
other [Kroki](https://kroki.io)-supported) diagrams to **flattened, white-background
PNGs** suitable for Confluence / Slack / docs, then reviews each render visually
for readability before declaring it done.

The skill drives a **local** Kroki stack: the main `kroki` engine plus its
`kroki-mermaid` companion, wired together on a shared Docker network.

## Architecture

```
Claude Code ──curl──▶ kroki  (localhost:8585)
                        │  delegates Mermaid rendering
                        ▼  via KROKI_MERMAID_HOST over kroki-net
                     kroki-mermaid  (internal, :8002)
```

Only `kroki` publishes a port. It defaults to **8585** (port 8000 is commonly
busy) and is overridable via `KROKI_PORT`. The companion is reachable only on the
internal `kroki-net` bridge network, by DNS hostname `kroki-mermaid`.

## Prerequisites

- **Docker** with Compose v2 (Docker Desktop on macOS/Windows, or Docker Engine
  + `docker compose` plugin on Linux)
- **`curl`** (preinstalled on macOS/Linux; on Windows use Git Bash, WSL, or the
  bundled `curl.exe`)
- **An image flattener** for the alpha-strip step — the skill auto-detects, in
  order of preference:
  | Tool | Platforms | Install |
  |---|---|---|
  | **ImageMagick** (`magick`/`convert`) | macOS, Linux, Windows | `brew install imagemagick` · `apt install imagemagick` · `choco install imagemagick` |
  | **`sips`** | macOS only | built in |
  | **Pillow** (`python3` + PIL) | any | `pip install pillow` |

  On Linux/Windows there is no `sips`, so install **ImageMagick** (recommended) or
  Pillow.

## Install

**Step 1 — start the Kroki stack** (both containers, shared network):

```bash
git clone https://github.com/tirthoguha/claude-skill-kroki-diagrams.git
cd claude-skill-kroki-diagrams
docker compose up -d                          # or: KROKI_PORT=9123 docker compose up -d
curl -sf http://localhost:8585/ >/dev/null && echo "Kroki OK"
```

**Step 2 — make the skill available to Claude Code.** Two options, below.

Then in Claude Code, ask for any diagram ("draw the architecture", "sequence
diagram of the auth flow", …) and the skill takes over.

## Installing the skill into Claude Code

A Claude Code skill is just a directory containing a `SKILL.md`. Claude discovers
it from one of these locations — pick whichever fits:

### Option A — manual (clone + symlink or copy)

The directory name becomes the command name (`/kroki-diagrams`). Symlink so the
skill tracks `git pull`s:

```bash
# Personal (all your projects):
ln -s "$PWD/skills/kroki-diagrams" ~/.claude/skills/kroki-diagrams
# …or project-scoped (this repo only, commit it for teammates):
ln -s "$PWD/skills/kroki-diagrams" .claude/skills/kroki-diagrams
```

> **Windows:** `mklink /D` (cmd, admin) or `New-Item -ItemType SymbolicLink`
> (PowerShell), or just copy the folder: `cp -r skills/kroki-diagrams ~/.claude/skills/`.

### Option B — as a plugin via the bundled marketplace

This repo doubles as a one-plugin marketplace (`.claude-plugin/marketplace.json`),
so others can install without cloning:

```text
/plugin marketplace add tirthoguha/claude-skill-kroki-diagrams
/plugin install kroki@kroki-diagrams-skill
```

Installed this way the skill is namespaced: invoke it as `/kroki:kroki-diagrams`.
Plugin installs update via `/plugin marketplace update`.

> Either way you still need the Kroki stack from **Step 1** running locally — the
> plugin ships the skill, not the Docker containers.

## Configuring the port

The stack and the skill both read `KROKI_PORT` (default `8585`). Export it once
in your shell so `docker compose` and Claude Code agree:

```bash
export KROKI_PORT=9123   # add to ~/.zshrc / ~/.bashrc to persist
```

## Configuring the output directory

By default the skill writes `.mmd` sources and rendered `.png`s to
`~/Documents/kroki-diagrams/<topic-slug>/`. Override the base directory by
exporting `KROKI_DIAGRAMS_DIR` (the skill reads it via the shell):

```bash
export KROKI_DIAGRAMS_DIR="$HOME/diagrams"   # add to ~/.zshrc / ~/.bashrc to persist
```

You can also override per request by telling Claude explicitly ("put these in
`./docs/diagrams`") — an explicit instruction wins over both the env var and the
default.

## Why a compose file instead of two `docker run`s

Kroki's Mermaid support lives in a separate `kroki-mermaid` container that the
main engine calls over the network. Running that by hand means two ordered
`docker run`s plus a manually created network. The compose file encodes the whole
topology declaratively so it comes up with one command:

- both containers on a shared `kroki-net` bridge network,
- `kroki` finds the companion by DNS via `KROKI_MERMAID_HOST=kroki-mermaid`,
- `restart: unless-stopped` so the stack survives reboots,
- the host port is parameterised (`KROKI_PORT`, default `8585`).

## What this repo deliberately does NOT contain

Rendered diagram outputs (the `.mmd` + `.png` files the skill produces) are
**work artifacts**, not source — so `.gitignore` keeps them out, and the skill
writes them outside any repo by default (`KROKI_DIAGRAMS_DIR`, default
`~/Documents/kroki-diagrams/<topic>/`). They belong in your docs/Confluence, not
in version control.

## How this repo is packaged (skill distribution formats)

A Claude Code skill is just a `SKILL.md` in a directory. This repo ships it in
the two formats Claude Code recognises, so consumers can pick either install
path above:

```
claude-skill-kroki-diagrams/
├── .claude-plugin/
│   ├── plugin.json         # marks the repo root as a plugin named "kroki"
│   └── marketplace.json    # lists that plugin → enables /plugin marketplace add
├── skills/
│   └── kroki-diagrams/
│       └── SKILL.md        # the skill itself (Option A symlinks this dir)
├── docker-compose.yml      # the local Kroki stack the skill drives
├── LICENSE
└── README.md
```

- **As a bare skill** — Claude loads any `<name>/SKILL.md` under `~/.claude/skills/`
  (personal) or `.claude/skills/` (project). The directory name is the command
  name. That's Option A.
- **As a plugin** — `.claude-plugin/plugin.json` (`name`, `description`,
  `version`, `author`) marks the repo as a plugin; its `skills/` dir is
  auto-discovered. Skills are namespaced `/<plugin>:<skill>`.
- **As a marketplace** — `.claude-plugin/marketplace.json` lists the plugin with
  `source: "."` (the repo itself), so one repo is both the plugin source and its
  own catalogue. That's what makes `/plugin marketplace add <owner/repo>` work
  (Option B).

To fork this into your own skill, swap the `SKILL.md`, rename the `skills/<dir>`,
and update the `name`/`description` in both `.claude-plugin/*.json` files.

## License / attribution

This repo (the skill, compose stack, and tooling) is MIT-licensed — see
[LICENSE](LICENSE). `yuzutech/kroki` and `yuzutech/kroki-mermaid` are third-party
images ([kroki.io](https://kroki.io), also MIT) and are not redistributed here —
the compose file just pulls them.

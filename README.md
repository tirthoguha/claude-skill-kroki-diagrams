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

**Step 2 — make the skill available to Claude Code.** A skill is just a directory
containing a `SKILL.md`; Claude discovers it under `~/.claude/skills/` (personal)
or `.claude/skills/` (this project only). The directory name becomes the command,
so installing it as `kroki` gives you **`/kroki`**. Symlink it so it tracks
`git pull`s:

```bash
# Personal (all your projects):
ln -s "$PWD/skills/kroki" ~/.claude/skills/kroki
# …or project-scoped (commit .claude/skills/kroki for teammates):
ln -s "$PWD/skills/kroki" .claude/skills/kroki
```

> **Windows:** use `mklink /D` (cmd, admin) or `New-Item -ItemType SymbolicLink`
> (PowerShell), or just copy the folder: `cp -r skills/kroki ~/.claude/skills/`.

Then in Claude Code ask for any diagram ("draw the architecture", "sequence
diagram of the auth flow", …) — or type `/kroki` — and the skill takes over. It
still needs the Kroki stack from **Step 1** running locally.

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

## How this repo is packaged

```
claude-skill-kroki-diagrams/
├── skills/
│   └── kroki/
│       └── SKILL.md        # the skill — symlink this dir into ~/.claude/skills/
├── docker-compose.yml      # the local Kroki stack the skill drives
├── LICENSE
└── README.md
```

It ships as a **bare skill**: Claude Code loads any `<name>/SKILL.md` placed under
`~/.claude/skills/` (personal) or `.claude/skills/` (project), and the directory
name is the command. Installed as `kroki`, that's `/kroki` — no namespace prefix.

> **Want it as a plugin instead?** Claude Code can also distribute skills as
> plugins via a marketplace (`/plugin marketplace add <owner/repo>` →
> `/plugin install …`). That route namespaces the command as `/<plugin>:<skill>`,
> which is why this repo skips it — a bare skill keeps the command a clean
> `/kroki`. To wrap it as a plugin anyway, add a `.claude-plugin/plugin.json`
> (`name`, `description`, `version`, `author`) at the repo root plus a
> `.claude-plugin/marketplace.json` listing it with `source: "."`.

## License / attribution

This repo (the skill, compose stack, and tooling) is MIT-licensed — see
[LICENSE](LICENSE). `yuzutech/kroki` and `yuzutech/kroki-mermaid` are third-party
images ([kroki.io](https://kroki.io), also MIT) and are not redistributed here —
the compose file just pulls them.

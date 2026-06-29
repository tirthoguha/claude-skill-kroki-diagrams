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

- **Docker** (Docker Desktop on macOS/Windows, or Docker Engine on Linux)
- **macOS `sips`** (built in) for the alpha-flatten step — or **ImageMagick**
  (`magick` / `convert`) as a cross-platform fallback
- `curl` (built in on macOS/Linux)

## Install

```bash
git clone <this-repo> claude-skill-kroki-diagrams
cd claude-skill-kroki-diagrams

# 1. Start the Kroki stack (both containers, shared network)
docker compose up -d                          # or: KROKI_PORT=9123 docker compose up -d
curl -sf http://localhost:8585/ >/dev/null && echo "Kroki OK"

# 2. Make the skill available to Claude Code (symlink, so updates track git)
ln -s "$PWD/skills/kroki-diagrams" ~/.claude/skills/kroki-diagrams
```

Then in Claude Code, ask for any diagram ("draw the architecture", "sequence
diagram of the auth flow", …) and the skill takes over.

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

The original setup used two manual commands. This compose file encodes the same
topology declaratively:

- both containers on a shared `kroki-net` bridge network,
- `kroki` finding the companion via `KROKI_MERMAID_HOST=kroki-mermaid` (DNS),
- `restart: unless-stopped` so the stack survives reboots,
- one `docker compose up -d` instead of two ordered commands.

## What this repo deliberately does NOT contain

Rendered diagram outputs (the `.mmd` + `.png` files the skill produces) are
**work artifacts** and stay out of version control. The skill writes them to
`~/Documents/kroki-diagrams/<topic>/` by default — keep them there (or in your
private doc system), not in this repo.

## License / attribution

`yuzutech/kroki` and `yuzutech/kroki-mermaid` are third-party images
([kroki.io](https://kroki.io), MIT). This repo only adds the skill definition
and the compose/install tooling around them.

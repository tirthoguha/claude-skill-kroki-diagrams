---
name: kroki
description: Generate architecture / sequence / state / flow diagrams as PNG files using the user's local Kroki Docker container, with white backgrounds suitable for Confluence and dark-mode-tolerant docs. Use this skill whenever the user asks for diagrams to visualize a system, flow, lifecycle, sequence, state machine, or architecture — and follow up with a vision-based review of each rendered diagram to catch readability issues.
---

# kroki

Render Mermaid (and other Kroki-supported) diagrams as PNG using the user's **local** Kroki Docker container, then **read the PNG back and review it visually** for readability before declaring the job done.

## When to use

- User asks for "a diagram", "architecture diagram", "flow diagram", "sequence diagram", "state machine", "lifecycle diagram", or similar.
- User asks for diagrams to put in docs / Confluence / a PR description.
- User says "render this mermaid" or "make a picture of this flow".

Do NOT use for: small inline diagrams that would fit in a code block (just use a fenced ```mermaid block). Use this skill when an image file is the actual deliverable.

## Prerequisites (verify before generating)

The user runs Kroki locally. The published port defaults to **8585** but is
overridable via the `KROKI_PORT` environment variable — resolve it through the
shell so the variable wins, and reuse `$PORT` everywhere below:

```bash
PORT="${KROKI_PORT:-8585}"
docker ps --filter "name=kroki" --format "{{.Names}}\t{{.Ports}}" \
  && curl -sf "http://localhost:$PORT/" > /dev/null && echo "Kroki OK" || echo "Kroki NOT REACHABLE"
```

If the container isn't running or the port doesn't respond, **stop** and tell the user:

> Kroki isn't reachable on `localhost:$PORT`. Start it with the bundled compose
> stack (recommended):
> ```bash
> docker compose up -d          # from the skill's repo dir
> ```
> …or manually on a shared network:
> ```bash
> docker network create kroki-net
> docker run -d --name kroki-mermaid --network kroki-net yuzutech/kroki-mermaid
> docker run -d --name kroki --network kroki-net \
>   -e KROKI_MERMAID_HOST=kroki-mermaid -p "${KROKI_PORT:-8585}:8000" yuzutech/kroki
> ```

Do not attempt to render until they confirm it's up.

## Ask first if context is thin

Before generating, confirm you have enough to make a *useful* diagram. If any of the following is unclear, ask the user **one consolidated question** (not a barrage):

- **Subject** — what system / flow / lifecycle the diagram should depict (the actual thing, not just the type)
- **Audience** — engineering, ops, product, customers, executives. Affects level of detail and jargon.
- **Scope** — full architecture vs. one slice (e.g., "just the cleanup path", "just the happy path")
- **Diagram type preference** — sequence vs. flowchart vs. state machine. Sometimes obvious from the subject; sometimes not.
- **How many diagrams** — one overview or several focused ones

Example clarifying ask: *"Before I render — is this an architecture overview, a step-by-step sequence, or a state machine? And who's reading it: engineering, or a customer-facing doc? That changes how much detail I bake in."*

Skip the ask if the request already includes the subject + at least one of {type, audience, scope}.

## Workflow

For each diagram requested:

1. **Decide the diagram type** based on the request (after asking if needed):
   - System or architecture overview → `flowchart LR` or `flowchart TB`
   - Step-by-step interaction (caller → service → external system) → `sequenceDiagram`
   - States and transitions of an entity → `stateDiagram-v2`
   - Dependency graph → `graph LR`
   - ERD → `erDiagram`
   - Git branching → `gitGraph`
   - Class hierarchy → `classDiagram`

2. **Write the `.mmd` source** to a **local-only** directory — diagrams are scratch artefacts intended for Confluence / Slack / docs, not committed to project repos.

   **Resolve the output base directory** from the `KROKI_DIAGRAMS_DIR`
   environment variable, falling back to `~/Documents/kroki-diagrams` when unset.
   Always expand it through the shell so the variable wins:

   ```bash
   BASE="${KROKI_DIAGRAMS_DIR:-$HOME/Documents/kroki-diagrams}"
   ```

   The full path for each source file is then:

   ```
   $BASE/<feature-or-topic-slug>/NN-<descriptive-name>.mmd
   ```

   Create the directory if it doesn't exist. The slug is short kebab-case (e.g., `custom-domain`, `billing-flow`, `auth-handshake`). NN is a zero-padded 2-digit ordinal (`01`, `02`, …).

   A user can also override the location for a single request by saying so
   explicitly ("put them in ./docs/diagrams") — an explicit instruction beats
   both the env var and the default. Never put diagrams under the current project
   repo unless told to — the default assumption is the user takes them to
   Confluence and doesn't want them tracked in git.

3. **Render to PNG** with white background — this is mandatory:
   ```bash
   curl -sX POST "http://localhost:$PORT/mermaid/png?bgColor=white" \
     -H 'Content-Type: text/plain' \
     --data-binary @<path>.mmd \
     -o <path>.png \
     -w "HTTP %{http_code}, %{size_download} bytes\n"
   ```

   For other diagram engines, swap `mermaid` in the URL for `plantuml`, `graphviz`, `bpmn`, `excalidraw`, `bytefield`, `nomnoml`, `wavedrom`, etc. See `https://kroki.io/#how-to-render-a-diagram`.

4. **Flatten the alpha channel** — this is mandatory. Kroki always outputs **RGBA** even with `?bgColor=white`. The pixels Mermaid draws are white, but the empty canvas regions are *transparent*. On dark Confluence / Slack themes, that transparency lets the page background bleed through and makes labels unreadable. Always strip the alpha channel:

   Use whichever flattener is available — detect, don't assume an OS:

   ```bash
   if command -v magick >/dev/null || command -v convert >/dev/null; then
     # ImageMagick — works on macOS, Linux, Windows (Git Bash / WSL)
     IM=$(command -v magick || command -v convert)
     "$IM" <path>.png -background white -alpha remove -alpha off <path>.png
   elif command -v sips >/dev/null; then
     # macOS built-in — JPEG round-trip forces RGB (JPEG has no alpha)
     sips -s format jpeg <path>.png --out /tmp/_flatten.jpg >/dev/null
     sips -s format png /tmp/_flatten.jpg --out <path>.png >/dev/null
     rm -f /tmp/_flatten.jpg
   else
     # Last resort — Python Pillow (pip install pillow); cross-platform
     python3 - "<path>.png" <<'PY'
   import sys; from PIL import Image
   p = sys.argv[1]; im = Image.open(p).convert("RGBA")
   bg = Image.new("RGB", im.size, (255, 255, 255)); bg.paste(im, mask=im.split()[3])
   bg.save(p)
   PY
   fi

   # Verify — output should say "8-bit/color RGB" (not RGBA).
   # `file` is absent on bare Windows; fall back to ImageMagick's identify.
   file <path>.png 2>/dev/null || { command -v identify >/dev/null && identify -format '%[channels]\n' <path>.png; }
   ```

   If the check still reports `RGBA` (or `rgba`), the flatten didn't take — retry
   with a different tool from the list above.

5. **Visually review the PNG** by Read-ing it (the tool returns the rendered image to the model). For each diagram, check the criteria in the next section and either:
   - Declare it acceptable, or
   - Edit the `.mmd` source and re-render (+ re-flatten) until it passes.

6. **Report results** to the user with a 1–2 line description of each diagram + the path. Do NOT embed the diagrams inside `.md` files unless the user explicitly asks — they may be heading to Confluence and want only the file references.

## Review criteria (the vision pass)

After Read-ing each PNG, check:

| Issue | Symptom | Fix |
|---|---|---|
| **Special chars in guards** | A bracket guard like `[CNAME wrong ✗]` renders as `[CNAME wrong [?]]` because Mermaid mis-parses unicode inside `alt`/`else` labels | Strip ✓/✗/emoji from guards; use plain text. Move the glyph into a `Note over` if needed. |
| **Too many lifelines** | Sequence diagram is wider than 1800 px or has 7+ participants | Split into multiple diagrams (`03a-…`, `03b-…`), or merge near-duplicate lifelines (e.g., "Customer" + "Customer's DNS" can become just "Customer" with a `Note`). |
| **Snaking arrows** | Flow lines cross over themselves or zigzag because nodes are placed awkwardly | Switch direction (`flowchart LR` ↔ `flowchart TB`) or move related nodes into a `subgraph` to constrain layout. |
| **Floating notes overlap** | `note right of X` produces a dashed connector that crosses real arrows | Drop redundant notes, or place them at the diagram edges (use `note left of` to push them outward). |
| **Tiny illegible text** | Diagram is wider than 2000 px and labels look pixelated when scaled down | Shorten participant names, abbreviate. Long technical names like `Background pool (customdomain-teardown)` → `Async pool` + a `Note over` with the long form. |
| **Font fallback** | Diagram uses default Mermaid font even though the source asked for Open Sans / Inter / etc. | The Kroki Mermaid container doesn't have custom fonts installed. Don't try to fight it — remove the `fontFamily` from `%%{init: …}%%` so the default doesn't generate a "missing font" warning. |
| **Transparent background** | PNG looks fine in your reader but has alpha — labels become unreadable on dark Confluence / Slack | `?bgColor=white` alone is NOT enough — Kroki still outputs RGBA. Must flatten with the OS-appropriate tool in step 4 (ImageMagick / sips / Pillow). Verify with `file <path>.png` → should say `8-bit/color RGB`, not `RGBA`. |
| **Logical errors** | Arrows don't match the actual code/flow, missing async vs sync distinction, etc. | Re-read the source code / docs the diagram represents, fix factual errors. |

If a fix requires re-rendering, **edit the `.mmd` and re-curl** — don't try to surgically patch the PNG.

## Standard init block

Start every Mermaid file with this header. No `fontFamily`, no theme variables that depend on installed fonts:

```mermaid
%%{init: {'theme':'default'}}%%
```

If the user wants a specific theme (forest, dark, base), add it — but don't add font hints. Theme variables for colors work fine; font ones don't unless they bake the font into the container.

## Useful color conventions

When using `classDef` for node coloring, these palettes look fine on white background and remain legible when shrunk:

| Role | Fill | Stroke |
|---|---|---|
| Edge / network function | `#e8f5f8` | `#5eadc6` |
| Data store / DB / cache | `#fff7e0` | `#d4a017` |
| External system / customer | `#f0f4ff` | `#3b5bdb` |
| Internal service | `#f3e8f8` | `#8b5cf6` |
| Async / background work | `#fff5d6` (in `rect rgb(…)` for sequence shading) | — |
| Synchronous / immediate | `#e8f5f8` (in `rect rgb(…)`) | — |

## Sequence diagram tip — rect for sync vs async

When a sequence has a synchronous prefix and an async tail (a common pattern), wrap each in a `rect rgb(…)` block with a `Note over` heading:

```
rect rgb(232, 245, 248)
Note over Svc: Synchronous (~2s)
… steps …
end

rect rgb(255, 245, 220)
Note over Pool: Async (1–10 min)
… steps …
end
```

This makes the phase boundary unmissable.

## Common Kroki gotchas

- **Long Mermaid sources** can fail silently with HTTP 400 — check the `-w "HTTP %{http_code}"` output every render.
- **POST body** must be UTF-8 — avoid stray BOMs or CRLF (some editors add them).
- **Render times** for big sequences can be 3–6 seconds — that's normal; not stuck.
- **`bgColor`** is supported for Mermaid PNG but NOT for SVG (SVG always has transparent background — set the bg in your stylesheet if needed).
- **PNGs are always RGBA** even with `?bgColor=white`. Kroki paints the canvas white but the output retains an alpha channel, so unused pixels remain transparent. Always flatten to RGB before delivering (see step 4) — otherwise the diagram is unreadable on dark themes.

## Final report format

When done, summarize as:

```
Generated N diagrams under $BASE/<slug>/:
  01-architecture.png         — <one-line description>
  02-state-machine.png        — <one-line description>
  …
All have white backgrounds and have been visually reviewed; specific tweaks made: <list>.
```

Don't list every byte count, dimension, or HTTP code in the final report — those are debugging output, not deliverables.

Reminder: do NOT embed these PNGs into project markdown files unless the user explicitly asks. The default assumption is that the user will copy them into Confluence or another external doc system.

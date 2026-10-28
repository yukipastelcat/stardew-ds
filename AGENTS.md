# AGENTS.md — stardew-ds (umbrella repo)

StardewDS lets a handheld Android device (touchscreen, gamepad-only) run
Stardew Valley with the game's HUD/inventory UI moved onto a companion
app. This repo is the **umbrella**: it contains no source code of its own,
only two git submodules, a devcontainer, a VS Code workspace, and the
release workflow.

| Path | Submodule of | What it is |
| --- | --- | --- |
| `mod/` | `yukipastelcat/stardew-ds-mod` | C# SMAPI mod (net6.0, Harmony). Hides the vanilla HUD and serves game state over local HTTP/WebSocket. |
| `companion/` | `yukipastelcat/stardew-ds-companion` | Flutter Android app. Connects to the mod and redraws the HUD/inventory/map/etc. |

Each submodule has its own `AGENTS.md` with the details — read it before
working in that directory. Changes to code belong in the submodule repo
(commit and push there), not in this repo.

## How the pieces fit together

- The mod runs a server on **port 8082** (`HttpListener`): `GET /ws`
  (WebSocket state stream), `GET /state`, `POST` actions (`/select`,
  `/move`, `/organize`, `/open-journal`) and many `GET` image endpoints
  (`/sprite`, `/portrait`, `/icon`, …) that serve PNGs cropped from the
  *real* game textures.
- The companion connects to it on `localhost` (mod and app run on the
  same device).
- The JSON contract lives in two places that must stay in sync:
  `mod/GameStateSnapshot.cs` (producer) and
  `companion/lib/models/game_state.dart` (consumer). A change to one side
  almost always needs the other. Keep the companion tolerant of older mod
  builds (it already ignores unknown fields / falls back when a newer
  query parameter is ignored).

## Layout

- `.devcontainer/` — Dockerfile + `devcontainer.json`: .NET SDK + Flutter
  (3.47.0) + Android tooling + adb. `postCreateCommand` runs
  `flutter pub get` in `companion/` and `dotnet --info`.
- `stardew-ds.code-workspace` — VS Code workspace with both folders and
  debug launch configs for the companion.
- `.github/workflows/release.yml` — manually dispatched (`version`,
  optional `mod_version` / `companion_version`). Pins the submodules to
  the given release tags, commits the pointer bump, downloads the mod
  `.zip` and companion `.apk` from those repos' releases and publishes a
  `stardew-ds` release. It builds nothing itself. Needs the
  `SUBMODULES_TOKEN` secret.

## Working in this repo

- Clone with `git clone --recurse-submodules`; submodule URLs are SSH
  (`git@github.com:…`). If `mod/` or `companion/` is empty, run
  `git submodule update --init`.
- Release flow is: bump version in each submodule → publish a release in
  each (their workflows build artifacts) → run this repo's `Release`
  workflow. Don't hand-edit submodule pointers except as part of that.
- There are no build or test commands at this level.

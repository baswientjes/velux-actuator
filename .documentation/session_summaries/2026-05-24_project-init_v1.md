# Session summary — 2026-05-24 — Project initialisation

## What was done

- Reviewed the existing `CLAUDE.md` (project context only, no code yet).
- Added the standard `# CLAUDE.md` prefix and a **Guidance for Claude** block containing:
  - Current phase statement (no code yet, mechanical commissioning pending)
  - Scope rules: no TBD values filled without user input, no final firmware until open items resolved, A4988 shutdown sequence must always be respected
  - ESPHome CLI commands for future use (`esphome config`, `esphome run`)
- Initialised a git repository in the project folder.
- Created an initial commit (`b55c537`).
- Created a public GitHub repo at https://github.com/baswientjes/velux-actuator.
- Push via `gh` CLI failed due to PAT scope issues; user will push via GitHub Desktop + VSCode.

## Decisions made

- Repo name: `velux-actuator`, public, personal account (`baswientjes`).
- `gh` CLI installed in WSL2 and authenticated via PAT (token requires `repo` scope for future `gh` operations).
- Git identity set locally in this repo: `Bas Wientjes / sebastiaanwientjes@gmail.com`.

## Next session

- Confirm push succeeded via GitHub Desktop.
- Resolve open items (window force measurement, gear ratio, endstop GPIO assignment, NEMA17 model).
- Begin ESPHome YAML scaffolding once open items are confirmed.

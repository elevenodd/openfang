# OpenFang Development Memory — Kaldway Solutions LLC

> Persistent context for Gemini. Updated: 2026-03-01.
> Full version: see `MEMORY.md` in project root.

## Project
- **OpenFang** v0.2.5 — open-source Agent OS written in Rust (14 crates)
- **Owner**: Ralph, Kaldway Solutions LLC
- **Business**: Environmental Site Assessments (ESA) per ASTM E1527-21
- **API**: `http://127.0.0.1:4200` — daemon command is `start`
- **Default model**: `gemini-2.5-flash` via `GEMINI_API_KEY` env var

## Custom Agents (Kaldway ESA Pipeline)
- **Philo** — Lead butler/orchestrator. Spawns Researcher + Drafter, manages `C:\AI_Work`, daily briefings.
- **Researcher** — EDR analysis, regulatory database review, structured Research Packages (RP-1 to RP-9).
- **Drafter** — NES/GEG report formatting, ASTM E1527-21 compliance, Sections 1-13.
- All three use `gemini-2.5-flash` and are registered in `bundled_agents.rs`.

## Skills Installed at `~/.openfang/skills/`
- `researcher/SKILL.md` — 9 Research Package templates, confidence flags, TERRA benchmarking
- `drafter/SKILL.md` — Full report formatting spec, boilerplate library, quality checklist
- Skills require YAML frontmatter (`---` block with name + description) to be detected

## Key Fixes (2026-03-01)
- **SSE streaming fix**: Normalized `\r\n` → `\n` in Gemini driver SSE parser (`drivers/gemini.rs`). Without this, streaming returns empty responses.
- **Safety settings**: Added `BLOCK_ONLY_HIGH` thresholds to prevent false positives on environmental content.
- **api_key_env**: This field holds the env var NAME (e.g., `GEMINI_API_KEY`), not the actual key.

## Workspace Paths
- Active projects: `C:\AI_Work\<ProjectName>`
- Training data: `D:\TERRA_ESA_Assistant_Files\DataFiles_for_Training\Organized_Projects`
- Config: `~/.openfang/config.toml`
- Skills: `~/.openfang/skills/`

## Pending Work
1. Update Researcher/Drafter agent.toml paths to correct D: drive location
2. Configure MCP servers for D: drive file access + Microsoft 365 email
3. End-to-end ESA pipeline test with real EDR data

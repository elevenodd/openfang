# OpenFang Development Memory — Kaldway Solutions LLC

> Persistent context file for Claude and Gemini. Updated: 2026-03-02.
> This file captures what was built, what works, what broke, and what's next.

---

## Project Identity

- **Product**: OpenFang — open-source Agent Operating System (Rust, 14 crates, v0.2.5)
- **Owner**: Ralph, Kaldway Solutions LLC
- **Business**: Environmental Site Assessments (ESA) per ASTM E1527-21
- **Goal**: Build an AI-powered ESA pipeline — Philo orchestrates, Researcher analyzes, Drafter writes reports

---

## What Was Built (Sessions: 2026-03-01 → 2026-03-02)

### Agents (all registered in `bundled_agents.rs`)
- **Philo** (`agents/philo/agent.toml`): Lead butler/orchestrator. 3 roles: ESA pipeline, email filter, butler. `workspace = "C:\\AI_Work"`, `mcp_servers = ["terra-training", "onedrive-nes"]`
- **Researcher** (`agents/researcher/agent.toml`): EDR analysis, regulatory review. `workspace = "C:\\AI_Work"`, `mcp_servers = ["terra-training"]`
- **Drafter** (`agents/drafter/agent.toml`): NES/GEG report formatting. `workspace = "C:\\AI_Work"`, `mcp_servers = ["terra-training"]`

### Skills Installed (`~/.openfang/skills/`)
- `researcher/SKILL.md` — 9 Research Packages (RP-1→RP-9), TERRA benchmarking
- `drafter/SKILL.md` — Full NES/GEG report spec, Sections 1-13, boilerplate

### MCP Filesystem Servers (`~/.openfang/config.toml`)
- `terra-training` → `D:\TERRA_ESA_Assistant_Files\DataFiles_for_Training` (14 tools)
- `onedrive-nes` → `C:\Users\ralph\OneDrive\National Environmental Services Projects` (14 tools)
- Both connect successfully on daemon boot (28 tools total)

---

## Critical Bugs Fixed

### 1. Gemini SSE Streaming Empty Response (FIXED ✓)
- `crates/openfang-runtime/src/drivers/gemini.rs` line ~561
- Root cause: `\r\n` line endings not normalized → SSE parser missed `\n\n` boundaries
- Fix: Normalize all line endings + JSON fallback + debug logging

### 2. Gemini Safety Filter False Positives (FIXED ✓)
- Added `BLOCK_ONLY_HIGH` safety settings to both `complete()` and `stream()`

### 3. MCP subprocess env_clear (FIXED ✓)
- `crates/openfang-runtime/src/mcp.rs` line ~402
- `connect_stdio()` called `cmd.env_clear()` then only passed `PATH`
- On Windows, npx.cmd needs `APPDATA`, `SystemRoot`, `USERPROFILE`, `PATHEXT`, etc.
- Fix: Added `SYSTEM_VARS` constant with 14 essential Windows env vars

### 4. MCP Tool Capability Filter (FIXED in code, not yet tested ✓)
- `crates/openfang-kernel/src/kernel.rs` `available_tools()` function
- `ToolInvoke` capability filter was removing MCP tools because they aren't in `[capabilities] tools`
- Fix: `is_mcp_tool()` bypass in the filter (MCP tools already filtered by allowlist)

### 5. MCP Tools Not Reaching Agents (OPEN BUG — #1 PRIORITY)
- Even after fix #4, `available_tools()` shows `cached_mcp_tools=0`
- `self.mcp_tools.lock()` returns empty vec despite `connect_mcp_servers()` logging 28 tools
- Debug logging added to trace the issue — rebuild done but daemon restart + test not yet run
- **Hypothesis**: Race condition or Mutex/Arc issue between `connect_mcp_servers` (async spawn) and `available_tools` (sync call). Both access `self.mcp_tools: std::sync::Mutex<Vec<ToolDefinition>>`.

---

## How to Start the Daemon

```bash
tasklist | grep -i openfang && taskkill //PID <pid> //F; sleep 3
export GEMINI_API_KEY=$(powershell.exe -Command "[System.Environment]::GetEnvironmentVariable('GEMINI_API_KEY', 'User')" | tr -d '\r\n')
cd /c/Users/ralph/.gemini/antigravity/playground/sidereal-coronal/openfang-dev
target/release/openfang.exe start &
sleep 8
curl -s http://127.0.0.1:4200/api/health
```

---

## Key Architecture Notes

- Agent workspace sandbox (`workspace_sandbox.rs`): Single root, no multi-dir support. MCP is the designed solution for external dirs.
- `available_tools()` in `kernel.rs:~4000`: Assembles builtins + skill tools + MCP tools, then filters by ToolInvoke capability
- MCP connection is async via `tokio::spawn` in `start_background_agents()`
- `mcp_tools: std::sync::Mutex<Vec<ToolDefinition>>` — populated by `connect_mcp_servers`, read by `available_tools`
- `mcp_connections: tokio::sync::Mutex<Vec<McpConnection>>` — holds live connections

## Windows Gotchas
- `PATH="/c/Users/ralph/.cargo/bin:$PATH"` for cargo
- `taskkill //PID <pid> //F` (double slashes in Git Bash)
- Release builds: ~10 minutes
- `gemini-pro` retired → use `gemini-2.5-flash`
- Skills need YAML frontmatter (`---` block) or parser silently rejects

---

## Next Steps (Priority Order)

1. **Fix MCP tool injection bug** — Debug why `self.mcp_tools` is empty in `available_tools()`. Detailed logging already added. Start daemon, send message, check `/tmp/openfang_debug.log`.
2. **End-to-end MCP test** — Once tools inject properly, test Philo listing D: drive training files
3. **Microsoft 365 email integration** — OAuth/Entra ID setup for Philo's email filter role
4. **First real ESA pipeline test** — Create project, feed EDR data, run Researcher→Drafter pipeline

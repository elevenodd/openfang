# OpenFang — Agent Instructions

## Project Overview
OpenFang is an open-source Agent Operating System written in Rust (14 crates).
- Config: `~/.openfang/config.toml`
- Default API: `http://127.0.0.1:4200`
- CLI binary: `target/release/openfang.exe` (or `target/debug/openfang.exe`)

## System Environment
**OS:** Windows 11 Home | **Shell:** Git Bash (use Unix syntax) | **Python:** 3.13.12
- Use `taskkill //PID <pid> //F` (double slashes in MSYS2/Git Bash)
- Paths: forward slashes in commands, backslashes escaped in TOML (`C:\\path`)

## BLOCKING REQUIREMENT: Context Management Before Troubleshooting

**Before attempting ANY fix, troubleshooting, or debugging:**

### Step 1: Check Official Documentation & Community Resources (MANDATORY)
You MUST check these resources BEFORE writing any fix code:

1. **Official Documentation:** https://www.openfang.sh/
   - Architecture guides
   - Configuration reference
   - Known issues and solutions

2. **GitHub Repository:** https://github.com/RightNow-AI/openfang
   - Search Issues tab for similar problems
   - Check closed PRs for fixes that may have been merged
   - Review Discussions for community solutions

### Step 2: Delegate Research to Exploration Agent
**Use the Task tool with subagent_type="Explore"** to spawn a research agent:

```python
Task(
    subagent_type="Explore",
    description="Search for [issue] solution",
    prompt="""Search for solutions to [specific error/issue]:
    - GitHub Issues: https://github.com/RightNow-AI/openfang/issues
    - Check closed PRs with fixes
    - Review Discussions for workarounds
    Provide working solutions with code examples."""
)
```

The exploration agent will:
- Search GitHub Issues for error messages or symptoms
- Review closed/merged PRs related to the problem area
- Check Discussions for community workarounds
- Summarize findings with links to relevant issues/PRs

**When to spawn exploration agent:**
- Unknown errors or unexpected behavior
- Configuration issues not documented in local files
- Multi-component problems (API + kernel + runtime)
- Performance or stability issues

**When to read local files directly:**
- Error message points to specific source file
- Issue is clearly architecture-related (need to understand code flow)
- Working on new feature (not debugging existing code)

### Step 3: Read Local Source Context
After checking external resources, read relevant source files:
- Error stack traces point to specific files
- Architecture documentation in `/docs` or crate-level `lib.rs` files
- Related test files that show intended behavior

### Why This Is Required
**Problem:** Wasting days debugging issues that have known solutions in documentation or community discussions.

**Example:** The exec_policy database caching issue (2 days) could have been found quickly by searching GitHub issues for "exec_policy" or "agent configuration not updating".

**This is a BLOCKING requirement** — do not attempt fixes until you've completed Steps 1-2.

## Build & Verify Workflow
After every feature implementation, run ALL THREE checks:
```bash
cargo build --workspace --lib          # Must compile (use --lib if exe is locked)
cargo test --workspace                 # All tests must pass (currently 1744+)
cargo clippy --workspace --all-targets -- -D warnings  # Zero warnings
```

## MANDATORY: Live Integration Testing
**After implementing any new endpoint, feature, or wiring change, you MUST run live integration tests.** Unit tests alone are not enough — they can pass while the feature is actually dead code. Live tests catch:
- Missing route registrations in server.rs
- Config fields not being deserialized from TOML
- Type mismatches between kernel and API layers
- Endpoints that compile but return wrong/empty data

### How to Run Live Integration Tests

#### Step 1: Stop any running daemon
```bash
tasklist | grep -i openfang
taskkill //PID <pid> //F
# Wait 2-3 seconds for port to release
sleep 3
```

#### Step 2: Build fresh release binary
```bash
cargo build --release -p openfang-cli
```

#### Step 3: Start daemon with required API keys
```bash
GROQ_API_KEY=<key> target/release/openfang.exe start &
sleep 6  # Wait for full boot
curl -s http://127.0.0.1:4200/api/health  # Verify it's up
```
The daemon command is `start` (not `daemon`).

#### Step 4: Test every new endpoint
```bash
# GET endpoints — verify they return real data, not empty/null
curl -s http://127.0.0.1:4200/api/<new-endpoint>

# POST/PUT endpoints — send real payloads
curl -s -X POST http://127.0.0.1:4200/api/<endpoint> \
  -H "Content-Type: application/json" \
  -d '{"field": "value"}'

# Verify write endpoints persist — read back after writing
curl -s -X PUT http://127.0.0.1:4200/api/<endpoint> -d '...'
curl -s http://127.0.0.1:4200/api/<endpoint>  # Should reflect the update
```

#### Step 5: Test real LLM integration
```bash
# Get an agent ID
curl -s http://127.0.0.1:4200/api/agents | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])"

# Send a real message (triggers actual LLM call to Groq/OpenAI)
curl -s -X POST "http://127.0.0.1:4200/api/agents/<id>/message" \
  -H "Content-Type: application/json" \
  -d '{"message": "Say hello in 5 words."}'
```

#### Step 6: Verify side effects
After an LLM call, verify that any metering/cost/usage tracking updated:
```bash
curl -s http://127.0.0.1:4200/api/budget       # Cost should have increased
curl -s http://127.0.0.1:4200/api/budget/agents  # Per-agent spend should show
```

#### Step 7: Verify dashboard HTML
```bash
# Check that new UI components exist in the served HTML
curl -s http://127.0.0.1:4200/ | grep -c "newComponentName"
# Should return > 0
```

#### Step 8: Cleanup
```bash
tasklist | grep -i openfang
taskkill //PID <pid> //F
```

### Key API Endpoints for Testing
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Basic health check |
| `/api/agents` | GET | List all agents |
| `/api/agents/{id}/message` | POST | Send message (triggers LLM) |
| `/api/budget` | GET/PUT | Global budget status/update |
| `/api/budget/agents` | GET | Per-agent cost ranking |
| `/api/budget/agents/{id}` | GET | Single agent budget detail |
| `/api/network/status` | GET | OFP network status |
| `/api/peers` | GET | Connected OFP peers |
| `/api/a2a/agents` | GET | External A2A agents |
| `/api/a2a/discover` | POST | Discover A2A agent at URL |
| `/api/a2a/send` | POST | Send task to external A2A agent |
| `/api/a2a/tasks/{id}/status` | GET | Check external A2A task status |

## Architecture Notes
- **Don't touch `openfang-cli`** — user is actively building the interactive CLI
- `KernelHandle` trait avoids circular deps between runtime and kernel
- `AppState` in `server.rs` bridges kernel to API routes
- New routes must be registered in `server.rs` router AND implemented in `routes.rs`
- Dashboard is Alpine.js SPA in `static/index_body.html` — new tabs need both HTML and JS data/methods
- Config fields need: struct field + `#[serde(default)]` + Default impl entry + Serialize/Deserialize derives

## Common Gotchas
- `openfang.exe` may be locked if daemon is running — use `--lib` flag or kill daemon first
- `PeerRegistry` is `Option<PeerRegistry>` on kernel but `Option<Arc<PeerRegistry>>` on `AppState` — wrap with `.as_ref().map(|r| Arc::new(r.clone()))`
- Config fields added to `KernelConfig` struct MUST also be added to the `Default` impl or build fails
- `AgentLoopResult` field is `.response` not `.response_text`
- CLI command to start daemon is `start` not `daemon`
- On Windows: use `taskkill //PID <pid> //F` (double slashes in MSYS2/Git Bash)

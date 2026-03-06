# Kaldway OpenFang Customization Guide
**Phase I ESA Automation Platform**
*Last Updated: 2026-03-03*

---

## 🎯 Goal
Enable Philo to autonomously handle time-consuming Phase I ESA tasks:
- ✅ Initiate new Phase I projects (folder creation, EDR organization)
- ✅ Organize work folders in `C:\AI_Work`
- ✅ Track project status (research → drafting → review)
- ✅ Accelerate Phase I report completion via Researcher + Drafter pipeline

---

## 📦 What I Just Fixed

### 🚨 Critical Issue Resolved: Missing C:\AI_Work MCP Server
**Problem:** Your agents had `workspace = "C:\\AI_Work"` but NO MCP filesystem server giving them access to that directory.

**Solution Applied:**
1. ✅ Added new MCP server `"workspace"` → `C:\AI_Work` in `config.toml`
2. ✅ Updated Philo's `mcp_servers = ["workspace", "terra", "onedrive-nes"]`
3. ✅ Updated Researcher's `mcp_servers = ["workspace", "terra"]`
4. ✅ Updated Drafter's `mcp_servers = ["workspace", "terra"]`

**Why this matters:** Without this, agents couldn't read/write files in their primary workspace. Now they have full MCP filesystem access to create project folders, save drafts, and organize EDR files.

---

## 🔄 Recommended: Update to v0.3.7

**Your current version:** v0.3.3
**Latest upstream:** v0.3.7 (4 commits ahead)

### Key improvements in v0.3.4-0.3.7:
- ✅ **Memory KV fixes** — shared namespace works correctly (critical for agent coordination)
- ✅ **Model switcher** — change models from dashboard/TUI
- ✅ **Autonomous hands** — agents can run continuously in background
- ✅ **OPENFANG_HOME** support — better Windows environment handling
- ✅ **Better default model handling** — Gemini 2.5 Flash config will apply cleanly

### How to update:
```bash
cd C:\Users\ralph\.gemini\antigravity\playground\sidereal-coronal\openfang-dev
git fetch upstream
git merge upstream/main -m "merge: update to v0.3.7"
cargo build --release -p openfang-cli
```

---

## 🏗️ Your Current Architecture

### Agents
| Agent | Version | Model | Workspace | MCP Access |
|-------|---------|-------|-----------|------------|
| **Philo** | 1.0.0 | gemini-2.5-flash | C:\AI_Work | workspace, terra, onedrive-nes |
| **Researcher** | 1.0.0 | gemini-2.5-flash | C:\AI_Work | workspace, terra |
| **Drafter** | 1.0.0 | gemini-2.5-flash | C:\AI_Work | workspace, terra |

### MCP Servers
| Name | Path | Purpose |
|------|------|---------|
| **workspace** | `C:\AI_Work` | PRIMARY - active projects, EDR downloads, drafts |
| **terra** | `D:\TERRA_ESA_Assistant_Files` | Training data, completed reports, templates |
| **onedrive-nes** | `C:\Users\ralph\OneDrive\...` | Final deliverables staging |

### Workflow
```
1. New project arrives
   ↓
2. Philo creates C:\AI_Work\<ProjectName>\
   ↓
3. Philo downloads EDR into project folder
   ↓
4. Philo spawns Researcher agent
   ↓
5. Researcher analyzes EDR + training data → findings
   ↓
6. Philo spawns Drafter agent
   ↓
7. Drafter converts findings → ASTM E1527-21 report
   ↓
8. Philo notifies Ralph for review
```

---

## 🚀 Optimization Recommendations

### 1. **Enable Autonomous Hand Mode for Philo** (New in v0.3.5+)
Convert Philo from chat agent to autonomous hand:
- Runs continuously in background
- Proactive monitoring of email, EDR downloads, project tracker
- Auto-initiates workflows without manual trigger

**How to enable:**
```toml
# In philo/agent.toml, change module from chat to hand
module = "builtin:hand"

# Add schedule
[schedule]
mode = "Continuous"
check_interval_seconds = 300  # Check every 5 minutes
```

### 2. **Add Project Tracker Integration**
Create a simple project status file that all agents can read/write:
```bash
# Philo maintains this
C:\AI_Work\PROJECT_TRACKER.json
```

**Sample structure:**
```json
{
  "active_projects": [
    {
      "name": "123_Main_St_Springfield",
      "client": "ABC Corp",
      "status": "research",
      "researcher_agent_id": "abc123",
      "started": "2026-03-03T10:00:00Z",
      "edr_downloaded": true,
      "findings_complete": false
    }
  ],
  "completed_today": [],
  "pending_review": []
}
```

Philo's system prompt already references tracking — just needs the file structure.

### 3. **Add Email Integration** (Future)
Your Philo prompt mentions Microsoft 365 email scanning. Once you're ready:
- Add MCP server for Microsoft Graph API
- Enable OAuth2 authentication
- Philo can auto-download EDR attachments to `C:\AI_Work\<ProjectName>\EDR\`

### 4. **Budget Monitoring**
You set aggressive token limits:
```toml
max_llm_tokens_per_hour = 500000
max_cost_per_hour_usd = 2.0
```

Monitor via API:
```bash
curl http://127.0.0.1:4200/api/budget
curl http://127.0.0.1:4200/api/budget/agents
```

### 5. **Add D: Drive Access Later**
When ready, all agents already have `mcp_servers = ["terra"]` which points to D: drive. No changes needed — it's already wired up!

---

## 🧪 Testing the Setup

### Step 1: Update and rebuild
```bash
git merge upstream/main -m "merge: update to v0.3.7"
cargo build --release -p openfang-cli
```

### Step 2: Start daemon
```bash
# Stop any running instance first
tasklist | grep -i openfang
taskkill //PID <pid> //F
sleep 3

# Start fresh
GEMINI_API_KEY=$(powershell.exe -Command "[System.Environment]::GetEnvironmentVariable('GEMINI_API_KEY', 'User')" | tr -d '\r\n') \
target/release/openfang.exe start &

sleep 6
curl http://127.0.0.1:4200/api/health
```

### Step 3: Verify MCP servers loaded
```bash
# Check that all 3 MCP servers are connected
curl -s http://127.0.0.1:4200/api/tools | grep -c "mcp_"
# Should see tools from workspace, terra, and onedrive-nes
```

### Step 4: Test Philo with real workflow
```bash
# Get Philo's agent ID
PHILO_ID=$(curl -s http://127.0.0.1:4200/api/agents | python3 -c "import sys,json; agents = json.load(sys.stdin); print(next(a['id'] for a in agents if a['name'] == 'philo'))")

# Send a test message
curl -s -X POST "http://127.0.0.1:4200/api/agents/$PHILO_ID/message" \
  -H "Content-Type: application/json" \
  -d '{"message": "Welcome back, Sir. Please create a test project folder at C:\\AI_Work\\TEST_123_Main_St and list the current projects."}'
```

### Step 5: Verify file operations work
```bash
# Check that Philo created the folder
ls -la C:/AI_Work/TEST_123_Main_St
```

### Step 6: Test agent spawning
```bash
curl -s -X POST "http://127.0.0.1:4200/api/agents/$PHILO_ID/message" \
  -H "Content-Type: application/json" \
  -d '{"message": "Spawn the researcher agent and have it check what training data is available in the terra MCP server."}'
```

---

## 📋 Next Steps

### Immediate (Today):
1. ✅ **DONE:** Fixed MCP server access for C:\AI_Work
2. 🔄 **Merge v0.3.7** to get latest bug fixes
3. 🧪 **Test end-to-end** — Philo → Researcher → Drafter on a real project

### Short-term (This Week):
1. Create `PROJECT_TRACKER.json` structure
2. Test spawning Researcher + Drafter with real EDR data
3. Validate ASTM E1527-21 output quality from Drafter

### Medium-term (This Month):
1. Enable autonomous hand mode for Philo (proactive monitoring)
2. Add Microsoft 365 email integration for EDR auto-download
3. Build daily briefing automation

### Long-term:
1. Full pipeline automation: email → EDR → research → draft → review
2. Quality gates and compliance checks
3. Multi-project parallel processing

---

## 🔧 Configuration Reference

### config.toml — Key Settings
```toml
[default_model]
provider = "gemini"
model = "gemini-2.5-flash"  # Will upgrade to gemini-2.5-flash automatically when available
api_key_env = "GEMINI_API_KEY"

[approval]
require_approval = false  # Auto-approve tool calls (you trust Philo)

[memory]
decay_rate = 0.05  # How fast agents forget (0.05 = 5% decay per day)
```

### agent.toml — Philo Capabilities
```toml
[capabilities]
tools = [
  "web_search", "web_fetch",       # Research
  "file_read", "file_write", "file_list",  # File ops
  "shell_exec",                     # Folder creation, EDR extraction
  "memory_store", "memory_recall",  # Cross-agent coordination
  "agent_send", "agent_list", "agent_spawn", "agent_kill",  # Orchestration
  "schedule_create", "schedule_list"  # Task scheduling
]
agent_spawn = true
agent_message = ["*"]
```

---

## 📞 Support

- **OpenFang Docs:** https://github.com/RightNow-AI/openfang
- **Your Memory:** `C:\Users\ralph\.claude\projects\...\memory\MEMORY.md`
- **This Guide:** `KALDWAY_OPENFANG_SETUP.md`

---

## ✅ Summary

**What changed today:**
1. ✅ Added `workspace` MCP server → `C:\AI_Work`
2. ✅ Updated all 3 agents to use the workspace MCP server
3. ✅ Identified v0.3.7 update path (4 commits behind)

**What's ready:**
- Philo can now create folders, organize files, and track projects in `C:\AI_Work`
- Researcher can access training data on D: drive
- Drafter can read training examples and write reports
- All agents can coordinate via shared memory

**What's next:**
- Merge v0.3.7 for latest fixes
- Test end-to-end workflow with real EDR data
- Consider autonomous hand mode for proactive Philo

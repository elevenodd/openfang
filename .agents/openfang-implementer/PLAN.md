# OpenFang v0.3.7 Merge + ESA Workflow Customization

## Context

Ralph has been using OpenFang v0.3.3 with custom fixes for his Phase I Environmental Site Assessment (ESA) workflow at Kaldway Solutions LLC. He needs to:
1. Update to v0.3.7 (4 commits behind) to get latest bug fixes
2. Resolve D drive MCP server issues he's experiencing
3. Verify Philo (butler) and Drafter agents are functional
4. Create a comprehensive customization plan for Phase I ESA automation
5. Learn how to maximize OpenFang for daily use

**Critical local changes to preserve:**
- `crates/openfang-runtime/src/drivers/gemini.rs` — Safety settings at BLOCK_ONLY_HIGH (prevents false positives on ESA content mentioning contamination, chemicals, etc.)
- `crates/openfang-runtime/src/mcp.rs` lines 426-448 — Windows env_clear bypass (Node.js v22+ CSPRNG fix)

**Current agent setup:**
- Philo: Orchestrator/butler for ESA pipeline coordination
- Researcher: Technical due diligence (EDR analysis, regulatory databases)
- Drafter: ASTM E1527-21 compliant report generation

**Model strategy (benchmark test):**
- Philo: `gemini-3.1-pro-preview-customtools` — Premium model for advanced tool orchestration and workflow coordination
- Researcher: `gemini-2.5-flash` — Cost-effective for structured analysis tasks
- Drafter: `gemini-2.5-flash` — Cost-effective for template-based writing (stable vs preview)

**MCP servers:**
- `workspace` → `C:\AI_Work` (just added, primary workspace)
- `terra` → `D:\TERRA_ESA_Assistant_Files` (training data — user reports "a lot of issues")
- `onedrive-nes` → OneDrive NES Projects (staging)

---

## Implementation Plan

### Phase 0: Create Persistent Claude Code Agent

**Objective:** Create a dedicated Claude Code agent that owns this implementation across multiple sessions

**Why this is needed:**
- Plan spans ~3.5 hours across multiple sessions
- Code changes, challenges, and decisions need to persist
- Agent maintains context and picks up where it left off
- Memory system tracks completed phases, blockers, and learnings
- Can be invoked anytime to continue work

**Agent Structure:**
```
.agents/
└── openfang-implementer/
    ├── AGENT.md          # Agent definition & core instructions
    ├── MEMORY.md         # Progress tracking & session log
    └── PLAN.md           # Copy of implementation plan
```

**Steps:**

1. **Create agent definition** → `.agents/openfang-implementer/AGENT.md`

   Core instructions for the agent:
   - Identity: Implementation Engineer for OpenFang v0.3.7 merge
   - Mission: Execute the implementation plan in PLAN.md
   - Workflow: Always read MEMORY.md first, update after each step
   - Authority: Can perform git operations, builds, file edits
   - Safety: Create rollback points, document decisions, ask before destructive ops

2. **Create memory tracking** → `.agents/openfang-implementer/MEMORY.md`

   Structure:
   - **Current Status**: Phase number, last action, next action
   - **Progress Checklist**: All steps with completion status
   - **Active Blockers**: What's blocking progress
   - **Key Decisions**: Important choices made and rationale
   - **Rollback Points**: Git branches/commits for safety
   - **Session Log**: Timeline of work across sessions
   - **Code Changes**: Files modified with descriptions
   - **Challenges**: Problems encountered and solutions

3. **Copy plan to agent** → `.agents/openfang-implementer/PLAN.md`

   Full implementation plan (this document) for agent reference

4. **Configure agent models** (benchmark strategy):
   - Update Philo: `model = "gemini-3.1-pro-preview-customtools"`
     - Rationale: Premium orchestration, advanced tool reasoning
   - Keep Researcher: `model = "gemini-2.5-flash"`
     - Rationale: Structured analysis, cost-effective
   - Keep Drafter: `model = "gemini-2.5-flash"`
     - Rationale: Template-based writing, proven stable

5. **Initialize agent state:**
   - Document workspace MCP changes already made
   - Document model configuration changes
   - Mark Phase 0 as in-progress
   - Note that Phases 1-5 are ready to execute
   - Set next action: Phase 1.1 (Pre-merge backup)

**How to use the agent:**
```bash
# Start/resume implementation work
# Agent will read MEMORY.md, identify current phase, and continue

# Agent automatically:
# - Reads MEMORY.md to restore context
# - Executes next step in plan
# - Updates MEMORY.md with progress
# - Documents decisions and changes
# - Creates rollback points before risky operations
```

**Deliverables:**
- [ ] Claude Code agent created in .agents/openfang-implementer/
- [ ] AGENT.md with core instructions
- [ ] MEMORY.md initialized with current state
- [ ] PLAN.md copied from implementation plan
- [ ] Philo model updated to gemini-3.1-pro-preview-customtools
- [ ] Researcher model confirmed as gemini-2.5-flash
- [ ] Drafter model confirmed as gemini-2.5-flash
- [ ] Phase 0 marked complete, ready for Phase 1

---

### Phase 1: Safe Merge to v0.3.7

**Objective:** Integrate 4 upstream commits while preserving Kaldway fixes

**Steps:**

1. **Pre-merge backup**
   ```bash
   cd "C:\Users\ralph\.gemini\antigravity\playground\sidereal-coronal\openfang-dev"
   git fetch upstream
   git branch backup-pre-v0.3.7
   git log --oneline HEAD..upstream/main  # Should show 4 commits
   ```

2. **Execute merge**
   ```bash
   git merge upstream/main --no-ff -m "merge: update from v0.3.3 to v0.3.7"
   ```

   **Expected conflicts:** Likely minimal or none. If conflicts occur in gemini.rs or mcp.rs:
   - Keep local changes (BLOCK_ONLY_HIGH, Windows env_clear bypass)
   - These are critical for ESA content and Windows MCP functionality

3. **Verification build**
   ```bash
   cargo clean
   cargo build --workspace --lib
   cargo test --workspace
   cargo clippy --workspace --all-targets -- -D warnings
   ```

   **Rollback if needed:**
   ```bash
   git merge --abort  # If still merging
   # OR
   git reset --hard backup-pre-v0.3.7  # If merge completed but broken
   ```

4. **Release build**
   ```bash
   cargo build --release -p openfang-cli
   ls -lh target/release/openfang.exe  # Should be ~15-25 MB
   ```

**Critical files:**
- `crates/openfang-runtime/src/drivers/gemini.rs:70-86` — Safety settings must remain BLOCK_ONLY_HIGH
- `crates/openfang-runtime/src/mcp.rs:426-448` — Windows env_clear bypass must be preserved

---

### Phase 2: Diagnose & Fix D Drive MCP Issues

**Objective:** Verify D drive MCP connectivity and resolve reported issues

**Current status:** D drive path is accessible (`ls "D:/TERRA_ESA_Assistant_Files"` succeeds), config.toml uses forward slashes correctly. Issues may be runtime connectivity, not configuration.

**Steps:**

1. **Stop any running daemon**
   ```bash
   tasklist | grep -i openfang
   taskkill //PID <pid> //F
   sleep 3
   ```

2. **Start daemon with debug logging**
   ```bash
   export GEMINI_API_KEY=$(powershell.exe -Command "[System.Environment]::GetEnvironmentVariable('GEMINI_API_KEY', 'User')" | tr -d '\r\n')

   RUST_LOG=openfang_runtime::mcp=debug,openfang=info \
   target/release/openfang.exe start &

   sleep 8
   curl -s http://127.0.0.1:4200/api/health
   ```

   **Look for in logs:**
   ```
   INFO openfang_runtime::mcp: MCP server connected server=terra tools=14
   ```

   **If terra fails:**
   ```
   ERROR openfang_runtime::mcp: Failed to connect to MCP server server=terra error="..."
   ```

3. **Verify MCP tools loaded**
   ```bash
   curl -s http://127.0.0.1:4200/api/tools | python3 -c "
   import sys, json
   tools = json.load(sys.stdin)
   terra_tools = [t for t in tools if 'terra' in t.get('name', '')]
   print(f'Terra MCP tools: {len(terra_tools)}')
   for t in terra_tools[:5]:
       print(f\"  - {t['name']}\")
   "
   ```

   **Expected:** ~14 tools with names like `mcp_terra_read_file`, `mcp_terra_list_directory`

4. **Test D drive read access**
   ```bash
   # Via direct path check (already confirmed working)
   ls -la "D:/TERRA_ESA_Assistant_Files"

   # Via MCP (requires agent to test, done in Phase 3)
   ```

5. **Common fixes if MCP fails:**
   - **npx not found:** Verify `"C:\Program Files\nodejs\npx.cmd" --version` works
   - **MCP package install fails:** Run `"C:\Program Files\nodejs\npx.cmd" -y @modelcontextprotocol/server-filesystem --help`
   - **Permission denied:** Check folder permissions with `icacls "D:\TERRA_ESA_Assistant_Files"`

**Diagnosis output:** Document whether MCP connects successfully and D drive tools are available.

---

### Phase 3: Verify Agents End-to-End

**Objective:** Confirm Philo, Researcher, and Drafter can coordinate on real tasks

**Steps:**

1. **List active agents**
   ```bash
   curl -s http://127.0.0.1:4200/api/agents | python3 -c "
   import sys, json
   agents = json.load(sys.stdin)
   print(f'Active agents: {len(agents)}')
   for a in agents:
       print(f\"  {a['name']}: {a.get('status', 'unknown')}\")
   "
   ```

2. **Get Philo agent ID**
   ```bash
   PHILO_ID=$(curl -s http://127.0.0.1:4200/api/agents | python3 -c "
   import sys, json
   agents = json.load(sys.stdin)
   philo = next((a for a in agents if a['name'] == 'philo'), None)
   print(philo['id'] if philo else 'NOT_FOUND')
   ")
   echo "Philo ID: $PHILO_ID"
   ```

   **If NOT_FOUND:** Agents may not auto-spawn. Check `~/.openfang/agents/` directory structure.

3. **Test Philo with workspace MCP**
   ```bash
   curl -s -X POST "http://127.0.0.1:4200/api/agents/$PHILO_ID/message" \
     -H "Content-Type: application/json" \
     -d '{"message": "List all directories in C:\\AI_Work. Then create a test folder called MERGE_TEST_2026_03_03."}' \
     | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('response', r))"
   ```

   **Success criteria:**
   - Philo lists existing directories
   - Creates test folder
   - No MCP tool errors

   **Verify:** `ls -la "C:/AI_Work/MERGE_TEST_2026_03_03"`

4. **Test Researcher with D drive access**
   ```bash
   RESEARCHER_ID=$(curl -s http://127.0.0.1:4200/api/agents | python3 -c "
   import sys, json
   agents = json.load(sys.stdin)
   r = next((a for a in agents if a['name'] == 'researcher'), None)
   print(r['id'] if r else 'NOT_FOUND')
   ")

   curl -s -X POST "http://127.0.0.1:4200/api/agents/$RESEARCHER_ID/message" \
     -H "Content-Type: application/json" \
     -d '{"message": "List the top-level directories in the terra training data (D drive). What ESA training files are available?"}' \
     | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('response', r))"
   ```

   **Success criteria:**
   - Researcher lists: `current_projects`, `DataFiles_for_Training`, `TERRA_AI`
   - No MCP connection errors
   - **This verifies D drive MCP works!**

5. **Test Philo → Researcher coordination**
   ```bash
   curl -s -X POST "http://127.0.0.1:4200/api/agents/$PHILO_ID/message" \
     -H "Content-Type: application/json" \
     -d '{"message": "Ask the researcher agent to count how many completed ESA reports are in the training data. Report back the count."}' \
     | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('response', r))"
   ```

   **Success criteria:**
   - Philo uses `agent_send` or `agent_message` tool
   - Researcher responds with count
   - Philo relays answer back

6. **Mock project workflow**
   ```bash
   curl -s -X POST "http://127.0.0.1:4200/api/agents/$PHILO_ID/message" \
     -H "Content-Type: application/json" \
     -d '{
       "message": "New Phase I ESA: 789 Oak Street, Chicago. Create project folder C:\\AI_Work\\789_Oak_St_Chicago with subfolders: EDR, Photos, Draft. Log this in PROJECT_LOG.md."
     }' \
     | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('response', r))"

   # Verify structure
   ls -la "C:/AI_Work/789_Oak_St_Chicago"
   cat "C:/AI_Work/789_Oak_St_Chicago/PROJECT_LOG.md"
   ```

7. **Cleanup test artifacts**
   ```bash
   rm -rf "C:/AI_Work/MERGE_TEST_2026_03_03"
   rm -rf "C:/AI_Work/789_Oak_St_Chicago"
   ```

**Critical files:**
- `C:\Users\ralph\.openfang\agents\philo\agent.toml` — Must have `mcp_servers = ["workspace", "terra", "onedrive-nes"]`
- `C:\Users\ralph\.openfang\agents\researcher\agent.toml` — Must have `mcp_servers = ["workspace", "terra"]`

---

### Phase 4: Customization Roadmap

**Objective:** Design practical enhancements for ESA automation

**Create documentation structure:**

1. **Project tracker template** → `C:\AI_Work\PROJECT_TRACKER.json`
   ```json
   {
     "schema_version": "1.0",
     "active_projects": [],
     "completed_projects": [],
     "on_hold_projects": []
   }
   ```

   **Schema for each project:**
   - id, name, client, address, status (initiated/research/drafting/review_ready/complete)
   - agent IDs (philo, researcher, drafter)
   - milestones (folder_created, edr_downloaded, research_complete, draft_complete)
   - file paths (edr, draft, final)
   - notes array

2. **Workflow documentation** → `C:\AI_Work\ESA_WORKFLOW_GUIDE.md`
   - New project checklist
   - EDR processing steps (manual now, auto later)
   - Research phase monitoring
   - Site recon integration
   - Drafting phase
   - Review and finalization
   - Tips for effective prompts

3. **Quality checklist** → `C:\AI_Work\.templates\QUALITY_CHECKLIST.md`
   - ASTM E1527-21 section completeness
   - REC analysis requirements
   - NES/GEG style compliance
   - Deliverables checklist

**Future enhancements to document:**
- Autonomous hand mode for Philo (proactive monitoring)
- Daily briefing automation (8 AM weekday reports)
- Microsoft 365 email integration (EDR auto-download)
- Multi-project parallel processing

---

### Phase 5: Teaching Materials

**Objective:** Create practical guides for daily use

**Documents to create:**

1. **Quick reference** → `C:\AI_Work\OPENFANG_QUICK_REF.md`
   - Daily startup procedure
   - Common commands (check agents, send messages, check budget)
   - Dashboard access
   - Troubleshooting tips

2. **Configuration reference** → `C:\AI_Work\CONFIG_REFERENCE.md`
   - File locations explained
   - Key settings (model, approval, memory decay, token limits)
   - MCP server configuration breakdown
   - Agent capabilities comparison (Philo vs Researcher vs Drafter)
   - Security notes

3. **Daily checklist** → `C:\AI_Work\DAILY_CHECKLIST.md`
   - Morning startup (5 min)
   - Check Philo status
   - New project workflow steps
   - Budget monitoring (weekly)
   - End of day shutdown (optional)
   - Weekly/monthly maintenance

4. **Troubleshooting guide** → `C:\AI_Work\TROUBLESHOOTING.md`
   - Connection refused → daemon not running
   - MCP tools not available → server connection failed
   - Gemini API errors → rate limit or safety block
   - Build failures → conflict resolution
   - Agent not responding → config or API key issues

---

## Verification Procedures

### Post-Merge Checklist
- [ ] `cargo build --workspace --lib` succeeds with zero warnings
- [ ] `cargo test --workspace` passes all tests
- [ ] `cargo clippy --workspace --all-targets -- -D warnings` clean
- [ ] Daemon starts without errors
- [ ] Health endpoint responds: `curl http://127.0.0.1:4200/api/health`

### Post-MCP Verification
- [ ] All 3 MCP servers show as connected in logs
- [ ] `curl http://127.0.0.1:4200/api/tools | grep mcp_terra` returns tools
- [ ] Researcher can list D drive directories
- [ ] Researcher can read files from D:\TERRA_ESA_Assistant_Files

### Post-Agent Verification
- [ ] Philo spawns/exists and responds to messages
- [ ] Philo can create folders in C:\AI_Work
- [ ] Researcher can access training data
- [ ] Philo can send messages to Researcher (coordination works)
- [ ] Mock project folder structure created successfully

### Documentation Complete
- [ ] Quick reference guide exists and is accurate
- [ ] Workflow guide covers full ESA pipeline
- [ ] Troubleshooting guide addresses common issues
- [ ] Configuration reference explains all settings

---

## Success Metrics

**Technical:**
- Zero compilation warnings
- All tests pass
- All 3 MCP servers connected
- All agents functional

**Functional:**
- End-to-end workflow completes (Philo → Researcher → Drafter)
- D drive training data accessible
- Project folder automation works

**Performance:**
- Daemon starts in <10 seconds
- Agent responses in <5 seconds (Gemini Flash)
- Research phase <1 hour
- Draft phase <2 hours

**Cost:**
- <200K tokens per project
- <$0.03 per project (Gemini Flash)
- <$5/month (10 projects)

---

## Risk Mitigation

**Merge conflicts:** Backup branch created, can rollback with `git reset --hard backup-pre-v0.3.7`

**Build failures:** Test compile before release build, can abort merge mid-process

**D drive issues:** Path already verified accessible. If MCP fails, can temporarily disable terra server and copy critical files to C:\AI_Work\training\

**Agent failures:** Test each capability independently. Can spawn agents manually via dashboard.

---

## Timeline

- **Phase 1 (Merge):** 25-30 minutes
- **Phase 2 (D drive diagnosis):** 15 minutes
- **Phase 3 (Agent verification):** 30 minutes
- **Phase 4 (Customization roadmap):** 1 hour (documentation)
- **Phase 5 (Teaching materials):** 1.5 hours (documentation)

**Total:** ~3.5 hours (1.25 hours technical, 2.25 hours documentation)

---

## Next Steps After Implementation

**Immediate (same day):**
- Test with real EDR data
- Validate Drafter output quality
- Process first real project

**Week 1:**
- Process 2-3 real projects through pipeline
- Iterate on prompts based on quality
- Establish baseline metrics

**Month 1:**
- Enable autonomous hand mode
- Implement PROJECT_TRACKER.json automation
- Build daily briefing

**Quarter 1:**
- Email integration (EDR auto-download)
- Quality gate automation
- Multi-project parallel processing

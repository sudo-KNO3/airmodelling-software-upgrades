# AERMOD Pipeline — Code Audit Report

**Date:** 2026-03-16
**Audited By:** Claude Code
**Project:** `N:\Central\Staff\KG\Kiefer's Coding Corner\Aermod\aermod_pipeline\`
**Overall Assessment:** PRODUCTION-QUALITY AGENTIC SYSTEM — functionally complete with targeted gaps

---

## Executive Summary

AERMOD Pipeline is a sophisticated, full-stack AI system that successfully automates the EPA AERMOD air dispersion modelling workflow end-to-end. The architecture is well-designed, the core agent loop is fully implemented and proven in production (40+ real completed runs), and the Ontario government data integration is real and functional. The primary gaps are security hardening for multi-user deployment, a small number of missing API endpoint helpers, and the fact that the web UI is not yet exposed as a polished standalone product.

---

## Architecture Assessment

### What Was Built

| Component | Status | Notes |
|---|---|---|
| ReAct agent loop | **Complete** | `agent.py` — full Reason→Act→Observe loop, MAX_STEPS=8, MAX_RETRIES=3 |
| Multi-agent orchestration | **Complete** | 5 specialized agents: Planner, Writer, Validator, Reviewer, Diagnostician |
| Ontario met data download | **Complete** | Real urllib.request to ontario.ca government website |
| Input file generation | **Complete** | All 5 processors: AERMOD, AERMET, AERMAP, BPIP, AERPLOT |
| Executable invocation | **Complete** | subprocess.run() in PipelineRunner + tools_executor |
| Knowledge base | **Complete** | 16 tools, 3,600+ recipes, vector RAG, verified configs |
| Persistent memory | **Complete** | memory.db (facts), run_history.db (lessons), vector_store.db (embeddings) |
| Self-healing / learning | **Complete** | diagnose_error, record_lesson, _inject_lessons, retry loop |
| SSE streaming API | **Complete** | /api/agent/run + /api/agent/stream with background threads |
| Test coverage | **Complete** | 18 test files, ~4,000+ lines |
| Real completed runs | **Confirmed** | 40+ workspaces with real .SFC/.PFL, aermod.out, .PLT, .kmz |

---

## Security Findings

### 1. No Authentication on Agent API
**Severity:** HIGH
**Endpoints:** `/api/agent/run`, `/api/chat/message`, `/api/agent/stream/<run_id>`

All agent and chat endpoints are unauthenticated. Any user on the network can start an agent run, stream results of another user's run by guessing a UUID, and consume OpenAI API quota without restriction.

**Impact:** API key abuse, unauthorized access to generated files and conversation history.

**Recommendation:** Add API key or session token validation before any multi-user deployment. For single-user local use this is acceptable.

---

### 2. Conversation Manager — In-Memory Only (No Restart Persistence)
**Severity:** MEDIUM
**File:** `chatbot/conversation.py` — `ConversationManager`

Conversation objects (`self.conversations`) are stored in a Python dict. Server restart loses all active conversations and their generated file references. The TTL is 1 hour and max is 100 conversations.

**Impact:** Users lose in-progress work on server restart. Does not affect completed runs (those are in workspaces on disk and in `run_history.db`).

**Recommendation:** Persist conversation metadata to SQLite on create/update, reload on startup.

---

### 3. Background Job Store — In-Memory Only
**Severity:** LOW
**File:** `app.py` — `_agent_jobs` dict

Active agent runs are tracked in memory only. Polling `/api/agent/poll/<run_id>` will return 404 after a server restart even for jobs that completed and wrote their results to `run_history.db`. The code does fall back to a DB lookup, so completed runs are recoverable.

**Impact:** SSE stream connections are lost on restart. Recoverable via `/api/agent/status/<run_id>` → DB fallback.

**Recommendation:** Low priority — the DB fallback handles most cases.

---

### 4. No Rate Limiting
**Severity:** MEDIUM

No per-user or per-IP rate limiting exists on the OpenAI-calling endpoints. A single runaway client can exhaust the API key quota.

**Recommendation:** Add Flask-Limiter or a simple per-IP token bucket before any exposure beyond localhost.

---

### 5. Workspace Cleanup — 7-Day TTL Only
**Severity:** LOW
**File:** `app.py` line 35 — `cleanup_old_workspaces(max_age_days=7)`

Workspace directories older than 7 days are deleted on server startup. This is a reasonable policy but could delete workspaces a user wants to keep. There is no user-triggered deletion or workspace archival.

**Recommendation:** Add a `DELETE /api/workspace/<name>` endpoint and/or an archival option.

---

### 6. No Input Sanitization on Workspace Tool Parameters
**Severity:** MEDIUM
**File:** `chatbot/tools_filesystem.py`

Workspace names are derived from LLM-generated strings. While the workspace root is enforced, the names themselves are not sanitized against path traversal characters (e.g. `../../`).

**Recommendation:** Apply `os.path.basename()` and sanitize to alphanumeric + underscore before joining to workspace root.

---

## Code Quality Findings

### 7. `orchestrator.py` — `_prepare_handoff()` Is a Stub
**Severity:** LOW
**File:** `chatbot/orchestrator.py`, lines 330–358

The `_prepare_handoff()` method — which is meant to copy output files between processors (e.g. AERMET `.sfc`/`.pfl` → AERMOD) — contains only `pass` statements with comments. The direct orchestrator (`Orchestrator.execute_plan()`) relies on files already being in the workspace from prior steps, which works when all steps run in the same workspace but is fragile.

**Impact:** The direct orchestrator (`/api/chat/orchestrate`) may fail for full pipeline runs. The multi-agent path (`MultiAgentOrchestrator`) handles this correctly because each agent explicitly writes files before the next agent runs.

**Recommendation:** Implement the handoff logic or add an assertion that the expected input files exist before each step.

---

### 8. Vector Store Search — Full In-Memory Scan
**Severity:** LOW
**File:** `chatbot/vector_store.py`, lines 472–531

Cosine similarity search loads all embeddings into memory and scores them in Python. For the current knowledge base size this is fine, but will scale poorly if the KB grows significantly.

**Impact:** None at current scale. Future concern if KB grows to tens of thousands of chunks.

**Recommendation:** No action needed now. If KB grows, consider SQLite-vec or a dedicated vector DB.

---

### 9. `app.py` — Startup Cleanup Exception Silently Swallowed
**Severity:** LOW
**File:** `app.py`, lines 33–39

```python
try:
    from chatbot.tools_filesystem import cleanup_old_workspaces
    _cleanup = cleanup_old_workspaces(max_age_days=7)
    ...
except Exception:
    pass  # non-critical
```

Any import error or cleanup failure is silently ignored. A missing module or permission error would be invisible.

**Recommendation:** At minimum log the exception at WARNING level.

---

### 10. `MultiAgentOrchestrator._persist_planner_values()` — Regex-Based Extraction
**Severity:** LOW
**File:** `chatbot/agent_orchestrator.py`, lines 609–676

The orchestrator extracts parameters (stack height, emission rate, UTM coordinates, etc.) from the Planner's text output using regex. This is fragile against varied LLM phrasings.

**Impact:** If the Planner uses different units or phrasing, values may not be extracted and the Writer may need to ask the user again. Self-correcting — the agent will ask if values are missing.

**Recommendation:** Prompt the Planner to always output a structured JSON summary block. Extract from JSON rather than free text.

---

## What Was Audited and Confirmed Correct

### Agent Loop (agent.py)
The `AgentLoop.run()` method is a complete, production-quality ReAct implementation:
- Builds conversation history with system prompt + prior messages + memory block
- Calls LLM with full tool schema
- Detects tool calls, dispatches them via `dispatch_agent_tool()`
- Injects tool results back into conversation
- Repeats until the LLM stops calling tools or MAX_STEPS is reached
- `_handle_failure()` extracts the error, calls `diagnose_error`, injects diagnosis, retries up to MAX_RETRIES=3
- `_inject_lessons()` queries `run_history.db` for past lessons before execution begins
- Protects downloaded Ontario met files from LLM overwrite attempts
- Auto-writes `.inp` files from build tools directly to workspace

### Ontario Data Download (tools_ontario.py)
Confirmed real HTTP downloads using `urllib.request`:
- `https://www.ontario.ca/sites/default/files/moe_mapping/mapping/data/met_data/1996_2000_regional_dataset/{region}_{land_use}.zip`
- `https://www.ontario.ca/sites/default/files/moe_mapping/mapping/data/DEM/cdem_dem_{tile_id}_tif.zip`
- Offline polygon lookup via `ontario_boundaries.json` (ray-casting algorithm)
- Haversine distance for DEM tile matching
- Nominatim geocoding for place name → lat/lon
- Input validation on lat/lon ranges and radius > 0
- Regex extraction of SF_ID, UA_ID, met_year from `.sfc` file headers

### Knowledge Base Tools (knowledge_base/chatbot_tools.py)
All 16 tools confirmed implemented and functional:
1. `list_processors()` — returns all processors
2. `get_processor_schema(processor)` — full keyword schema
3. `get_pathway_keywords(processor, pathway)`
4. `lookup_keyword(processor, keyword)`
5. `get_source_type_info(source_type)` — POINT/VOLUME/AREA/etc
6. `get_verified_configs(processor)`
7. `get_base_config(processor, stage)`
8. `find_recipe(tags, processor, top_k)` — SQLite tag-scored search with LRU cache
9. `validate_config(processor, data)` — validates against KB rules
10. `generate_input_file(processor, data)` — generates `.inp` content
11. `build_config_from_description(processor, description)` — merges onto base
12. `get_workflow_steps(scenario)`
13. `search_knowledge_base(query)` — free-text search
14. `get_dependencies(processor, feature)`
15. `build_and_generate_aermod_input(...)` — flat-param AERMOD generation
16. `build_and_generate_aerplot_input(...)` — flat-param AERPLOT generation

### AERMOD Validation (validators/aermod_validator.py)
Comprehensive validation including:
- Mandatory CO fields (TITLEONE, POLLUTID, MODELOPT output type, averaging times)
- DFAULT blocking rules (blocks FLAT, PVMRM, OLM, SCREEN, and 12 other options)
- NO2 method mutual exclusions (PVMRM/OLM/ARM/ARM2 — only one allowed)
- PVMRM/OLM ozone data source requirement
- BETA flag requirements for ARM2, LOWWIND1, LOWWIND2, PSDCREDIT
- Source parameter range checks (exit temp 200–2000 K, stack height 0–3000 m, diameter)
- Source type requirements (POINTCAP/POINTHOR require BETA, LINE width ≥ 1 m)
- ME pathway requirements (SURFFILE, PROFFILE, SURFDATA, UAIRDATA)
- RE pathway — at least one receptor or include file required
- OU pathway — warns if no output options specified

### Flask Application (app.py)
All endpoints are fully implemented — no stub endpoints:
- Direct generation for all 5 processors
- AERMOD validation endpoint
- Single tool execution and full pipeline execution
- All chatbot endpoints (new, message, files, download, orchestrate, generate_single)
- Complete async agent API (run, stream, poll, status, audit, history, lessons, export)
- Vector store API (index, search, stats)
- Startup workspace cleanup (7-day TTL)
- Background job eviction (10-minute TTL for completed jobs)

### Test Coverage (tests/)
18 test files confirmed in `tests/` directory:
- `test_agent_loop.py` — unit tests for AgentLoop
- `test_e2e_agent.py` — end-to-end agent tests
- `test_live_full_run.py` — live run against real executables
- `test_live_llm_aermod.py` — live LLM + AERMOD integration
- `test_ontario.py` — Ontario data download tests
- `test_multi_agent.py` — multi-agent orchestration tests
- Plus 12 additional test files covering parsers, generators, knowledge base, and more

### Real Run Evidence (data/workspaces/)
40+ completed workspace directories confirmed, including:
- `Toronto_crops_22112.SFC` / `.PFL` — Ontario MECP files (multi-MB real data)
- `Sudbury_crops_22112.SFC` / `.PFL` — Ontario MECP files
- `buffalo_2024.sfc` / `.pfl` — ECCC/NOAA met files
- `aermod.out` — AERMOD concentration outputs
- `ODOR_ANNUAL.PLT` — hourly concentration plot files
- `DEMO_AERPLOT.kmz` — Google Earth outputs
- `full_chain_test_20260316_180320/` — complete chain: AERMET stage1+2 → AERMOD → AERPLOT with real ECCC data
- Full AERMET intermediate outputs: `surf.ext`, `surf.qa`, `upp.ext`, `upp.qa`
- AERMAP outputs: `DOMDETAIL.OUT`, `MAPDETAIL.OUT`, `receptors.inc`
- BPIP outputs: `BPIP.OUT`, `BPIP.SUM`

---

## Findings Summary

| Severity | Issue | Count |
|---|---|---|
| HIGH | No authentication on agent/chat endpoints | 1 |
| MEDIUM | No rate limiting on OpenAI-calling endpoints | 1 |
| MEDIUM | Workspace name not sanitized against traversal | 1 |
| MEDIUM | Conversation state lost on server restart | 1 |
| LOW | `_prepare_handoff()` stub in direct orchestrator | 1 |
| LOW | Vector search is full in-memory scan | 1 |
| LOW | Startup cleanup exception silently swallowed | 1 |
| LOW | Planner value extraction via regex (fragile) | 1 |
| LOW | Background job store lost on restart | 1 |

**Total: 9 issues** — none are blockers for current single-user use.

---

## Recommendations — Prioritised

### Phase 1: Before Any Multi-User Deployment (Critical)
1. Add API key or session token authentication to all `/api/agent/` and `/api/chat/` endpoints
2. Add rate limiting (e.g. Flask-Limiter, 10 req/min per IP on agent endpoints)
3. Sanitize workspace names: `re.sub(r'[^a-zA-Z0-9_-]', '_', name)` before path join

### Phase 2: Reliability Improvements
4. Persist conversation metadata to SQLite so conversations survive server restart
5. Implement `_prepare_handoff()` in `orchestrator.py` for the direct pipeline path
6. Log startup cleanup exceptions at WARNING level instead of silently swallowing
7. Change Planner prompt to output a structured JSON block for parameter extraction

### Phase 3: Scalability (Low Priority, Future)
8. Replace full-scan vector search with SQLite-vec or pgvector if KB grows significantly
9. Add workspace archival / user-triggered deletion endpoint
10. Add per-run token usage tracking and budget cap

---

## Overall Assessment

This is a mature, functionally complete agentic AI system for AERMOD air modelling. The core engineering — the ReAct loop, multi-agent orchestration, Ontario data pipeline, knowledge base, and persistence layer — is all real, tested, and proven. The 40+ completed real-world runs in `data/workspaces/` are strong evidence that the system works end-to-end.

The gaps are:
1. Security hardening required before any exposure beyond the developer's machine
2. One stub method in the lesser-used direct orchestrator path
3. Minor robustness improvements to error logging and parameter extraction

For the intended use case (single-user local tool for an environmental professional), the system is production-ready today.

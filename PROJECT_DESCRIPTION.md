# AERMOD Pipeline — Project Description

**"ChatGPT for Air Modelling"**
**Audit Date:** 2026-03-16
**Project Location:** `N:\Central\Staff\KG\Kiefer's Coding Corner\Aermod\aermod_pipeline\`

---

## What Is This?

AERMOD Pipeline is a full-stack, AI-powered system for preparing, running, and interpreting EPA AERMOD air dispersion model runs. It wraps the entire AERMOD ecosystem (AERMOD, AERMET, AERMAP, BPIP, AERPLOT) in a Flask web application that exposes both a form-based UI and a natural language chatbot. Users can describe a scenario in plain English and the system autonomously generates all required input files, downloads real meteorological data from the Ontario government, runs the Fortran executables, interprets the output, and self-corrects on failure.

---

## The Problem Being Solved

EPA's AERMOD modelling system is the regulatory standard for air dispersion analysis across North America. The toolchain is powerful but notoriously difficult:
- Multiple Fortran executables must be run in the correct sequence
- Input files use rigid, whitespace-sensitive formats from the 1980s
- Meteorological and terrain data must be acquired, validated, and converted manually
- Users must read hundreds of pages of technical documentation

This platform eliminates all of those barriers through an AI agent that understands AERMOD deeply.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.x, Flask 3.0 |
| AI / LLM | OpenAI API (gpt-4o-mini default), function-calling / tool use |
| Embeddings | OpenAI text-embedding-3-small (RAG over knowledge base) |
| Persistence | SQLite: memory.db, run_history.db, vector_store.db, config_recipes.db |
| Met Data | Ontario MECP 1996–2000 regional dataset via urllib.request |
| Geocoding | Nominatim / OpenStreetMap (no API key required) |
| Executables | EPA Fortran binaries: AERMOD, AERMET, AERMAP, BPIPPRM, AERPLOT |
| Frontend | Flask/Jinja2 templates, vanilla JavaScript |

---

## Directory Structure

```
aermod_pipeline/
├── app.py                          # Flask entry point — all REST + SSE endpoints
├── requirements.txt                # flask>=3.0, openai>=1.0, python-dotenv>=1.0
├── .env                            # OPENAI_API_KEY + OPENAI_MODEL
│
├── chatbot/
│   ├── agent.py                    # Core ReAct agent loop (AgentLoop class)
│   ├── agent_orchestrator.py       # MultiAgentOrchestrator — 5-agent pipeline
│   ├── agents.py                   # Agent factory: Writer, Reviewer, Planner, Validator, Diagnostician
│   ├── conversation.py             # ConversationManager + per-conversation state
│   ├── llm_client.py               # OpenAI client + full OpenAI function-calling tool schema
│   ├── memory.py                   # Persistent key-value memory in SQLite (memory.db)
│   ├── orchestrator.py             # Direct workflow orchestrator (no LLM)
│   ├── prompts.py                  # System prompts for each agent role
│   ├── run_history.py              # Run history + lessons database (run_history.db)
│   ├── audit.py                    # AuditAgent — per-run audit trail
│   ├── tools_ontario.py            # Ontario MECP met data download (urllib.request)
│   ├── tools_executor.py           # run_processor, run_pipeline, check_run_status
│   ├── tools_filesystem.py         # create_workspace, write_file, list_workspace, read_file
│   ├── tools_parser.py             # parse_plt_file, parse_aermod_summary, interpret_results
│   ├── tools_history.py            # diagnose_error, record_lesson, query_past_runs
│   └── vector_store.py             # SQLite vector store with cosine similarity search
│
├── knowledge_base/
│   ├── chatbot_tools.py            # 16 LLM-callable tools via TOOLS_MANIFEST
│   ├── processor_schema.json       # Full keyword/parameter schema for all processors
│   ├── verified_configs.json       # Verified working base configurations
│   ├── config_recipes.db           # SQLite — 3,600+ tagged configuration recipes
│   └── _gen_recipes.py             # Recipe DB generator script
│
├── file_generators/
│   ├── aermod_gen.py               # generate_aermod_inp()
│   ├── aermet_gen.py               # generate_aermet_stage1(), generate_aermet_stage2()
│   ├── aermap_gen.py               # generate_aermap_inp()
│   ├── bpip_gen.py                 # generate_bpip_inp()
│   └── aerplot_gen.py              # generate_aerplot_inp()
│
├── validators/
│   └── aermod_validator.py         # Standalone AERMOD config validation rules
│
├── runners/
│   └── pipeline_runner.py          # PipelineRunner — subprocess calls to Fortran executables
│
├── models/
│   └── project.py                  # Project data model
│
├── templates/
│   ├── index.html                  # Form-based UI
│   └── chat.html                   # Chatbot UI
│
├── static/
│   ├── css/style.css
│   └── js/app.js
│
├── tests/                          # 18 test files (~4,000+ lines)
│   ├── test_agent_loop.py
│   ├── test_e2e_agent.py
│   ├── test_live_full_run.py
│   ├── test_live_llm_aermod.py
│   ├── test_ontario.py
│   ├── test_multi_agent.py
│   └── (12 additional test files)
│
├── data/
│   ├── memory.db                   # Agent persistent conversation memory
│   ├── run_history.db              # All past runs + learned lessons
│   ├── vector_store.db             # Embedded knowledge base (OpenAI embeddings)
│   ├── ontario_boundaries.json     # Offline Ontario region polygon data (ray-casting)
│   └── workspaces/                 # 40+ completed run directories
│       ├── Toronto_crops_*/        # Real MECP .SFC/.PFL + aermod.out
│       ├── Barrie_*/
│       ├── Sudbury_*/
│       ├── full_chain_test_*/      # Complete AERMET→AERMOD→AERPLOT chain runs
│       └── ...
│
└── executables/                    # EPA Fortran binaries (not in repo)
    ├── aermod.exe
    ├── aermet.exe
    ├── aermap.exe
    ├── Bpipprm.exe
    └── aerplot.exe
```

---

## Core Capabilities

### 1. Natural Language Input File Generation
Users describe a scenario in plain English. The LLM agent uses the knowledge base tools to look up the correct keywords, build validated configurations, and write all required `.inp` files into a timestamped workspace directory.

Example: *"Generate an AERMOD run for a 25 m stack in Toronto emitting 2.5 g/s of PM2.5"* → full `aermod.inp`, Ontario met download, execution, interpreted results.

### 2. Real Ontario MECP Meteorological Data (urllib.request)

The system downloads real, pre-processed meteorological data from the Ontario Ministry of the Environment, Conservation and Parks:

- **Base URL:** `https://www.ontario.ca/sites/default/files/moe_mapping/mapping/data/`
- **Met files:** `met_data/1996_2000_regional_dataset/{region}_{land_use}.zip` → `.sfc` + `.pfl`
- **Terrain:** `DEM/cdem_dem_{tile_id}_tif.zip` → GeoTIFF
- **Land use variants:** Crops, Forest, Urban, Suburban, Raw
- **Geocoding:** Nominatim/OpenStreetMap converts place names to lat/lon
- **Spatial lookup:** Ray-casting point-in-polygon against offline `ontario_boundaries.json`
- **NTS tile matching:** Haversine distance to match lat/lon to CDEM tile IDs
- **Key function:** `download_ontario_met_to_workspace(latitude, longitude, workspace)` — full pipeline, returns `sfc_file`, `pfl_file`, `station_id`, `ua_station_id`, `met_year`
- **Effect:** Replaces the need to run AERMET — the pre-processed `.sfc`/`.pfl` files are used directly in AERMOD

### 3. Multi-Agent AI Architecture (ReAct Pattern)

Five specialized agents coordinate via `MultiAgentOrchestrator`:

| Agent | Role | Tool Subset |
|---|---|---|
| **Planner** | Determines which processors are needed and in what order | KB read-only, geocoding, history |
| **Writer** | Creates workspaces, generates input files, downloads met data | KB all 16, filesystem write, Ontario tools, memory |
| **Validator** | Pre-flight config checks and post-run output quality gates | KB read-only, filesystem read, parsers |
| **Reviewer** | Runs executables, reads output, parses and interprets results | Executor, parsers, filesystem read |
| **Diagnostician** | Diagnoses errors, applies fixes, records lessons | History/learning, executor, KB fix tools |

The orchestrator classifies user intent (`generate` / `execute` / `review` / `diagnose` / `validate` / `plan` / `conversation`) and routes to the appropriate agent sequence. After generation, validation and review run automatically. On failure, the Diagnostician is invoked. Maximum 6 orchestrator rounds per run.

### 4. Self-Healing with Learning

- `AgentLoop` in `agent.py` implements the full ReAct loop: reason → act → observe → repeat
- `MAX_STEPS=8`, `MAX_RETRIES=3`
- On failure: `_handle_failure()` calls `diagnose_error`, injects the diagnosis into the conversation, retries
- After each run: lessons written to `run_history.db` and injected into future agent system prompts via `_inject_lessons()`
- The orchestrator auto-extracts key numeric values from the Planner's output and stores them in `memory.db` so the Writer and follow-up turns always have context without re-asking the user

### 5. Knowledge Base (3,600+ Recipes)

- **`processor_schema.json`:** Full keyword/parameter definitions for AERMOD, AERMET, AERMAP, BPIP, AERPLOT
- **`verified_configs.json`:** Verified working base configurations for each processor
- **`config_recipes.db`:** SQLite with 3,600+ tagged recipes, scored by tag match (LRU cached)
- **16 LLM-callable tools** exposed via OpenAI function-calling schema (in `llm_client.py`)
- **RAG:** `VectorStore` (SQLite + OpenAI embeddings) searched per-turn; relevant context injected as a transient system message before each LLM call

### 6. Direct Executable Execution

- `runners/pipeline_runner.py`: `subprocess.run()` calls to Fortran binaries, 600 s timeout
- `chatbot/tools_executor.py`: `run_processor()` used by AI agents
- Supports individual tool runs and full ordered pipelines
- Processor execution order enforced: AERMET → AERMAP → BPIP → AERMOD → AERPLOT

### 7. Input File Generation for All Processors

| Processor | Generator | Outputs |
|---|---|---|
| AERMOD | `aermod_gen.py` | aermod.inp |
| AERMET Stage 1 | `aermet_gen.py` | stage1.inp |
| AERMET Stage 2 | `aermet_gen.py` | stage2.inp |
| AERMAP | `aermap_gen.py` | aermap.inp |
| BPIP | `bpip_gen.py` | BPIP.inp |
| AERPLOT | `aerplot_gen.py` | aerplot.inp |

### 8. Asynchronous Agent API with Server-Sent Events (SSE)

- `POST /api/agent/run` — starts agent in background thread, returns `run_id` immediately
- `GET /api/agent/stream/<run_id>` — SSE stream of step-by-step progress (action type, tool name, content)
- `GET /api/agent/poll/<run_id>` — polling fallback for simple clients
- Background job store in-memory with 10-minute TTL eviction
- `GET /api/agent/history` — browse all past runs
- `GET /api/agent/audit/<run_id>` — full audit trail per run
- `GET /api/agent/lessons` — view all learned lessons
- `GET /api/agent/export/<run_id>` — export complete run as JSON

---

## AERMOD Ecosystem Processors

| Processor | Purpose | Key Outputs |
|---|---|---|
| AERMET | Meteorological pre-processing | .sfc, .pfl, surf.qa, upp.qa |
| AERMAP | Terrain elevation processing | receptors.inc, DOMDETAIL.OUT |
| BPIPPRM | Building downwash dimensions | BPIP.OUT, BPIP.SUM |
| AERMOD | Gaussian plume dispersion model | aermod.out, .PLT files |
| AERPLOT | Post-processing / KMZ visualization | .kmz (Google Earth) |

---

## Data Flow

```
User message (natural language)
       ↓
MultiAgentOrchestrator
  → classify_intent()
       ↓
Planner Agent (4 steps max)
  → determines processors, extracts parameters to memory
       ↓
Writer Agent (8 steps max)
  → download_ontario_met_to_workspace() — real .sfc/.pfl from ontario.ca
  → find_recipe() + validate_config() + generate_input_file()
  → create_workspace() + write_file()
       ↓
Validator Agent — pre-flight quality check
       ↓
Reviewer Agent (8 steps max)
  → run_processor() — subprocess to Fortran binary
  → parse_aermod_summary() → interpret_results()
       ↓
[On failure] Diagnostician Agent
  → diagnose_error() + record_lesson() + fix + retry
       ↓
Final reply + files_generated list + workspace path
```

---

## Persistence Architecture

| Database | Contents |
|---|---|
| `data/memory.db` | Per-conversation key-value facts (source type, coordinates, met files, etc.) |
| `data/run_history.db` | All past runs, per-task status, audit logs, learned lessons |
| `data/vector_store.db` | Embedded knowledge base chunks with OpenAI text-embedding-3-small |
| `knowledge_base/config_recipes.db` | 3,600+ tagged configuration recipes |

---

## REST API Summary

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Form-based UI |
| GET | `/chat` | Chatbot UI |
| GET | `/api/reference` | All reference data for the form UI |
| POST | `/api/generate/<processor>` | Direct file generation (no LLM) |
| POST | `/api/validate/aermod` | AERMOD config validation |
| POST | `/api/run/<tool>` | Run a single executable |
| POST | `/api/run/pipeline` | Run full pipeline |
| POST | `/api/chat/new` | Start a new conversation |
| POST | `/api/chat/message` | Send a message (LLM + tool-use mode) |
| GET | `/api/chat/download/<id>/<key>` | Download a generated file |
| POST | `/api/agent/run` | Start async multi-agent run |
| GET | `/api/agent/stream/<run_id>` | SSE step stream |
| GET | `/api/agent/poll/<run_id>` | Poll run status |
| GET | `/api/agent/history` | Browse past runs |
| GET | `/api/agent/audit/<run_id>` | Full audit trail |
| GET | `/api/agent/lessons` | View learned lessons |
| GET | `/api/agent/export/<run_id>` | Export run as JSON |
| POST | `/api/vector/index` | Re-index knowledge base |
| POST | `/api/vector/search` | Semantic search |
| GET | `/api/vector/stats` | Index statistics |

---

## Confirmed Operational Evidence

`data/workspaces/` contains 40+ completed real runs, including:
- `Toronto_crops_22112.SFC` / `.PFL` — real Ontario MECP files
- `Sudbury_crops_22112.SFC` / `.PFL` — real Ontario MECP files
- `buffalo_2024.sfc` / `.pfl` — ECCC/NOAA met files
- `aermod.out` — confirmed AERMOD outputs
- `ODOR_ANNUAL.PLT` — hourly concentration plot files
- `DEMO_AERPLOT.kmz` — Google Earth KMZ outputs
- `full_chain_test_20260316_180320/` — complete AERMET stage1+2 → AERMOD → AERPLOT chain with ECCC data
- AERMET outputs: `stage1.rpt`, `stage2.rpt`, `surf.ext`, `surf.qa`, `upp.ext`, `upp.qa`
- AERMAP outputs: `DOMDETAIL.OUT`, `MAPDETAIL.OUT`, `receptors.inc`
- BPIP outputs: `BPIP.OUT`, `BPIP.SUM`

---

## Dependencies

```
flask>=3.0
openai>=1.0
python-dotenv>=1.0
```

Requires `OPENAI_API_KEY` (and optionally `OPENAI_MODEL`) in `.env`. EPA Fortran executables must be placed in `executables/`.

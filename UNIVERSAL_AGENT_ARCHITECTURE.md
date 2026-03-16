# Universal Source-Aware Software Agent — Architecture

**Date:** 2026-03-16
**Status:** Design / Pre-Build
**Origin:** Evolved from aermod_pipeline — AERMOD AI automation system

---

## Concept

A system that maps the source code of any software at startup, builds a deep
structural understanding of what the program does and how, then inserts itself
as an intelligent agent layer that lets users operate the software through
natural language.

Instead of mapping a UI or API from the outside, the system reads the source
code itself and derives constraints, execution order, data flows, valid parameter
ranges, and error states directly from the code — with no manual knowledge base
curation required.

---

## High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          USER INTERFACE LAYER                                   │
│                                                                                 │
│  Browser / Desktop UI                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Natural Language Input                                                   │  │
│  │  "Model a 25m stack in Toronto emitting 2.5 g/s NO2"                    │  │
│  │  "Draw a floor plan for a 10m x 8m room in AutoCAD"                     │  │
│  │  "Run a stormwater analysis for this catchment"                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│               │                           │                                     │
│         SSE Stream                   REST API                                   │
│         (live steps)              (send message)                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                │                           │
                ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         FLASK APPLICATION LAYER                                 │
│                                                                                 │
│  app.py                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│  │ POST            │  │ GET             │  │ GET             │                │
│  │ /api/agent/run  │  │ /api/agent      │  │ /api/software   │                │
│  │                 │  │ /stream/{id}    │  │ /list           │                │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                │
│                                                                                 │
│  Background Job Store (in-memory, 10-min TTL)                                  │
│  thread → run_id → {status, steps, result, step_queue}                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       STARTUP MAPPING ENGINE                                    │
│                    (runs once per software version)                             │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                         SOURCE LOCATOR                                    │ │
│  │                                                                           │ │
│  │   target_software = "AERMOD"                                              │ │
│  │           │                                                               │ │
│  │           ├── Has source code? ──► SourceCodeMapper                      │ │
│  │           ├── Has SDK/headers? ──► SDKMapper                             │ │
│  │           ├── Has binary only? ──► BinaryAnalyzer                        │ │
│  │           └── Cached model?    ──► Load from disk (skip mapping)         │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                            │
│        ┌───────────────────────────┼──────────────────────────┐                │
│        ▼                           ▼                          ▼                │
│  ┌──────────────┐         ┌─────────────────┐       ┌──────────────────┐       │
│  │ SOURCE CODE  │         │   SDK MAPPER    │       │ BINARY ANALYZER  │       │
│  │   MAPPER     │         │                 │       │                  │       │
│  │              │         │ Reads:          │       │ Reads:           │       │
│  │ Reads:       │         │ - Header files  │       │ - Import tables  │       │
│  │ - .f files   │         │ - API docs      │       │ - String tables  │       │
│  │ - .py files  │         │ - Type defs     │       │ - Debug symbols  │       │
│  │ - .cpp files │         │ - Enumerations  │       │ - Probed I/O     │       │
│  │              │         │                 │       │                  │       │
│  │ Produces:    │         │ Produces:       │       │ Produces:        │       │
│  │ Full model   │         │ Partial model   │       │ Partial model    │       │
│  └──────┬───────┘         └────────┬────────┘       └────────┬─────────┘       │
│         │                          │                         │                 │
│         └──────────────────────────┼─────────────────────────┘                 │
│                                    │                                            │
│                                    ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                       ANALYSIS PIPELINE                                   │ │
│  │                                                                           │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │ │
│  │  │ AST PARSER  │  │ CALL GRAPH  │  │  DATA FLOW   │  │  CONSTRAINT   │  │ │
│  │  │             │  │ BUILDER     │  │  ANALYZER    │  │  EXTRACTOR    │  │ │
│  │  │ Fortran:    │  │             │  │              │  │               │  │ │
│  │  │ fparser2    │  │ What calls  │  │ How data     │  │ IF blocks     │  │ │
│  │  │             │  │ what, when, │  │ moves between│  │ → valid       │  │ │
│  │  │ Python:     │  │ with what   │  │ functions    │  │   ranges      │  │ │
│  │  │ ast module  │  │ arguments   │  │              │  │               │  │ │
│  │  │             │  │             │  │ Which inputs │  │ FORMAT stmts  │  │ │
│  │  │ C/C++:      │  │ Produces:   │  │ feed which   │  │ → field       │  │ │
│  │  │ clang AST   │  │ execution   │  │ outputs      │  │   widths      │  │ │
│  │  │             │  │ order map   │  │              │  │               │  │ │
│  │  └─────────────┘  └─────────────┘  └──────────────┘  └───────────────┘  │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                            │
│                                    ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                          PROGRAM MODEL                                    │ │
│  │                       (persisted to disk)                                 │ │
│  │                                                                           │ │
│  │  functions:    {name, params, constraints, calls, modifies, reads}        │ │
│  │  call_graph:   {node → [children], execution_order}                       │ │
│  │  data_flows:   [{source_fn, target_fn, variable, type}]                   │ │
│  │  constraints:  [{param, operator, value, source_line}]                    │ │
│  │  input_schema: {required_params, optional_params, valid_ranges}           │ │
│  │  output_schema:{files, formats, fields}                                   │ │
│  │  entry_points: [{name, required_inputs, produces}]                        │ │
│  │  error_states: [{condition, subroutine, line, message}]                   │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                            │
│                                    ▼                                            │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                       SEMANTIC EMBEDDER                                   │ │
│  │                                                                           │ │
│  │  Converts Program Model → searchable vector knowledge base                │ │
│  │  Model:   text-embedding-3-small  OR  local nomic-embed-text              │ │
│  │  Storage: SQLite vector store                                             │ │
│  │  Chunks:  one per function, constraint group, error state, data flow      │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    Mapping complete. Agent ready.
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     MULTI-AGENT ORCHESTRATION LAYER                             │
│                                                                                 │
│  MultiAgentOrchestrator                                                         │
│  classify_intent() → generate│execute│review│diagnose│validate│plan│explain    │
│                                    │                                            │
│       ┌────────────────────────────┼────────────────────────────┐              │
│       ▼                            ▼                            ▼              │
│  ┌──────────────┐         ┌─────────────────┐         ┌──────────────────┐     │
│  │   PLANNER    │         │     WRITER      │         │    VALIDATOR     │     │
│  │              │         │                 │         │                  │     │
│  │ Tools:       │         │ Tools:          │         │ Tools:           │     │
│  │ search_      │         │ All KB tools    │         │ read_file        │     │
│  │  program_    │         │ create_workspace│         │ list_workspace   │     │
│  │  model       │         │ write_file      │         │ parse_output     │     │
│  │ get_entry_   │         │ download_data   │         │ search_program_  │     │
│  │  points      │         │ generate_input  │         │   model          │     │
│  │ get_workflow │         │ recall_facts    │         │ check_constraints│     │
│  │ recall_facts │         │ store_fact      │         │ recall_facts     │     │
│  │              │         │                 │         │                  │     │
│  │ 4 steps max  │         │ 8 steps max     │         │ 4 steps max      │     │
│  └──────────────┘         └─────────────────┘         └──────────────────┘     │
│                                                                                 │
│       ┌────────────────────────────┬────────────────────────────┐              │
│       ▼                            ▼                            ▼              │
│  ┌──────────────┐         ┌─────────────────┐         ┌──────────────────┐     │
│  │   REVIEWER   │         │  DIAGNOSTICIAN  │         │  SOURCE ANALYST  │     │
│  │              │         │                 │         │  (new agent)     │     │
│  │ Tools:       │         │ Tools:          │         │                  │     │
│  │ execute_     │         │ diagnose_error  │         │ Tools:           │     │
│  │  software    │         │ trace_call_     │         │ search_source    │     │
│  │ read_file    │         │   graph         │         │ get_function     │     │
│  │ parse_output │         │ find_error_     │         │ get_constraints  │     │
│  │ interpret_   │         │   state         │         │ explain_why      │     │
│  │  results     │         │ record_lesson   │         │ trace_data_flow  │     │
│  │              │         │ retry           │         │                  │     │
│  │ 8 steps max  │         │ 5 steps max     │         │ 4 steps max      │     │
│  └──────────────┘         └─────────────────┘         └──────────────────┘     │
│                                                                                 │
│  Routing:                                                                       │
│  generate  → Planner → Writer → Validator → Reviewer → [Diagnostician]         │
│  diagnose  → Source Analyst → Diagnostician → Writer → Reviewer                │
│  explain   → Source Analyst only                                                │
│  execute   → Reviewer only                                                      │
│  validate  → Validator only                                                     │
│  plan      → Planner only                                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CORE REACT AGENT LOOP                                 │
│                                                                                 │
│  AgentLoop.run(user_message, conversation_id, llm_fn, system_prompt)           │
│                                                                                 │
│  BUILD CONTEXT                                                                  │
│  system_prompt + memory_block + prior_messages + program_model_summary         │
│          │                                                                      │
│          ▼                                                                      │
│  INJECT LESSONS                                                                 │
│  query run_history.db for past lessons relevant to this software + task        │
│  prepend to system prompt                                                       │
│          │                                                                      │
│          ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                     REACT LOOP  (max 8 steps)                           │   │
│  │                                                                         │   │
│  │   LLM CALL                                                              │   │
│  │   messages → OpenAI / local model with full tool schemas                │   │
│  │        │                                                                │   │
│  │        ├── Text response? ──► Done. Return reply.                       │   │
│  │        │                                                                │   │
│  │        └── Tool calls?    ──► DISPATCH TOOLS                           │   │
│  │                                    │                                   │   │
│  │                                    ▼                                   │   │
│  │                             Execute tool(s)                            │   │
│  │                             Append results to messages                 │   │
│  │                             Emit step event via SSE                    │   │
│  │                             Loop back to LLM CALL                      │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│          │                                                                      │
│          ▼                                                                      │
│  FAILURE HANDLER                                                                │
│  diagnose_error() → inject diagnosis → retry up to MAX_RETRIES=3               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               TOOL LAYER                                        │
│                                                                                 │
│  ┌───────────────────┐  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │ PROGRAM MODEL     │  │  KNOWLEDGE BASE  │  │      FILESYSTEM TOOLS        │ │
│  │ TOOLS             │  │  TOOLS           │  │                              │ │
│  │                   │  │                  │  │  create_workspace()          │ │
│  │ search_program_   │  │  find_recipe()   │  │  write_file()                │ │
│  │   model(query)    │  │  validate_config │  │  read_file()                 │ │
│  │                   │  │  generate_input  │  │  list_workspace()            │ │
│  │ get_function(name)│  │  build_config    │  │  copy_to_workspace()         │ │
│  │                   │  │  get_workflow    │  │  cleanup_workspace()         │ │
│  │ get_constraints(  │  │  lookup_keyword  │  │                              │ │
│  │   param)          │  │  search_kb       │  └──────────────────────────────┘ │
│  │                   │  │  get_schema      │                                   │
│  │ trace_call_graph( │  │  get_dependencies│  ┌──────────────────────────────┐ │
│  │   function)       │  │                  │  │      EXECUTION TOOLS         │ │
│  │                   │  └──────────────────┘  │                              │ │
│  │ trace_data_flow(  │                         │  execute_software(           │ │
│  │   variable)       │  ┌──────────────────┐  │    software,                 │ │
│  │                   │  │  DATA SOURCE     │  │    workspace,                │ │
│  │ explain_error(    │  │  TOOLS           │  │    entry_point,              │ │
│  │   error_msg)      │  │                  │  │    timeout)                  │ │
│  │                   │  │ download_data(   │  │                              │ │
│  │ get_entry_points()│  │   software,      │  │  check_run_status()          │ │
│  │                   │  │   data_type,     │  │  get_run_log()               │ │
│  └───────────────────┘  │   location)      │  │                              │ │
│                         │                  │  └──────────────────────────────┘ │
│                         │ geocode_         │                                   │
│                         │   location()     │  ┌──────────────────────────────┐ │
│                         │                  │  │       PARSER TOOLS           │ │
│                         │ [plugin-specific │  │                              │ │
│                         │  data sources]   │  │  parse_output(               │ │
│                         └──────────────────┘  │    software, workspace)      │ │
│                                               │                              │ │
│                                               │  interpret_results(          │ │
│                                               │    software, output)         │ │
│                                               │                              │ │
│                                               │  summarize_output(           │ │
│                                               │    software, output)         │ │
│                                               └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PLUGIN LAYER                                       │
│                                                                                 │
│  SoftwarePlugin interface — every plugin implements:                            │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │  name()            → str                                                  │ │
│  │  description()     → str                                                  │ │
│  │  get_tools()       → list[ToolDefinition]                                 │ │
│  │  get_validators()  → list[Validator]                                      │ │
│  │  download_data()   → DataBundle                                           │ │
│  │  get_executable()  → str                                                  │ │
│  │  generate_input()  → str                                                  │ │
│  │  parse_output()    → OutputModel                                          │ │
│  │  validate_config() → list[Issue]                                          │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│  │  AERMOD PLUGIN  │  │  SWMM PLUGIN    │  │  AUTOCAD PLUGIN │                │
│  │                 │  │                 │  │                 │                │
│  │ source:         │  │ source:         │  │ source:         │                │
│  │ aermod.f        │  │ swmm5.c         │  │ ObjectARX SDK   │                │
│  │ aermet.f        │  │ (public)        │  │ headers         │                │
│  │ aermap.f        │  │                 │  │                 │                │
│  │ (public EPA)    │  │ data_sources:   │  │ execution:      │                │
│  │                 │  │ NOAA rainfall   │  │ win32com API    │                │
│  │ data_sources:   │  │ USGS streamflow │  │ (COM automation)│                │
│  │ Ontario MECP    │  │ NHD catchments  │  │                 │                │
│  │ NOAA NCEI       │  │                 │  │ data_sources:   │                │
│  │ ECCC            │  │ execution:      │  │ DXF files       │                │
│  │                 │  │ swmm5.exe       │  │ survey data     │                │
│  │ execution:      │  │                 │  │                 │                │
│  │ aermod.exe      │  │ parsers:        │  │ parsers:        │                │
│  │ aermet.exe      │  │ .rpt file       │  │ .dxf reader     │                │
│  │ aermap.exe      │  │ .out file       │  │ object model    │                │
│  │ Bpipprm.exe     │  │                 │  │                 │                │
│  │ aerplot.exe     │  └─────────────────┘  └─────────────────┘                │
│  └─────────────────┘                                                           │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│  │ HEC-RAS PLUGIN  │  │ MODFLOW PLUGIN  │  │ CALPUFF PLUGIN  │                │
│  │                 │  │                 │  │                 │                │
│  │ source:         │  │ source:         │  │ source:         │                │
│  │ HEC-RAS HDF5    │  │ MODFLOW 6       │  │ calpuff.f       │                │
│  │ SDK             │  │ (public USGS)   │  │ (public EPA)    │                │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           EXECUTION LAYER                                       │
│                                                                                 │
│  ExecutionRouter — selects strategy based on software type                      │
│                                                                                 │
│  TIER 1 — NATIVE API (preferred)                                                │
│  Software has Python / COM / REST API                                           │
│    autocad  → win32com.client.Dispatch("AutoCAD.Application")                  │
│    revit    → Revit API via pyRevit                                             │
│    excel    → win32com.client.Dispatch("Excel.Application")                    │
│                                                                                 │
│  TIER 2 — SCRIPT ENGINE                                                         │
│  Software accepts script or command file                                        │
│    aermod   → subprocess.run([aermod.exe], cwd=workspace)                      │
│    autocad  → .scr script file sent via command line                           │
│    swmm     → subprocess.run([swmm5.exe, inp, rpt, out])                       │
│                                                                                 │
│  TIER 3 — UI CONTROLLER (fallback — no API, no script)                         │
│    screenshot → vision model → identify UI elements                            │
│    → pyautogui click/type → verify state change                                │
│    → screenshot again → confirm result                                         │
│                                                                                 │
│  StateMonitor (runs continuously during execution)                              │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │  poll_state()             → {active_command, selection, errors, done}     │ │
│  │  wait_for_completion()    timeout=600                                     │ │
│  │  detect_error_state()     → ErrorState | None                             │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          PERSISTENCE LAYER                                      │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐   │
│  │   memory.db     │  │ run_history.db  │  │      vector_store.db         │   │
│  │                 │  │                 │  │                              │   │
│  │ Per-conversation│  │ All past runs   │  │ Program model embeddings     │   │
│  │ key-value facts │  │ Per-step status │  │ Knowledge base chunks        │   │
│  │                 │  │ Audit logs      │  │ Lesson embeddings            │   │
│  │ source_type     │  │ Learned lessons │  │                              │   │
│  │ coordinates     │  │ Error patterns  │  │ Searchable by natural        │   │
│  │ met_files       │  │ Success rates   │  │ language query               │   │
│  │ workspace       │  │                 │  │                              │   │
│  │                 │  │ Indexed per     │  │ Cosine similarity search     │   │
│  │ SQLite WAL      │  │ software plugin │  │ SQLite blob storage          │   │
│  └─────────────────┘  └─────────────────┘  └──────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐   │
│  │ program_models/ │  │ config_recipes/ │  │       workspaces/            │   │
│  │                 │  │                 │  │                              │   │
│  │ aermod_22112/   │  │ aermod.db       │  │ run_20260316_180320/         │   │
│  │   model.json    │  │ swmm.db         │  │   aermod.inp                 │   │
│  │   vectors.db    │  │ hec_ras.db      │  │   aermod.out                 │   │
│  │   call_graph    │  │                 │  │   Toronto.SFC                │   │
│  │                 │  │ 3,600+ recipes  │  │   Toronto.PFL                │   │
│  │ swmm_5.1/       │  │ per software    │  │   AERPLOT.kmz                │   │
│  │   model.json    │  │                 │  │                              │   │
│  │   vectors.db    │  │ Tag-scored      │  │ 7-day TTL auto-cleanup       │   │
│  └─────────────────┘  └─────────────────┘  └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              LLM LAYER                                          │
│                                                                                 │
│  LLMClient — abstracted, swappable at config level                              │
│                                                                                 │
│  ┌──────────────────────────┐         ┌──────────────────────────┐             │
│  │   CLOUD (development)    │         │   LOCAL (production)     │             │
│  │                          │   OR    │                          │             │
│  │   OpenAI API             │         │   Ollama                 │             │
│  │   gpt-4o-mini            │         │   llama3.1:70b           │             │
│  │   text-embedding-3-small │         │   qwen2.5:72b            │             │
│  │                          │         │   nomic-embed-text       │             │
│  │   Pros: easy setup       │         │                          │             │
│  │   Cons: data leaves      │         │   Pros: private data     │             │
│  │         network          │         │   zero API cost          │             │
│  │         API cost at scale│         │   on-premise deploy      │             │
│  └──────────────────────────┘         └──────────────────────────┘             │
│                                                                                 │
│  Swap: one config change  OPENAI_BASE_URL=http://localhost:11434                │
│  Tool schema format: OpenAI function-calling (identical for both)               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## File Structure

```
universal_agent/
│
├── app.py                          # Flask entry point
├── requirements.txt
├── .env                            # API keys, model selection
│
├── core/
│   ├── agent/
│   │   ├── react_loop.py           # Core ReAct loop (from aermod_pipeline)
│   │   ├── orchestrator.py         # Multi-agent coordination
│   │   ├── memory.py               # Persistent SQLite facts
│   │   ├── run_history.py          # Run history + lessons
│   │   ├── vector_store.py         # Embedding search
│   │   └── audit.py                # Per-run audit trail
│   │
│   ├── mapper/
│   │   ├── source_mapper.py        # Main mapper entry point
│   │   ├── sdk_mapper.py           # SDK / header mapping
│   │   ├── binary_analyzer.py      # Binary analysis fallback
│   │   ├── ast_parsers/
│   │   │   ├── fortran_parser.py   # fparser2-based
│   │   │   ├── python_parser.py    # stdlib ast module
│   │   │   └── cpp_parser.py       # clang AST bindings
│   │   ├── call_graph.py           # Call graph builder
│   │   ├── data_flow.py            # Data flow analyzer
│   │   ├── constraint_extractor.py # IF-block constraint mining
│   │   └── semantic_embedder.py    # Program model → vectors
│   │
│   ├── executor/
│   │   ├── router.py               # Selects API/script/UI strategy
│   │   ├── api_bridge.py           # COM / REST / Python API
│   │   ├── script_engine.py        # Script/command file execution
│   │   ├── ui_controller.py        # Computer vision fallback
│   │   └── state_monitor.py        # Software state polling
│   │
│   ├── tools/
│   │   ├── program_model_tools.py  # search_program_model, get_function, etc.
│   │   ├── filesystem_tools.py     # workspace management
│   │   ├── execution_tools.py      # execute_software, check_status
│   │   ├── parser_tools.py         # parse_output, interpret_results
│   │   ├── history_tools.py        # diagnose_error, record_lesson
│   │   └── memory_tools.py         # store_fact, recall_facts
│   │
│   └── plugin_base.py              # SoftwarePlugin abstract interface
│
├── plugins/
│   ├── aermod/                     # Existing aermod_pipeline code, refactored
│   │   ├── plugin.py
│   │   ├── knowledge_base/
│   │   ├── generators/
│   │   ├── validators/
│   │   ├── data_sources/
│   │   └── parsers/
│   │
│   ├── swmm/                       # Storm Water Management Model
│   ├── hec_ras/                    # River hydraulics / flood modelling
│   ├── modflow/                    # Groundwater flow
│   ├── calpuff/                    # Long-range air dispersion
│   ├── phreeqc/                    # Groundwater geochemistry
│   └── autocad/                    # CAD automation
│
├── data/
│   ├── memory.db
│   ├── run_history.db
│   ├── vector_store.db
│   ├── program_models/             # Cached program models per software version
│   │   ├── aermod_22112/
│   │   │   ├── model.json
│   │   │   ├── vectors.db
│   │   │   └── call_graph.json
│   │   └── swmm_5.1/
│   ├── config_recipes/             # Per-software recipe databases
│   │   ├── aermod.db
│   │   └── swmm.db
│   └── workspaces/                 # Run output directories (7-day TTL)
│
├── templates/
│   ├── index.html
│   └── chat.html
│
├── static/
│   ├── css/style.css
│   └── js/app.js
│
└── tests/
```

---

## Request Data Flow (End to End)

```
User: "Model a compressor station near Barrie, 15m stack, 2.5 g/s NO2"
        │
        ▼
POST /api/agent/run
Creates run_id, starts background thread
Returns {run_id, status: "running"} immediately
        │
        ▼ (background thread)
classify_intent() → "generate"
agent_sequence = [planner, writer]
        │
        ▼
Planner Agent (max 4 steps)
  Step 1: search_program_model("NO2 point source AERMOD")
          → returns PVMRM requirement, BETA constraints
  Step 2: get_workflow_steps("point_source_no2")
          → [BPIP, AERMOD, AERPLOT]
  Step 3: geocode_location("Barrie Ontario")
          → {lat: 44.39, lon: -79.69}
  Step 4: store_fact(conversation_id, "coordinates", ...)
        │
        ▼
Writer Agent (max 8 steps)
  Step 1: recall_facts()
          → retrieves coordinates from memory
  Step 2: download_ontario_met_to_workspace(44.39, -79.69, ws)
          → downloads real .SFC/.PFL from ontario.ca
  Step 3: find_recipe(["point_source", "no2", "pvmrm"])
          → returns best recipe from config_recipes.db
  Step 4: validate_config("AERMOD", config)
          → checked against program model constraints
  Step 5: generate_input_file("AERMOD", config)
          → aermod.inp content returned
  Step 6: write_file(workspace, "aermod.inp", content)
  Step 7: generate_input_file("AERPLOT", config)
  Step 8: write_file(workspace, "aerplot.inp", content)
        │
        ▼
Validator Agent — pre-flight check against program model
        │
        ▼
Reviewer Agent (max 8 steps)
  Step 1: execute_software("AERMOD", workspace_path)
          → subprocess.run(aermod.exe, cwd=workspace)
  Step 2: parse_output("AERMOD", workspace)
          → reads aermod.out, extracts concentrations
  Step 3: execute_software("AERPLOT", workspace_path)
          → AERPLOT.kmz written
  Step 4: interpret_results("AERMOD", parsed_output)
          → plain English compliance summary
        │
        ▼
SSE "done" event:
{
  reply: "Peak 1-hr NO2: 142 μg/m³ at 340m NE. 75% of standard. Compliant.",
  files_generated: ["aermod.inp", "aermod.out", "Toronto.SFC", "AERPLOT.kmz"],
  workspace: "barrie_no2_20260316_143022",
  agents_used: ["planner", "writer", "validator", "reviewer"],
  duration_ms: 187000
}
```

---

## What This Is vs. What Exists Now

```
aermod_pipeline (current)          universal_agent (this design)
──────────────────────────         ──────────────────────────────
Knowledge base: hand-crafted  →    Knowledge base: derived from source code
One software: AERMOD          →    Any software via plugin
Ontario data only             →    Any data source via plugin
Fixed tool set                →    Dynamically generated from program model
Manual constraint rules       →    Constraints extracted from code automatically
Single domain                 →    Universal
```

Current project is ~70% of this architecture.
Missing 30%: mapping engine + plugin abstraction layer.

---

## Target Software for Plugins

| Software | Domain | Source Available |
|---|---|---|
| AERMOD | Air dispersion | Yes — public EPA |
| CALPUFF | Long-range dispersion | Yes — public EPA |
| SWMM | Stormwater | Yes — public EPA |
| HEC-RAS | River hydraulics / flood | Yes — public USACE |
| MODFLOW | Groundwater flow | Yes — public USGS |
| PHREEQC | Geochemistry | Yes — public USGS |
| HELP Model | Landfill hydrology | Yes — public EPA |
| AutoCAD | CAD | SDK / headers |
| Revit | BIM | SDK / headers |
| SCREEN3 | Screening dispersion | Yes — public EPA |

---

## Key Design Decisions

**1. Program model is cached per version**
Mapping runs once. Results are stored in `data/program_models/{software}_{version}/`.
Subsequent startups load from cache in milliseconds.
Re-maps automatically when a new executable version is detected.

**2. Plugin interface is minimal**
Each plugin implements 9 methods. Everything else — agent loop, memory,
streaming, lessons, orchestration — is handled by the core framework.

**3. LLM is fully swappable**
One environment variable change swaps OpenAI for a local Ollama model.
Critical for on-premise deployment to government and regulated industry clients.

**4. Execution strategy is automatic**
The router checks for API availability first, falls back to script,
falls back to UI controller. Plugin author does not need to choose.

**5. Source Analyst is a new agent role**
Not in aermod_pipeline. Exists specifically to query the program model
when users ask "why" questions or when the Diagnostician needs to trace
an error back to source code.

---

## Build Order

```
Phase 1 — Extract and generalize (aermod_pipeline → universal_agent)
  1. Extract core framework from aermod_pipeline
  2. Refactor AERMOD code into plugin structure
  3. Confirm everything still works

Phase 2 — Build mapping engine
  4. Fortran AST parser (fparser2)
  5. Constraint extractor (IF-block mining)
  6. Call graph builder
  7. Semantic embedder
  8. Map AERMOD source as proof of concept

Phase 3 — Second plugin
  9. CALPUFF plugin (same customers, complementary to AERMOD)
  10. Confirm framework handles two plugins correctly

Phase 4 — Expand
  11. Each additional plugin gets faster
  12. SWMM, HEC-RAS, MODFLOW as environmental suite
  13. AutoCAD for non-environmental markets

Phase 5 — UI controller (last)
  14. Only add computer vision fallback when needed
  15. Requires most maintenance — prioritize API paths
```

---

## Commercial Expansion Path

```
aermod_pipeline (today)
  └── Single software, Ontario only, ~$0 commercial value

universal_agent + AERMOD + CALPUFF (Phase 3)
  └── Full air quality suite, all geographies
  └── Target: environmental consulting firms
  └── Realistic ARR: $500K–$2M

+ SWMM + HEC-RAS + MODFLOW (Phase 4)
  └── Full environmental engineering suite
  └── Target: municipal engineering, hydrogeology
  └── Realistic ARR: $3M–$8M

+ AutoCAD + Revit (Phase 4)
  └── Breaks out of environmental niche
  └── Target: any engineering firm
  └── Realistic ARR: $10M–$30M

+ Permit renewal SaaS layer
  └── Recurring subscription on top of tool access
  └── High retention, predictable revenue
  └── Realistic ARR: $20M–$60M
```

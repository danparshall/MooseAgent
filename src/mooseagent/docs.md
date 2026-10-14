# Noridoc: mooseagent

Path: @/src/mooseagent

### Overview

This is the Python package for the whole system. It holds two LangGraph workflows: the **MooseAgent** pipeline, which turns a natural-language simulation requirement into one or more MOOSE input cards (`.i` files), runs them, and repairs them; and the **AutoComment** pipeline, which annotates uncommented MOOSE example inputs to build the RAG corpus. The live pipeline is `graph.py`. `graph_ablation.py` and `graph_old.py` are alternate or older copies of it, and the rest of the folder is shared infrastructure: configuration, state schemas, prompts, model loading, retrieval, and error analysis.

### How it fits into the larger codebase

- **No package install; imports go through `RUN_PATH`.** Every entry script calls `load_dotenv()`, reads `RUN_PATH` from `.env`, and does `sys.path.append(run_path)` before `from mooseagent... import`. `RUN_PATH` therefore has to point at `@/src`, the directory that contains `mooseagent/`. The scripts also treat `RUN_PATH` as the base for `database/dp.json` and `log/<timestamp>.log`. `@/pyproject.toml` and `@/langgraph.json` still describe the LangGraph `react_agent` starter template (`./src/react_agent/graph.py:graph`, which does not exist), so neither `pip install` nor `langgraph dev` reaches this package.
- **Reads `@/src/database`** (see `@/src/database/docs.md`): the FAISS/Chroma vector stores (`*_inpcard`, `*_dp`) and `dp.json`, a dictionary from MOOSE object name to parameter dump. The graph's `__main__` loads `dp.json` into graph state, and AutoComment uses it for app lookup. `load_vector_database.py` and `autocomment.py` are the only writers to that folder.
- **Calls out to MOOSE** through `subprocess.run`, wrapped by `launcher.sh` (conda env activation). It writes generated cards to `Configuration.save_dir`, which lives outside the repo.
- **External services:** the LLM providers in `utils.load_chat_model`, and either a local HuggingFace `BAAI/bge-m3` model or a remote embedding API.
- `@/tests` does not exercise this package (see `@/tests/docs.md`).

### Core Implementation

**Configuration (`configuration.py`).** `Configuration` is a `kw_only` dataclass. `from_runnable_config(config)` builds each field from `os.environ[FIELD_NAME.upper()]` first, then `config["configurable"][field]`, then the class default. Any value that is falsy after this lookup is dropped (`if v`), so the default wins.

| Role (field) | Default | Used by |
|---|---|---|
| `alignment_model` | `huoshan/deepseek-v3-241226` | requirement alignment; also the plain-LLM "helper" in `graph_ablation.modify` |
| `architect_model` | `huoshan/deepseek-r1-250120` | drafts each card (structured output `InpcardContentState`) |
| `assistant_model` | `huoshan/deepseek-v3-241226` | tool-calling RAG helper subgraph used by `modify` |
| `query_model` | `huoshan/deepseek-v3-241226` | enhanced-retrieval question generation |
| `rag_model` | `openai/gpt-4o-mini` | enhanced-retrieval summarization |
| `rearchitect_model` | `huoshan/deepseek-v3-241226` | repeated-error judge (`RearchitechState`) |
| `extracter_model` | `openai/gpt-4o-mini` | turns free-text LLM replies into Pydantic objects (`ExtracterFileState`, `ModifyState`) |
| `review_model`, `writer_model` | deepseek-v3 | `graph_old.py` only |
| `comment_rag_model`, `comment_writer_model` | `openai/gpt-4o-mini` | `autocomment.py` |
| `embedding_function` | `BGE_M3_EmbeddingFunction` (alternative `"OPENAI"`) | all vector-store loads |

Other knobs: `MAX_ITER=7` (modify rounds per architecture attempt), `MAX_REARCHITECT=3`, `mpi=1`, `top_k=3`, `vector_store="faiss"`, `enable_human_intervention=True`, `enable_alignment_feedback=False`, and `error_threshold_for_intervention=3`. The absolute paths (`ABSOLUTE_PATH`, `MOOSE_DIR`, `save_dir`) are hardcoded to the author's machine under `/home/zt/...`. The store paths are computed as `ABSOLUTE_PATH/database/{embedding_function}_{vector_store}_{inpcard|dp}`.

**Model dispatch (`utils.load_chat_model`).** The name is split on the first `/` into `provider/model`:

| Prefix | Client | Env vars |
|---|---|---|
| `huoshan/` | `ChatDeepSeek(api_base=HUOSHAN_API_BASE, api_key=HUOSHAN_API_KEY)` (Volcengine Ark) | `HUOSHAN_*` |
| `siliconflow/` | `ChatOpenAI(base_url=SILICONFLOW_API_BASE, ...)` | `SILICONFLOW_*` |
| anything else (e.g. `openai/`) | `langchain.chat_models.init_chat_model(model, model_provider=provider)` | provider-standard (`OPENAI_API_KEY`, optional `OPENAI_API_BASE`) |

The default temperature is 0.01. `Configuration.TEMPERATURE` is not used anywhere.

**State (`state.py`).** `FlowState` is a plain `TypedDict` with no reducers, so every node update overwrites the key.

| Key | Meaning |
|---|---|
| `requirement` | user's raw request |
| `feedback` | alignment feedback |
| `file_list` | `list[FileState(file_name, description)]`; the **first entry is the main card that gets executed** |
| `dp_json` | the entire `dp.json` dict, carried in state and therefore checkpointed by `MemorySaver` every step |
| `run_result` | list of MOOSE stderr strings; set to the string `"success"` on success |
| `reason` | modify explanations; written but never read by any node |
| `review_count`, `rearchitect_count` | loop counters |
| `history_error` | the rearchitect judge's diagnosis, fed to the next architect pass |
| `error_patterns`, `pending_human_intervention`, `human_intervention_feedback` | human-in-the-loop bookkeeping |

The Pydantic models in `state.py` are the `with_structured_output` schemas. `OneFileState`, `ReviewOneFileState`, `ExtracterArchitectState`, and `InpcardState` are used only by `graph_old.py` or not at all.

**The live graph (`graph.py`).** The graph is a single `StateGraph(FlowState)` compiled with a `MemorySaver` checkpointer. There are no subgraphs and no `Send` fan-out. Per-card parallelism is `asyncio.gather` inside one node. Routing out of `human`, `run_inpcard`, and `human_intervention` happens only through returned `Command(goto=...)`, with no `add_conditional_edges`, so a rendered graph (`graph.ipynb`) does not show those edges.

```
START -> align_simulation_description -> human
human --Command--> architect                       (feedback == "yes")
      --Command--> align_simulation_description    (any other string; sets state.feedback)
architect -> run_inpcard            (static edge)
modify    -> run_inpcard            (static edge)
human_intervention -> modify        (static edge + Command goto modify)

run_inpcard --Command-->
  "End"                (stderr == ""  -> run_result="success")
  human_intervention   (some error signature seen >= error_threshold_for_intervention times)
  modify               (review_count < MAX_ITER)
  architect            (rearchitect_count < MAX_REARCHITECT and judge says "True";
                        resets review_count/run_result/reason/error_patterns, sets history_error)
  modify               (judge says not repeating; review_count decremented by 1)
  "End"                (rearchitect budget exhausted, or MOOSE_DIR path missing)
```

| Node | What it does |
|---|---|
| `align_simulation_description` | `alignment_model` with `SYSTEM_ALIGNMENT_PROMPT + MultiAPP_PROMPT` and `HUMAN_ALIGNMENT_PROMPT` expands the requirement into a per-file plan. A second call to `extracter_model` parses that free text into `ExtracterFileState.file_list`. |
| `human` | If `enable_alignment_feedback`, calls `interrupt(...)` with the formatted file list. Otherwise it auto-approves with `"yes"`. |
| `architect` (`architect_all`) | Increments `rearchitect_count`, then runs `architect_input_card` for every file concurrently. Each one builds a fresh `EnhancedRetrievalSystem`, retrieves, calls `architect_model`, and **writes the card straight to `save_dir/<file_name>`**. Card contents never live in graph state; the filesystem is the source of truth. |
| `run_inpcard` | Runs `[launcher.sh, "mpiexec", "-n", mpi, MOOSE_DIR, "-i", save_dir/file_list[0].file_name]` with `capture_output=True`. **Success is defined as empty stderr**; the return code is not checked. On failure it appends stderr to `run_result`, recomputes `error_patterns` over the whole `run_result` list with `ErrorAnalyzer`, and routes as above. |
| `modify` | Increments `review_count`. Re-reads every card from disk and concatenates them with their descriptions. Appends `check_app(...)` output to the latest error; this lists every `type = X` whose `X` is not a key in `dp_json`. Fills `MODIFY_PROMPT` (with optional human feedback) and sends it to the **tool-calling helper subgraph**. `extracter_model` then parses the reply into `ModifyState(filename, error, code)`, and `code` overwrites `save_dir/<filename>`. Only one file is rewritten per round. |
| `human_intervention` | If `enable_human_intervention` is off, it goes straight to `modify`. Otherwise it calls `interrupt(...)` with the error-pattern summary and stores the resume value in `human_intervention_feedback`, which `modify` injects into `MODIFY_PROMPT`. |

**RAG helper subgraph (`helper.py`).** At import time, the module loads both vector stores (FAISS by default, or Chroma) with the configured embedding and wraps them as retrievers with `k=top_k`. It exposes them as the LangChain retriever tools `retrieve_moose_case` (the `*_inpcard` store of annotated example inputs) and `retrieve_moose_dp` (the `*_dp` store of per-object documentation). `bulid_helper(model)` compiles a small ReAct loop, `helper -> tools_condition -> ToolNode -> helper`, in which the LLM decides when to query. `graph.py` and `graph_ablation.py` build it once at import using `assistant_model`. In the live graph, only `modify` uses this path.

**Enhanced retrieval (`enhanced_retrieval.py`).** Each architect call runs `enhanced_retrieve(description)`:

1. `query_model` is prompted with `ENHANCED_RETRIEVAL_QUERY_PROMPT` to produce a numbered list of questions. The list is parsed with regexes (numbered, then `Question N:`, then `- ` bullets, then a raw-line fallback), keeps at most 7 questions, and assigns each a keyword-based "category" and a position-based "priority".
2. For every question in parallel, it queries both retrievers (`k=top_k` each) and joins the `page_content`s. A "relevance score" of `min(1, len/1000)` is assigned, and results at or below 0.1 are dropped.
3. `rag_model` with `CONTENT_REFINEMENT_PROMPT` summarizes the question, first 3 key points, and the first 500 chars of content for each result. If that call fails, it falls back to concatenating key points.

The architect then wraps the summary as `similar_cases`, plus `MultiAPP_PROMPT` when there are multiple files, and passes it as `cases=` to `SYSTEM_ARCHITECT_PROMPT.format(...)`.

**Error analysis (`human_intervention.py`).** `ErrorAnalyzer.extract_error_signature` normalizes numbers and `/paths`, then returns the first regex match among `...Error`, `... Exception`, `... Failed`, `... Not Found`, `undefined X`, `invalid X`, and `missing X`, lowercased. If nothing matches, it returns the first 50 chars. `needs_human_intervention` fires when any signature count reaches the threshold. `HumanInterventionHandler` formats the interrupt messages. `parse_human_feedback` extracts verb-object "suggestions" with regexes; the only effect of a non-empty result is that `state["feedback"]` gets set, and no live node reads that key after alignment.

**Embeddings (`utils.BGE_M3_EmbeddingFunction`).** The default tries to load `BAAI/bge-m3` locally through `transformers`/`torch`, using CUDA if available and **CLS pooling** on one text at a time. If that fails, a bare `except` falls back to a remote OpenAI-compatible `/embeddings` endpoint (`BGE_M3_API_BASE`, `BGE_M3_API_KEY`, `EMBEDDING_MODEL`). `torch` and `transformers` are not in `@/requirements.txt`.

**AutoComment (`autocomment.py`).** This is a separate `StateGraph(InputCardWorkflowState)`: `START -> input_card_selector -> (route_comment) -> rag -> writer -> save_to_json -> input_card_selector`, ending when the path list is empty or `num_commented >= max_commented` (400 in `__main__`).

- The selector randomly picks paths from `case_name_uncommented.txt` until it finds a file with more than 10 lines.
- `rag` collects `type = X` names, either by string parsing or, if `use_llm_rag`, through an LLM, and concatenates the matching `dp.json` entries.
- `writer` (`comment_writer_model`) returns `CommentState(overall_description, annotated_input_card)`.
- `save_to_json` appends to `comment.json`, writes `<COMMENT_PATH minus .json>/<name>`, i.e. a `database/comment/` directory, and every `save_every=3` cards rewrites the remaining-uncommented list.

`autorun.py` restarts `autocomment.py` in a loop until it exits 0.

**Index building (`load_vector_database.py`).** This is a script; all work happens at import. It loads `Configuration.rag_json_path` with `JSONLoader(jq_schema=".[]", text_content=False)`, so each document's `page_content` is the JSON-serialized record. It dedupes by SHA-256 of `page_content` against `hashes.json` in `PERSIST_DIRECTORY` (or recomputes hashes from the existing store) and adds documents in `batch_size` batches. Then it calls `save_local` (FAISS) or `persist` (Chroma) and rewrites `hashes.json`. Which corpus goes into which store is controlled only by editing `rag_json_path` and `PERSIST_DIRECTORY`.

**Experiments (`statistics.py`).** It imports `architect_builder` from **`graph_ablation`**, not `graph`. It runs a topic `n_runs` times, recompiling without a checkpointer each time, moves `save_dir` into `RUN_PATH/experiments/<case>_<ts>/run_i`, and writes `stats.csv` with success rate, mean tokens (from `get_openai_callback`), code length, and `review_count`. `pandas` is not in `@/requirements.txt`.

### Things to Know

**Live entry point.** The only working way to run the agent is `python src/mooseagent/graph.py`. Its `__main__` hardcodes a phase-field solidification `topic`, redirects stdout through `utils.Logger` to `RUN_PATH/log/<ts>.log` (the `log/` dir must already exist), runs `rm -r save_dir` first, calls `graph.ainvoke` with `thread_id="1"` and `recursion_limit=10000`, and prints the total generated code length and the token counts. The `__main__` blocks never resume an `interrupt()`. When `human_intervention` fires, which is enabled by default, `ainvoke` returns at the interrupt and the script proceeds to its summary printout.

**Variant graphs.**

| | `graph.py` (live) | `graph_ablation.py` | `graph_old.py` |
|---|---|---|---|
| Retrieval for architect | enhanced multi-query | none (`similar_cases = ""`; the query LLM is still called and its output is discarded) | single `retriever_input.ainvoke(description)` |
| `modify` LLM | tool-calling RAG helper | plain `alignment_model`, no tools | per-file `writer_model` + helper; a separate `review_inpcard` node picks the failing file |
| Human-in-the-loop | alignment interrupt (off) + repeated-error interrupt (on) | none (`human` hardcodes "yes") | none |
| Routing | `Command` from `run_inpcard` | same as live, minus intervention | `add_conditional_edges(run_inpcard, route_run_inpcard)` → `review_inpcard -> modify -> check_onefile -> modify\|run_inpcard`; architecting happens inside `human`; no rearchitect |
| Runs MOOSE through `launcher.sh` | yes | yes | no (bare `mpiexec`) |

Neither variant runs against the current shared modules:

- `graph_ablation.modify` calls `MODIFY_PROMPT.format(...)` without the `human_feedback*` keys that the current prompt requires, which raises `KeyError`. Those keys were added in the later "human feedback" commit, and `statistics.py` depends on this file.
- `graph_old.py` calls `bulid_helper()` with no argument at import.
- `graph_old.py` also formats `SYSTEM_ARCHITECT_PROMPT` without `history_error`.
- `graph_old.py` uses nested same-quote f-strings, which require Python 3.12 or later.

**Retrieved context does not reach the architect prompt.** `SYSTEM_ARCHITECT_PROMPT` has no `{cases}` placeholder; it was removed in commit `f3b938b` ("architect is also helper"). `str.format` silently ignores the extra `cases=` kwarg, so in the live graph the enhanced-retrieval summary and the appended `MultiAPP_PROMPT` are computed but never shown to `architect_model`. Retrieval only affects the output through the tool-calling helper in `modify`. `MultiAPP_PROMPT` does still reach the alignment step.

**Termination uses `goto="End"`.** `"End"` is not a registered node; LangGraph's sentinel is `"__end__"`. Under the pinned `langgraph==0.3.11`, a `Command` to an unknown name triggers nothing, so the run simply stops. This was verified with a minimal probe graph. It is not a real edge to `END`.

**Loop accounting.**

- `review_count` resets only on rearchitect.
- The "not repeating" verdict decrements `review_count` by one, which grants exactly one more modify round per judge call. The judge can be called again on each subsequent failure until it says "True" (at most `MAX_REARCHITECT` architect passes in total, counting the first).
- The judge sees only `run_result[-5:]`, which is MOOSE stderr plus any `check_app` notes that `modify` appended; `reason` is not passed.
- `error_patterns` are recomputed over the full `run_result` history and are not cleared after a human intervention. Once a signature reaches the threshold, every later failure in that architecture pass routes back to `human_intervention` before the `MAX_ITER` check.
- `modify` mutates `state["run_result"][-1]` in place to append the `check_app` feedback.

**Config override gotchas.**

- Derived paths (`docs_dir`, `rag_json_path`, `PERSIST_DIRECTORY`, `dp_database_path`, `input_database_path`, `dp_json_path`) are evaluated once at class-definition time from the default `ABSOLUTE_PATH` and `embedding_function`. Overriding those two via env or config does not move the derived paths.
- `lanucher_script` (sic) and `MAX_REARCHITECT` have no type annotation, so they are class attributes rather than dataclass fields and cannot be overridden at all.
- Env-var overrides arrive as strings, e.g. `MAX_ITER="7"`, and are never cast.
- The falsy-drop rule means `False` and `0` passed through `configurable` are ignored. The string `"False"` from env is truthy.

**Portability.**

- Machine-specific absolute paths appear in `configuration.py` (`/home/zt/...`), `launcher.sh` (sources `/home/zt/miniforge/etc/profile.d/conda.sh`, then `conda activate moose`), `autocomment.py` and `autorun.py` (`E:/vscode/...` Windows paths), and in `adjust_path`'s hardcoded Windows prefixes.
- With `vector_store="chroma"`, the computed directory is `..._chroma_...` in lowercase, while the committed dirs are `..._Chroma_...`.
- `EnhancedRetrievalSystem` is constructed per card per architect pass, and each construction reloads both stores and re-instantiates the BGE-M3 model.

**Dead code.**

- `tools.py` is the template's Tavily search tool. It is not imported anywhere and references a nonexistent `configuration.max_search_results`.
- `helper.py`'s `__main__` calls `bulid_helper()` without its required argument.
- `utils.extract_files_and_descriptions` and `extract_sub_tasks` are unused, as are `combine_code_with_description` in the graphs and several prompts (`SYSTEM_REVIEW_PROMPT`, `SYSTEM_HELPER_PROMPT`, `SUGGESTION_PROMPT`).
- `graph.ipynb` only renders mermaid diagrams of `graph` and the AutoComment `workflow`.

**README mismatches.** The README lists `review_writer_model` and `writer_model`, but the live code uses `assistant_model`, `rearchitect_model`, `query_model`, and `rag_model`. The README says annotated cards go to `database/comments/`, but the code writes to `database/comment/`. The README says `python graph.py` is the entry; that is true, but it has to be run so that `RUN_PATH` is set.

Created and maintained by Nori.

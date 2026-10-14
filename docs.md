# Noridoc: MooseAgent (repository root)

Path: @/

### Overview

MooseAgent is a LangGraph multi-agent system that writes, runs, and repairs input cards (`.i` files) for MOOSE, Idaho National Laboratory's Multiphysics Object-Oriented Simulation Environment, starting from a natural-language simulation requirement. A second LangGraph workflow, AutoComment, annotates MOOSE's own example inputs with an LLM to build the "similar cases" RAG corpus. The repo was bootstrapped from the LangGraph `react_agent` starter template. Much of the root-level tooling (`pyproject.toml`, `langgraph.json`, `Makefile`, CI, `tests/`) still describes that template rather than this code.

### How it fits into the larger codebase

```
 .env (RUN_PATH, API keys)                         MOOSE build (Configuration.MOOSE_DIR)
        |                                                    ^
        v                                                    | launcher.sh -> mpiexec -n N <app>-opt -i <card>
 @/src/mooseagent  ------------------------------------------+
   graph.py (live pipeline) -- prompts / state / configuration / utils
     | align -> architect (per-card, parallel) -> run -> modify loop -> rearchitect / human interrupt
     | retrieval: enhanced_retrieval.py (architect), helper.py tool-calling ReAct (modify)
     v
 @/src/database   vector stores (*_inpcard from comment.json, *_dp from dp_detail.json), dp.json lookup
     ^
     | autocomment.py appends annotated cards -> load_vector_database.py re-indexes
```

- **Code:** `@/src/mooseagent` holds both workflows and all shared infrastructure. See `@/src/mooseagent/docs.md` for the node-by-node graph topology, the model-role table, the retrieval paths, and the MOOSE execution details.
- **Data:** `@/src/database` holds the corpora and the persisted FAISS/Chroma indexes (see `@/src/database/docs.md`).
- **Tests:** `@/tests` contains only template tests and notebooks that do not target this package (see `@/tests/docs.md`).
- `@/static` holds the README diagrams (`agentflow.png`, `autocomment.png`) and the template's `studio_ui.png`.
- The Nori research-tracking files (`STATUS.md`, `HISTORY.md`, `docs/`) belong to the fork's workflow, not to upstream.

### Core Implementation

- **Running.** Set `RUN_PATH` in `.env` to the absolute path of `@/src`. Every entry script appends `RUN_PATH` to `sys.path` to import `mooseagent`; there is no installable package. Edit the hardcoded `ABSOLUTE_PATH`, `MOOSE_DIR`, and `save_dir` in `@/src/mooseagent/configuration.py`, and the conda path and env name in `@/src/mooseagent/launcher.sh`. Then run `python src/mooseagent/graph.py`, whose `topic` string is hardcoded in its `__main__`. Generated cards land in `save_dir`, and the console is teed to `RUN_PATH/log/<timestamp>.log`.
- **Environment (`@/.env.example`).**
  - `OPENAI_API_KEY` (optional `OPENAI_API_BASE`) for the `openai/` models: the extracter, RAG summarizer, and AutoComment.
  - `HUOSHAN_API_KEY` / `HUOSHAN_API_BASE` for the default `huoshan/` DeepSeek models.
  - `SILICONFLOW_*` for an optional provider.
  - `BGE_M3_API_KEY` / `BGE_M3_API_BASE` / `EMBEDDING_MODEL`, used only if the local `BAAI/bge-m3` model cannot be loaded.
  - `TAVILY_API_KEY` is a template leftover.
  - Any `Configuration` field can also be overridden by an env var with its upper-cased name.
- **Dependencies.** Pinned versions in `@/requirements.txt` include `langgraph==0.3.11`, `langchain==0.3.20`, `langchain-deepseek`, and `faiss-cpu`. Packages imported but not pinned: `torch`/`transformers` (local embeddings), `chromadb` (Chroma backend), `pandas` (`statistics.py`), and `jq` (pinned; needed by `JSONLoader`). The README's `pip install requirements.txt` omits `-r`.

### Things to Know

- `@/langgraph.json` points at `./src/react_agent/graph.py:graph`, and `@/pyproject.toml` packages `src/react_agent`. Neither exists, so `langgraph dev`/Studio and `pip install .` do not reach MooseAgent. Because the in-repo `__main__` blocks never resume, nothing in the repo provides a way to answer the graph's `interrupt()` prompts.
- `@/.gitignore` ignores `*log`, `run_path/`, `**experiments`, `*chroma*`, and `.env`. The run-log directory (`src/log/`) must therefore be created by hand, and Chroma's sqlite metadata is not committed.
- `@/.vscode/launch.json` is the stock "Python: Current File" debug config.
- `README.md` and `README_CN.md` describe the same flow. Where they diverge from the code (model-field names, an output directory name), the details are recorded in `@/src/mooseagent/docs.md`.

Created and maintained by Nori.

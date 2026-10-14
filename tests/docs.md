# Noridoc: tests

Path: @/tests

### Overview

This folder holds the test scaffolding inherited from the LangGraph `react_agent` starter template, plus a few early author experiments. None of it exercises the MooseAgent pipeline in `@/src/mooseagent`.

### How it fits into the larger codebase

- `@/Makefile` (`make test` → `pytest tests/unit_tests/`) and `@/.github/workflows/unit-tests.yml` / `integration-tests.yml` target this folder. The workflows install from `@/pyproject.toml`, which still declares the template package `react_agent` at `src/react_agent`. That directory does not exist in this repo.
- `unit_tests/test_configuration.py` and `integration_tests/test_graph.py` import `react_agent` / `react_agent.configuration`. The integration test asks the template ReAct agent "Who is the founder of LangChain?" and asserts "harrison" is in the answer.
- `cassettes/` holds the template's recorded Anthropic API exchange (`claude-3-5-sonnet` with a Tavily `search` tool). LangSmith replays it when `LANGSMITH_TEST_CACHE=tests/cassettes` (set in the integration workflow).
- `unit_tests/test_retriever.py` is module-level code, not a test function. It imports `mooseagent.retrieval.get_local_retriever`, a module that does not exist in `@/src/mooseagent`, after adding `filepath.ABSOLUTE_PATH` to `sys.path`.
- `filepath.py` hardcodes `ABSOLUTE_PATH` to the author's Windows checkout (`E:/vscode/python/Agent/langgraph_learning/mooseagent/src`).

### Core Implementation

`test.ipynb` is a scratch notebook that predates the current graph:

- smoke-calls `ChatDeepSeek`
- builds a Chroma store from Lilian Weng blog posts and from `@/src/database/syntax.md` (the `create_vector_store` helper)
- prototypes `combine_code_with_description`, which later moved to `@/src/mooseagent/utils.py`
- holds Self-RAG-style graders (retrieval relevance, hallucination, answer, question re-writer)

The graders correspond to the `self_rag/` module that was deleted from `@/src/mooseagent` in commit `fe41664`.

### Things to Know

- Every collected test in `unit_tests/` and `integration_tests/` fails at import, because neither `react_agent` nor `mooseagent.retrieval` exists. Nothing here validates graph topology, prompts, retrieval, or MOOSE execution.
- The only end-to-end exercise of the system is running `@/src/mooseagent/graph.py` (or `statistics.py` for repeated runs over `@/src/database/cases`), which needs real LLM keys, the vector stores, and a MOOSE build.

Created and maintained by Nori.

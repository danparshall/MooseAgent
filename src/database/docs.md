# Noridoc: database

Path: @/src/database

### Overview

This folder holds the knowledge base that MooseAgent reads at runtime and that AutoComment and `load_vector_database.py` write to. It contains raw MOOSE dumps (example inputs, the object-parameter dump, markdown docs), two curated JSON corpora derived from them, and the persisted vector indexes built from those corpora under two embedding models.

### How it fits into the larger codebase

There is no data-access layer. The code in `@/src/mooseagent` reaches these files through paths built in `@/src/mooseagent/configuration.py` from the hardcoded `ABSOLUTE_PATH` (`/home/zt/workspace/MooseAgent/src`), and the entry scripts build some paths from `RUN_PATH` (`RUN_PATH/database/dp.json`). The runtime consumers are:

```
              dp.json  ──(graph __main__ loads into FlowState.dp_json)──> modify: check_app()
                       ──(autocomment rag node)──> per-app docs pasted into the comment prompt

  comment.json ──load_vector_database.py──> {EMB}_{faiss|Chroma}_inpcard ──┐
  dp_detail.json ──load_vector_database.py──> {EMB}_{faiss|Chroma}_dp ─────┤
                                                                          v
                         helper.py tools (retrieve_moose_case / retrieve_moose_dp)
                         enhanced_retrieval.py (queries both stores per question)
```

`{EMB}` is `Configuration.embedding_function`: `BGE_M3_EmbeddingFunction` by default, or `OPENAI`. The suffix is `Configuration.vector_store`, which defaults to `faiss`. The store has to be loaded with the same embedding that built it; the code does not check this.

### Core Implementation

| Artifact | Shape | Role |
|---|---|---|
| `dp.json` | dict: MOOSE object name (e.g. `AnalyticalIndicator`) → its parameter-dump text (`name = default # "doc"` lines) | Lookup table for `check_app` (any `type = X` where `X` is not a key gets flagged back to the LLM as "not found in the documentation") and for AutoComment's per-app docs. |
| `dp_detail.json` | list of `{App_name, description, document}`, where `description` is the prose from the MOOSE docs page and `document` is the same parameter dump as in `dp.json` | Source corpus for the `*_dp` vector stores. |
| `comment.json` | list of `{input_card_name, overall_description, annotated_input_card}` produced by AutoComment | Source corpus for the `*_inpcard` vector stores ("similar cases"). AutoComment appends to it. |
| `case_name.txt` / `case_name_uncommented.txt` | one Windows path per line (`E:/vscode/moose\examples\...\*.i`) | The full list of MOOSE example inputs, and the work queue that AutoComment pops from and periodically rewrites. |
| `case_all.txt` | concatenated `case name: ...` / `inpcard:` blocks of raw example inputs | Raw dump. No current code reads it. |
| `dp.txt` | raw MOOSE `--dump` output (`### START DUMP DATA ###`) | Raw source for `dp.json`/`dp_detail.json`. No current code reads it. |
| `modules.md`, `source.md`, `syntax.md` | concatenated MOOSE documentation markdown | Earlier RAG source (`Configuration.DATABASE_NAME="*.md"` and `docs_dir` remain, but nothing uses them; `@/tests/test.ipynb` experimented with `syntax.md`). |
| `test.txt` | one JSON-string-encoded input card | Scratch artifact. |
| `cases/` | `caseN.txt`: short natural-language benchmark task descriptions (heat conduction, mechanics, phase field, MultiApp coupling, etc.); `case.txt` is a combined list with "Key Points to Test" | Benchmark prompts. `@/src/mooseagent/statistics.py` reads `cases/case{i}.txt` as the `topic`. |

**Vector-store directories.** These are binary indexes. Do not hand-edit them.

| Directory pattern | Format | Contents |
|---|---|---|
| `<EMB>_faiss_<inpcard\|dp>/` | LangChain `FAISS.save_local` (`index.faiss` + pickled docstore `index.pkl`) | Loaded with `FAISS.load_local(..., allow_dangerous_deserialization=True)`. |
| `BGE_M3_EmbeddingFunction_Chroma_<inpcard\|dp>/` | Chroma HNSW segment files under a UUID subdir | Only the segment binaries are committed. Chroma's `chroma.sqlite3` metadata DB is matched by the `*chroma*` pattern in `@/.gitignore` and is absent. |
| `hashes.json` (in every store) | list of SHA-256 hex digests of each indexed document's `page_content` | The incremental-update ledger used by `load_vector_database.py` to skip already-embedded records. |

Each indexed document is one whole JSON record, serialized by `JSONLoader(jq_schema=".[]", text_content=False)`. There is no chunking. A retrieved "case" is therefore the full annotated input card with its description, and a retrieved "dp" hit is the whole object's prose plus parameter dump.

### Things to Know

- **Store coverage differs.** The BGE-M3 FAISS and Chroma stores and `OPENAI_faiss_dp` have one hash per record of the current JSONs. `OPENAI_faiss_inpcard` has far fewer hashes than `comment.json` has records, which means it was built from an earlier, smaller `comment.json` (git history shows `comment.json` growing after the OpenAI index was created). Switching `embedding_function` to `"OPENAI"` therefore also shrinks the case corpus.
- **Name case mismatch.** The default config uses lowercase `faiss`, which matches the FAISS dirs. Setting `vector_store="chroma"` yields `..._chroma_...`, but the committed dirs are `..._Chroma_...`. They only coincide on case-insensitive filesystems.
- **Rebuilding an index** means editing `rag_json_path` and `PERSIST_DIRECTORY` in `@/src/mooseagent/configuration.py` (there is no CLI) and running `load_vector_database.py`. Re-embedding needs the same embedding backend (local `BAAI/bge-m3` with CLS pooling, a remote BGE-M3 endpoint, or OpenAI) to stay consistent with existing vectors.
- AutoComment also writes per-card text files into a `comment/` directory next to `comment.json` (derived as `COMMENT_PATH.replace(".json", "/" + name)`). That directory is not in the repo.
- Paths in `case_name*.txt` are Windows absolute paths to a local MOOSE checkout on the author's machine. They are only meaningful for AutoComment on that machine.

Created and maintained by Nori.

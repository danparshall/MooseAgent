<!-- Generated during: convos/20261014_mooseagent_codebase_orientation.md -->

# MooseAgent: author, paper, and code audit

Upstream repo `taozhan18/MooseAgent` at `075083e` (2025-09-04), checked against arXiv:2504.08621 v1 (2025-04-11) and v2 (2025-10-14). Paper: `papers/ZhangT_Jiao__2025--mooseagent.pdf`; entry `mooseagent` in PAPER_SUMMARIES.md.

## Verdict (Dan, 2026-10-14)

**Not worth further time.** Dan's stated reason: this is not INL work. The authors are all at the Nuclear Power Institute of China, and INL appears only as a citation. Separately, the technical findings below suggest the public code cannot reproduce the paper.

## Who wrote it ("zt")

Evidence table: [`20261014_author_evidence.csv`](20261014_author_evidence.csv). Both rows were checked with `identify-chinese-person`'s `verify`.

- **Tao Zhang (张涛).** The characters and affiliation come from his own homepage, https://taozhan18.github.io/, which also links this repo.
  - The `/home/zt/...` paths in the code are his initials in Chinese order (Zhang Tao).
- **Current position:** master's student at the Nuclear Power Institute of China (NPIC), Chengdu, under Prof. Jiao Yongjun.
- **Prior:**
  - BS in nuclear engineering with a computer science minor, Tsinghua University, 2018–2022.
  - Visiting student in Tailin Wu's lab at Westlake University.
- **Paper byline:** all four authors (Tao Zhang, Zhenhai Liu, Yong Xin, Yongjun Jiao) share one affiliation, *State Key Laboratory of Advanced Nuclear Energy Technology, NPIC*.
  - NPIC is a reactor-design institute under the state-owned CNNC.
  - INL appears only as reference [9], the standard MOOSE citation.
  - We did not look up the co-authors' Chinese characters.
  - We did not check NPIC's or CNNC's US export-control status.
- **Repo history:** one developer wrote everything. There are 55 commits from `taozhan18`/`taozhan`, all with one QQ email (a Tencent messenger account number). Paths are hardcoded for two of his machines: `/home/zt` on Linux and `E:/vscode` on Windows. The tests and `langgraph.json` are leftovers from the LangGraph template. Several dependencies are missing. The overall pattern is code that was only ever run on the author's own machines.

## Code bugs (verified by reading the code)

1. **Retrieved cases never reach the architect.**
   - `graph.py:183-185` passes `cases=similar_cases` into `SYSTEM_ARCHITECT_PROMPT.format(...)`, but the template has no `{cases}` placeholder.
   - `str.format` ignores extra keyword arguments without any error, so nothing is raised.
   - The placeholder was removed in `f3b938b` (2025-04-09, "architect is also helper").
   - `d36c43e` (2025-04-11) restored the retrieval call but not the placeholder.
   - `f750c88` (2025-09-04, "enhance retrieval") upgraded that retrieval to multi-query. Its output is also dropped.
   - `git grep "{cases}"` finds no match at `d36c43e`, `812c517` or `075083e`.
   - Retrieval *does* reach the `modify` (error-correction) step through the helper's tools.
2. **"Success" means empty stderr.**
   - `graph.py:271` tests `result.stderr == ""` and never checks the return code.
   - A crash that writes nothing to stderr counts as a success, and a run that only prints warnings counts as a failure.
3. Smaller issues are recorded in `src/mooseagent/docs.md`:
   - routing to a node named "End" that doesn't exist
   - human-intervention interrupts that nothing can resume
   - configuration overrides that mostly don't take effect
   - incomplete Chroma stores
   - tests that can't import

## Paper vs code

| Paper claim | Code / evidence | Consequence |
|---|---|---|
| §2.1: the architect "retrieves similar input files … to serve as references for generating the initial input file code"; §3.2 credits 100% cases to "a knowledge base rich with similar reference cases" | Bug 1: from v1 onward, the pushed code never puts the cases into the architect prompt | If the results came from pushed code, retrieval at generation time contributed nothing |
| §3.3 / Table 2 ablation "without the knowledge base": 93% → 76% | `graph_ablation.py` makes two changes: (a) `similar_cases = ""` at the architect, which does nothing because of Bug 1; (b) replaces the RAG tool-calling helper in `modify` with a plain LLM | On pushed code, the whole drop comes from RAG during error correction, not from examples at generation |
| The ablation drop is "significant" | 42/45 vs 34/45 runs (9 cases × 5 runs); Fisher exact two-sided p ≈ 0.039, treating runs as independent. They aren't, since runs are grouped by case | Weak evidence. Each per-case "1 → 0.8" is a single run |
| "Success rate" (v1 §4.1: "the ratio of the number of successfully run test cases"; v2 never defines it) | Same as the code's stderr test (Bug 2). No comparison with reference or analytic solutions, although steady 1D conduction has an exact one and v1 promises "numerical accuracy" | 93% means "ran without an error message", not "computed the right answer" |
| "maximum number of iterations to three" | `MAX_ITER = 7`, `MAX_REARCHITECT = 3` | Configuration mismatch |
| v2 Table 1 | Steady heat, elasticity and phase field rows have the same token counts as v1 (24673, 40857, 86696) | The April runs appear to be reused in October |
| v2 treats human feedback as future work | HEAD has `enable_human_intervention=True` (added 2025-09-04) | The pushed HEAD is not the configuration the paper evaluates |

## Did the results come from the pushed code? Probably not.

- From `aa01fdb` (Apr 11) until `812c517`, `statistics.py` imports `mooseagent.graph1`, which had already been renamed, so the harness could not run.
- At `d36c43e`, `graph1`'s success path never sets `run_result="success"`, so the harness would score 0%. This was fixed only in `3ee030d` (Jul 4).
- `812c517`'s harness runs `case22.txt`; only case1–10 are in the repo.
- The iteration count and human-intervention setting differ, as in the table above.

**What we can fairly say:** the paper's results cannot be reproduced from the public repo, and the public code contradicts the paper's explanation of where the retrieval gain comes from. That is weaker than "the paper's mechanism is wrong."

**What would settle it:** the authors' run logs or `experiments/*/stats.csv`, the unpushed case files, or rerunning the pushed ablation and logging the architect's prompt.

**Regardless of which code was used:** "success" does not test correctness. Treat this paper as weak evidence for the claim that "LLM agents can autonomously run nuclear simulation codes."

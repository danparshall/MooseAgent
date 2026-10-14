# 20261014_mooseagent_codebase_orientation

**Date:** 2026-10-14
**Branch:** codebase-docs
**Surface:** claude-code
**Machine:** Dans-MacBook-Pro (pro)
**Session:** Dan (pro, MooseAgent, 20261014T1207, opus-5.5) · /Users/dan/.claude/projects/-Users-dan-code-MooseAgent/690a0e54-8dec-4eb0-8f40-6066957c0057.jsonl · —

## Summary

Dan cloned MooseAgent, a LangGraph multi-agent system that writes and debugs input files ("input cards") for MOOSE, INL's open-source multiphysics framework. He wanted to run the full Nori summarization pass to understand how it dispatches agents. The repo belongs to someone else and we can only read it, so we forked it to `danparshall/MooseAgent`. We kept `master` identical to upstream and put all the Nori material on `codebase-docs` in `.worktrees/codebase-docs`:
- research scaffolding: STATUS, HISTORY, `docs/active/codebase-docs/`
- `docs.md` files from `nori-initial-documenter`
- a paper collection holding the MooseAgent arXiv paper

The documenter found, and we confirmed by reading the code, that retrieved cases never reach the architect prompt and that "success" means MOOSE wrote nothing to stderr. Neither bug raises an error. Python's `str.format` ignores extra keyword arguments, and `subprocess.run` doesn't raise on failure unless `check=True` is passed. We identified the author "zt" as Tao Zhang (张涛), a master's student at the Nuclear Power Institute of China. A subagent then compared the paper with the code. The paper says retrieved examples help the first draft. Its ablation, run on the pushed code, would only measure retrieval during error correction. Its "success" metric never checks correctness. Several signs suggest the results came from code that was never pushed.

Dan's verdict: the repo isn't worth more time because it isn't INL work.

## Topics Explored
- How Nori fits a third-party repo you can't push to: fork as a separate remote, keep `master` identical to upstream, keep all scaffolding on a branch, and ignore `.worktrees/` in `.git/info/exclude`
- The architecture: graph structure, which model plays which role, retrieval, how MOOSE is run (see `src/mooseagent/docs.md`)
- Why the bugs don't raise errors: Python behavior, not LangGraph
- Who the author is, and whether INL was involved
- The paper's claims against the code history

## Provisional Findings
- Author: Tao Zhang (张涛), NPIC master's student; all four authors are at NPIC; no INL involvement. Evidence: `results/20261014_author_evidence.csv`.
- Bug 1 (retrieval dropped at the architect) and Bug 2 (stderr-only success) are confirmed in the code.
- The paper's ablation (93% → 76%) on pushed code measures retrieval in the error-correction loop, not at generation. The gap is weak: 42/45 vs 34/45 runs, p ≈ 0.039.
- The paper's results probably did not come from the pushed code, so "not reproducible from the public repo" is a fair claim and "the mechanism is wrong" is not.
- Full details: `results/20261014_paper_code_audit.md`.

## Decisions Made
- Fork plus worktree, with no changes to `master` (Dan's choice).
- Dan considers the repo not worth further time because it isn't INL work. The line is not formally closed; STATUS.md still lists it as active.

## Results
- [results/20261014_paper_code_audit.md](../results/20261014_paper_code_audit.md): author, bugs, and a paper-versus-code table
- [results/20261014_author_evidence.csv](../results/20261014_author_evidence.csv): verified identification evidence
- Paper `mooseagent` in PAPER_SUMMARIES.md; `docs.md` files at the repo root, `src/mooseagent/`, `src/database/` and `tests/`

## Open Questions
- Close and archive this line with `finishing-a-research-branch`?
- Does INL have its own LLM-agent work on MOOSE? That might be the thing Dan actually wants. Not searched.
- NPIC/CNNC export-control status was not checked.
- Two `git commit` denials happened this session, with an unknown trigger; they were not logged to chain-hook-maintenance.

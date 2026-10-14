# Paper Summaries

<!-- BEGIN GENERATED: summaries-header -->
> 1 papers total (1 with local PDFs, 0 text-only).
<!-- END GENERATED: summaries-header -->

One `### <slug>` entry per paper, under `##` section headings, in the metadata format of the `paper-index` skill. Add papers with the `add-paper` skill; PAPER_INDEX.md and PAPER_RELATED.md are generated from this file.

## LLM agents for simulation codes

### mooseagent

**Title:** MooseAgent: A LLM Based Multi-agent Framework for Automating Moose Simulation (Zhang, Liu, Xin, Jiao, 2025 — Nuclear Power Institute of China)

- **Authors:** Tao Zhang, Zhenhai Liu, Yong Xin, Yongjun Jiao (State Key Laboratory of Advanced Nuclear Energy Technology, Nuclear Power Institute of China)
- **Date:** October 2025 (v2; v1 April 2025)
- **File:** `ZhangT_Jiao__2025--mooseagent.pdf`
- **Source:** https://arxiv.org/abs/2504.08621v2
- **Focus:** LLM agent for MOOSE input files
- **One-liner:** LangGraph multi-agent system (DeepSeek-R1 writes, DeepSeek-V3 elsewhere) turns natural-language specs into MOOSE input files with RAG and an execute-analyze-correct loop; 93% "success" (ran without error) over 9 cases x 5 runs, falling to 76% with the knowledge base removed.
- **Summarized:** opus-5.5, 2026-10-14

**Summary:** The paper presents MooseAgent, the system implemented in this repo: an alignment agent expands the user's request into per-file descriptions (with optional human confirmation), an architect writes each MOOSE input file after retrieving similar annotated input files, and an execute-analyze-correct loop feeds MOOSE errors to a "Moose Assistant" (an LLM with retrieval tools over the knowledge base), re-architecting when errors recur. The knowledge base is 8,000+ MOOSE input files annotated by an LLM plus scraped per-object documentation, embedded with BGE-M3 into FAISS (top-3 retrieval).

**Key findings:**
- Table 1 (v2): success rate 1.0 on 7 of 9 cases; plasticity 0.6, thermal-mechanical coupling (MultiApp) 0.8; mean 93%. Each case run 5 times. Mean 60,777 tokens per run, stated as under 1 yuan at DeepSeek-R1 prices.
- Table 2 ablation, "without database": mean success falls to 76% (plasticity 0.2, phase change 0.6, thermal-mechanical 0.6, linear elasticity/porous/phase field 0.8; steady heat and prismatic fuel stay 1.0); mean tokens fall to 54,724. The paper credits the drop to losing "essential context and examples" from the knowledge base.
- "Success" is never defined in v2 beyond the loop ending when "the simulation runs successfully"; v1 defines success rate as the share of "successfully run" cases. No comparison against analytic or reference solutions is reported, although v1's contribution list promises "numerical accuracy."
- All failed runs are attributed to repeated, unfixable errors that hit the iteration cap (stated as 3).
- v1 (April 2025) had 8 cases, mean success about 76% (thermal-mechanical 0.4; porous, phase change, plasticity 0.6) and no ablation. v2 adds the prismatic fuel case and the ablation; three v2 Table 1 rows (steady heat, elasticity, phase field) carry token counts identical to v1, so they appear to be the April runs reused.

**Relevance:** This is the paper behind the code in this repo. Its retrieval claims bear directly on the code finding that retrieved cases never reach the architect prompt (see src/mooseagent/docs.md), and its success metric matches the code's stderr-only success test.

---

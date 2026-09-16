# SCHISM

**Conflict-Aware Multi-Hypothesis Analysis of Multi-Source Cyber Threat
Intelligence**

When multiple cyber threat intelligence (CTI) reports describe the same attack
campaign, their conclusions occasionally conflict. SCHISM organizes the support
and conflict relations among reports into a **signed evidence graph** and
distills the most credible conclusion instead of collapsing onto a single
answer too early:

1. **Signed evidence graph construction** — each report is distilled into a
   structured behavior profile; deterministic matching over canonical keys
   turns support and conflict relations among reports into graph edges, and
   spectral partitioning carves out competing hypotheses aligned with the
   query.
2. **Adaptive scheduling** — a frustration-index-driven gate decides the
   reasoning depth: with no conflict a consensus conclusion is returned
   directly, and the heavier the conflict, the more rounds of reasoning are
   invested.
3. **Counterexample-driven hypothesis testing** — counterexamples that could
   overturn the leading hypothesis are sought on demand; the change of its
   strength on the graph decides whether the conclusion has stabilized.

The system is orchestrated by one deterministic controller plus three LLM
agents (ABP extraction, evidence graph, hypothesis), all closed on the same
signed evidence graph.

## Repository layout

```
├── src/                  # SCHISM implementation
│   ├── core/             # signed graph, spectral partition, conflict,
│   │                     # canonical matching, ABP, transitivity
│   ├── agents/           # orchestrator + 3 LLM agents and their prompts
│   ├── tasks/            # task adapters (TAA / RCM / ATE / MCQ)
│   ├── retrieval/        # hybrid retriever over the CTI corpus
│   ├── utils/            # LLM client (vLLM OpenAI-compatible), settings
│   ├── controller.py     # deterministic orchestration loop
│   └── run.py            # main entry point
├── baselines/            # compared methods: ArbGraph, CyberRAG,
│                         # ITER-RETGEN, NaiveRAG, RAGIntel, Search-o1
│                         # (adapted to CTIBench; upstream repos referenced
│                         # in each directory)
├── experiments/          # run scripts for the 4 CTIBench tasks and baselines
│   └── results/          # raw + report JSON of the main experiments on
│                         # backbone LLMs (GLM-4-9B, Qwen3-30B-A3B,
│                         # Mistral-7B)
└── data/                 # CTIBench task sets and the CTI report corpus
```

## Setup

1. Serve a backbone LLM behind an OpenAI-compatible endpoint (e.g. vLLM) and
   point `src/settings.yaml` → `models.chat_model.api_base` at it, and serve
   the retrieval corpus in Weaviate (`models.weaviate_model.url`, class
   `APT` with fields `content`, `report_title`, `section_title`,
   `source_file`).
2. Install dependencies:

```bash
pip install -r requirements.txt
```

3. Run one task (TAA here; RCM/ATE/MCQ analogous):

```bash
python -m src.run --task taa
```

Baselines are run through `experiments/run_taa_baselines.py` and
`experiments/run_mcq_baselines.py`; `experiments/run_matrix.sh` /
`experiments/run_backbones.sh` show the full experiment matrix.

## Results

`experiments/results/<backbone>/` contains, per method and per task, the raw
per-sample outputs (`*_results.json`) and the aggregated metrics
(`*_results_report.json`) used in the paper. Backbones shipped here:
`qwen3-30b-a3b` (Qwen3-30B-A3B), `glm-9b` (GLM-4-9B), and `mistral`
(Mistral-7B).

## Data sources

- The task sets (TAA/ATE/MCQ/RCM) come from the
  [CTIBench](https://github.com/IBM/CTIBench) benchmark.
- The retrieval corpus aggregates publicly available APT reports from three
  community-maintained GitHub collections:
  [blackorbird/APT_REPORT](https://github.com/blackorbird/APT_REPORT),
  [RedDrip7/APT_Digital_Weapon](https://github.com/RedDrip7/APT_Digital_Weapon),
  and
  [CyberMonitor/APT_CyberCriminal_Campagin_Collections](https://github.com/CyberMonitor/APT_CyberCriminal_Campagin_Collections).
  The reports are converted to plain text and sectioned for retrieval.

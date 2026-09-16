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
   turns the support and conflict relations among reports into graph edges,
   and spectral partitioning carves out competing hypotheses aligned with the
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
│   ├── agents/           # orchestrator + LLM agents and their prompts
│   ├── tasks/            # task adapters (TAA / RCM / ATE / MCQ)
│   ├── retrieval/        # hybrid retriever over the CTI corpus
│   ├── utils/            # LLM client (vLLM OpenAI-compatible), settings
│   ├── controller.py     # deterministic orchestration loop
│   ├── metrics.py        # deterministic metrics (conflict level, gates)
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

## Code structure

- `src/controller.py` — the deterministic orchestrator: pipeline staging,
  conflict gating, iteration, and the refutation chain; no LLM calls.
- `src/agents/` — the LLM agents (`hypothesis_agent`, `evidence_graph_agent`,
  `reporter_agent`) plus prompt templates in `src/agents/prompts/` (YAML).
  Prompts default to the TAA wording in the root directory; the `ate/`,
  `mcq/`, and `rcm/` subdirectories hold per-task overrides, loaded with a
  fallback to the default (`get_task_prompt` in `src/utils/settings.py`).
- `src/core/` — the deterministic core: `awm.py` (AWM data structures,
  `GraphEdge` with three edge classes, `BehaviorProfile` with canonical
  keys), `abp.py` (behavior profile extraction), `canonical_match.py`
  (four-dimension canonical-key matching that decides support / conflict /
  undecidable), `signed_graph.py`, `spectral_partition.py` (signed Laplacian,
  eigengap, k-means), `conflict.py` and `transitivity.py` (frustration index
  and edge-sign inference).
- `src/tasks/` — task adapters over a common `TaskAdapter` base
  (`taa.py`, `mcq.py`, `rcm.py`, `ate.py`).
- `src/retrieval/` — the Weaviate retriever, hybrid semantic + BM25 search
  with RRF fusion.
- `src/utils/llm_client.py` — LLM calls through a vLLM OpenAI-compatible
  endpoint, with automatic truncation that guards against context overflow.

## Configuration

All shared settings live in `src/settings.yaml`:

| Key | Default | Meaning |
|---|---|---|
| `global.top_k` | 10 | retrieval size, unified across our method and all baselines |
| `global.retrieval_mode` | hybrid | `semantic` / `keyword` / `hybrid` |
| `global.hybrid_alpha` | 0.4 | semantic weight in RRF fusion |
| `global.timeout` | 1800 | per-sample timeout (s), unified across methods |

Pipeline-specific parameters (gates $\theta_1$, $\theta_2$, per-task
$I_{\max}$, hypothesis-strength blending, counterexample pool size) are
documented inline in the `pipeline:` section of the same file.

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

## Running experiments

Task datasets: TAA 50 samples, ATE 60, RCM 1000, MCQ 2500. All task runners
share the same flags: `--start/--end` (sample range, for debugging),
`--resume` (checkpoint restart), `--timeout` (per-sample seconds).

Our method:

```bash
python experiments/run_taa.py                     # full TAA run (50)
python experiments/run_mcq.py                     # full MCQ run (2500, lighter defaults)
python experiments/run_rcm.py                     # full RCM run (1000)
python experiments/run_ate.py                     # full ATE run (60)
python experiments/run_taa.py --start 0 --end 3   # debug a few samples
python experiments/run_taa.py --resume            # resume from checkpoint
```

Baselines (TAA and MCQ use dedicated runners; RCM and ATE switch to
`--mode baseline`):

```bash
python experiments/run_taa_baselines.py --method naive_rag
python experiments/run_mcq_baselines.py --method all
python experiments/run_rcm.py --mode baseline --method search_o1
python experiments/run_ate.py --mode baseline --method arbgraph
# methods: naive_rag | iter_retgen | search_o1 | cyberrag | rag_intel | arbgraph | all
```

`experiments/run_matrix.sh` and `experiments/run_backbones.sh` show the full
backbone-by-task matrix used for the paper.

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
  The original reports (PDF and web pages) are converted to plain text with
  DeepSeek-OCR and sectioned for retrieval.

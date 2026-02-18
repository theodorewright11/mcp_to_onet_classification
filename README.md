# MCP Automation Classification

A pipeline for classifying [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers against the [O\*NET occupational framework](https://www.onetcenter.org/), measuring how much each MCP server can automate real-world occupational tasks.

## What It Does

MCP servers are plugin-like tools that give AI assistants access to external services, APIs, and data sources. This project asks: **which O\*NET occupational tasks could each MCP automate, and to what degree?**

For each of ~9,000+ MCP servers scraped from [mcp.so](https://mcp.so):

1. **Embedding retrieval** — Voyage-4-large embeddings find the top-80 most semantically similar Detailed Work Activities (DWAs) from O\*NET.
2. **DWA selection** — GPT-4.1 evaluates those 80 DWAs and selects up to 15 whose underlying tasks the MCP could plausibly automate.
3. **Task rating** — GPT-4.1 rates each O\*NET task (1–5 scale) for how much this specific MCP could automate it.
4. **Aggregation** — Ratings are aggregated across all MCPs to produce a ranked list of tasks by automation potential.

## Deliverable Files

| File | Description |
|------|-------------|
| `data/mcp/results/mcp_results_YYYY-MM-DD.csv` | Per-MCP classification results (DWA selections + task ratings) |
| `data/mcp/results/task_results_YYYY-MM-DD.csv` | Aggregated task-level statistics across all MCPs |

The most recent dated version of each is the current canonical dataset. Older dated files are kept in the same folder for reference.

### Output Columns — `mcp_results_*.csv`

| Column | Description |
|--------|-------------|
| `title` | MCP server name |
| `url` | mcp.so URL |
| `text_for_llm` | Combined description used as LLM input |
| `uploaded_clean` | Upload date (MM-DD-YYYY) |
| `dwa_status` | `selected` / `none` / `not_enough_info` / `not_occ_relevant` / `api_error` |
| `dwas_selected` | Semicolon-separated DWA titles selected by LLM (up to 15) |
| `n_dwas_selected` | Count of selected DWAs |
| `task_ratings` | `"task text (occupation): rating; ..."` — 1–5 automation rating per task |
| `n_tasks_rated` | Number of tasks rated |
| `n_tasks_sent` | Number of tasks sent to the rating prompt |
| `dwas_used_for_tasks` | DWAs that contributed tasks (subset of `dwas_selected`) |

### Output Columns — `task_results_*.csv`

| Column | Description |
|--------|-------------|
| `task` | O\*NET task text |
| `occupation` | Occupation title |
| `n_ratings` | Number of MCPs that rated this task |
| `mean_rating` | Average automation rating (1–5) |
| `median_rating` | Median rating |
| `max_rating` / `min_rating` | Max and min ratings |
| `p25_rating` / `p75_rating` | 25th and 75th percentile ratings |

### Task Rating Scale

| Rating | Meaning |
|--------|---------|
| 1 | No meaningful automation (0–10%) — MCP doesn't address this task |
| 2 | Minimal support (10–30%) — assists with a small component |
| 3 | Partial automation (30–60%) — automates a meaningful portion; human judgment needed |
| 4 | Substantial automation (60–90%) — performs most of the workflow |
| 5 | Near-full automation (90–100%) — executes end-to-end with little human involvement |

---

## Project Structure

```
mcp_classification_final/
├── scripts/
│   ├── mcp_scraper.py           # Scrapes mcp.so for new MCP servers
│   ├── data_prep.ipynb          # Cleans raw scrape data + generates embeddings
│   ├── llm_classification.ipynb # LLM-based classification pipeline (sync + batch)
│   └── analysis.ipynb           # Research notebook: embedding quality + model comparison
│
├── data/
│   ├── mcp/
│   │   ├── results/
│   │   │   ├── mcp_results_*.csv      # Classification results (DELIVERABLE)
│   │   │   └── task_results_*.csv     # Aggregated task ratings (DELIVERABLE)
│   │   ├── raw/
│   │   │   ├── mcp_scraped_*.csv      # Raw scrape output
│   │   │   └── mcp_data_*.csv         # Cleaned MCP data (input to classification)
│   │   ├── mcp_classification_teddy.csv        # Manual ground-truth (30 MCPs)
│   │   └── gpt-4.1_v5.2_occ_gwa_iwa_dwa_task.csv  # Prior GPT-4.1 v5.2 classifications
│   ├── onet/
│   │   ├── onet_data.csv              # O*NET tasks (23,850 rows)
│   │   ├── dwa_reference_v30.1.csv    # DWA hierarchy reference
│   │   └── tasks_to_dwas_v30.1.csv    # Task-to-DWA mapping
│   ├── embeddings/                    # Voyage-4-large .npy embedding arrays
│   │   ├── voyage_dwa_emb.npy         # DWA embeddings (2,083 × 1024) — generated once
│   │   ├── voyage_mcp_emb.npy         # MCP embeddings for the Jan 2026 full run
│   │   └── voyage_mcp_emb_*.npy       # MCP embeddings for subsequent scrape batches
│   └── batch/                         # Batch API intermediate state (git-ignored)
│
├── requirements.txt
├── .env                               # API keys (git-ignored — see Setup below)
└── .gitignore
```

---

## Setup

### 1. Create a virtual environment and install dependencies

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Create a `.env` file with your API keys

```
OPENAI_API_KEY=sk-proj-...
VOYAGE_API_KEY=pa-...
```

---

## Running the Pipeline

### Step 1 — Scrape new MCPs

```bash
python scripts/mcp_scraper.py
```

- Automatically loads the most recent `mcp_results_*.csv` from `data/mcp/results/` to skip already-classified URLs.
- Outputs `data/mcp/raw/mcp_scraped_YYYY-MM-DD.csv` with only new MCP servers.

### Step 2 — Clean and embed

Open `scripts/data_prep.ipynb` and run all cells.

- **Config cell** (top): set `RAW_SCRAPE_FILE` to the filename from Step 1, or leave as `None` to auto-detect the most recent scrape in `data/mcp/raw/`.
- **Outputs:**
  - `data/mcp/raw/mcp_data_YYYY-MM-DD.csv` — cleaned MCPs
  - `data/embeddings/voyage_mcp_emb_YYYY-MM-DD.npy` — Voyage-4-large embeddings for the new MCPs
  - `data/embeddings/voyage_dwa_emb.npy` — DWA embeddings (generated once; reused on subsequent runs)

### Step 3 — Classify

Open `scripts/llm_classification.ipynb` and run the relevant pipeline section.

- **Config cell** (top): set `MCP_DATA_FILE` and `MCP_EMB_FILE` to the filenames from Step 2. `EXISTING_RESULTS_FILE` auto-detects the most recent `mcp_results_*.csv` in `data/mcp/results/` to merge into.
- **Choose a pipeline:**
  - **Synchronous** ("Run Classification Pipeline" section) — simpler, runs all at once, higher cost per request.
  - **Batch API** ("Batch API Classification Pipeline" section) — 50% cheaper, higher rate limits, async (up to 24h turnaround). Recommended for large runs.
- **Outputs** (both pipelines produce the same format):
  - `data/mcp/results/mcp_results_YYYY-MM-DD.csv` — combined (historical + new) classification results
  - `data/mcp/results/task_results_YYYY-MM-DD.csv` — aggregated task-level statistics over all MCPs

Previous `mcp_results_*` and `task_results_*` files are preserved. The latest-dated file is the current canonical dataset.

---

## Batch API Pipeline Details

The batch pipeline is the recommended method for large runs. It submits all requests to OpenAI's Batch API and polls for completion.

**How it works (run cells in this order):**

1. Prerequisites: "Imports and Config", "Load Data", "Core Functions", "Select MCPs to Classify"
2. "Batch Config + Output Paths"
3. "Generate DWA Selection JSONL" → "Upload + Create Batch"
4. DWA poll cell (auto-waits every 30s until complete)
5. "Parse DWA Results"
6. "Generate Task Rating JSONL" → "Upload + Create Task Batch"
7. Task poll cell (auto-waits)
8. "Parse + Combine + Save"
9. "Batch Task-Level Aggregation"

**Kernel-restart safe:** Intermediate state is saved to `.pkl` files in `data/batch/`. If the kernel restarts between steps, re-run the prerequisite cells, set `DWA_BATCH_ID` / `TASK_BATCH_ID` manually, and continue from the poll cell.

**Resuming a stopped batch:** The pipeline automatically retries any `api_error` rows from a partial run. Simply re-run from "Batch Config + Output Paths" — it will pick up where it left off.

---

## O\*NET Hierarchy

```
GWA (General Work Activity)      — e.g., "Processing Information"
  └─ IWA (Intermediate Work Activity) — e.g., "Compile records, documentation, or other data."
       └─ DWA (Detailed Work Activity) — e.g., "Compile operational data."
            └─ Task                    — e.g., "Compile information about flights from flight plans..."
                 └─ Occupation         — e.g., "Aircraft Pilots and Copilots"
```

Classification works bottom-up: the pipeline selects DWAs, then retrieves and rates the tasks beneath them.

---

## Embedding Approach

Embeddings are committed to the repository in `data/embeddings/`.

- **Model:** `voyage-4-large` (Voyage AI, 1024-dim) — requires `VOYAGE_API_KEY` to regenerate
- **DWA embeddings** (`voyage_dwa_emb.npy`) — 2,083 O\*NET DWA titles; generated once, never changes
- **MCP embeddings** — one `.npy` file per scrape batch, named `voyage_mcp_emb_YYYY-MM-DD.npy`

To regenerate embeddings (e.g. after adding new MCPs), run the embedding section of `data_prep.ipynb`.

---

## Ground-Truth Data

`data/mcp/mcp_classification_teddy.csv` — 30 MCPs manually classified with full GWA/IWA/DWA/task hierarchy, deployability scores, and notes. Used as ground truth for validating the pipeline.

`data/mcp/gpt-4.1_v5.2_occ_gwa_iwa_dwa_task.csv` — 31 MCPs classified by an earlier GPT-4.1 pipeline (v5.2) with extensive per-field reasoning. Kept for comparison and validation.

`scripts/analysis.ipynb` — Research notebook that compares embedding nearest-neighbor retrieval against these manual classifications to validate `TOP_K_DWAS = 80` captures the right DWAs.

---

## Key Statistics

> *To be updated once the preliminary dataset is finalized.*

| Metric | Value |
|--------|-------|
| MCPs classified | — |
| MCPs with DWA selections | — |
| Total task-rating pairs | — |
| Unique (task, occupation) pairs | — |
| Unique DWAs in O\*NET | 2,083 |
| O\*NET tasks total | 23,850 |

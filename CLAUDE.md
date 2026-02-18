# MCP Classification Project

## What This Project Does
Classifies ~8,957 MCP (Model Context Protocol) servers from mcp.so against the O*NET occupational framework. Maps each MCP to Detailed Work Activities (DWAs), Intermediate Work Activities (IWAs), General Work Activities (GWAs), and occupational tasks. Scores each on deployability (1-6) and workflow automation potential (1-6).

## Directory Structure
```
data/
  mcp/
    mcp_scraped_2026-01-22.csv      # Raw scraped data (11,307 rows)
    mcp_data_2026-01-22.csv         # Cleaned MCP data (8,957 rows) - key col: text_for_llm
    mcp_classification_teddy.csv    # 30 manually classified MCPs (ground truth)
    gpt-4.1_v5.2_occ_gwa_iwa_dwa_task.csv  # 31 GPT-4.1 classified MCPs
  onet/
    onet_data.csv                   # O*NET tasks (23,850 rows) - key col: dwa_title (~2,083 unique)
    dwa_reference_v30.1.csv         # DWA hierarchy reference (2,088 DWAs)
    tasks_to_dwas_v30.1.csv         # Task-to-DWA mapping
  embeddings/                       # Saved .npy embedding arrays
scripts/
  data_prep.ipynb                   # Data cleaning + embedding + comparison
  llm_classification.ipynb          # LLM-based classification pipeline
  mcp_scraper.py                    # Web scraper for mcp.so
```

## Key Data Columns
- **MCP data**: `title`, `url`, `text_for_llm` (combined description/features/use_cases), `uploaded_clean`
- **O*NET data**: `dwa_title`, `task`, `title` (occupation name)
- **Classification files**: `title`, `url`, `occ_relevant`, `gwa`, `iwa`, `dwa` (semicolon-separated), `task`, `deployability`, `workflow_auto`

## Embedding Models
Currently comparing two embedding models in `data_prep.ipynb`:
1. **Voyage-4-large** (API, `VOYAGE_API_KEY` env var required) - 1024-dim
2. **all-mpnet-base-v2** (local, sentence-transformers) - 768-dim

Embeddings saved as `.npy` files in `data/embeddings/`.

## Environment
- Python venv at `venv/`
- Key packages: pandas, numpy, voyageai, sentence-transformers, torch, openai, anthropic, google-genai, scikit-learn
- API keys loaded via `python-dotenv` from `.env`

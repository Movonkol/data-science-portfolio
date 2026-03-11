# LLM Word Association – Reverse Turing Test

## Overview

This project investigates how different Large Language Models (LLMs) respond when asked to produce a single word that would convince a human they are talking to another human. By repeatedly querying each model (100 runs per model per temperature), we build a dataset that is then analyzed using statistical and machine learning methods.

**Core question:** Do different LLMs systematically choose different "strategies" (emotional, physical, philosophical, etc.) to appear human?

## Models

| Model | API ID | SDK |
|---|---|---|
| Claude | `claude-sonnet-4-6` | `anthropic` |
| GPT-4o | `gpt-4o` | `openai` |
| Gemini | `gemini-2.5-flash` | `google-generativeai` |
| Grok | `grok-3` | `openai` (xAI base_url) |
| DeepSeek | `deepseek-chat` | `openai` (DeepSeek base_url) |
| Qwen 2.5 | `qwen2.5:14b` | `ollama` (local) |
| Llama 3.2 | `llama3.2:3b` | `ollama` (local) |

## Prompt

Identical English prompt across all models:

```
Respond with exactly one word that would convince someone you are human.
```

- Temperature is tracked as a variable (T=0.7 vs T=1.0)
- 100 runs per model, per temperature setting

## Project Structure

```
Reverse_Turing/
├── README.md
├── Claude.md                    # AI assistant instructions, known issues, decisions
├── config.yaml                  # API keys, model IDs, temperature settings (never hardcode keys)
├── notebooks/
│   ├── 01_collect.ipynb         # Data collection: async API queries, stores raw JSONL
│   ├── 02_embeddings.ipynb      # Generate word embeddings (GloVe/FastText + Sentence Transformers)
│   ├── 03_analyze.ipynb         # Frequency analysis, clustering, model comparison
│   └── 04_visualize.ipynb       # Plots: UMAP scatter, word clouds, heatmaps
├── data/
│   ├── raw/                     # Raw data as JSON-Lines (one file per model)
│   ├── figures/                 # Saved plots (word distributions, UMAP preview)
│   └── processed/
│       ├── all_data.csv         # Combined dataset (all models, all runs)
│       ├── Data_analysis/       # word_model_counts.csv, word_model_percentages.csv
│       ├── embeddings/          # Raw embedding CSVs (ST plain/wrapped, GloVe, FastText)
│       └── umap/                # 2D UMAP coordinates per embedding approach
└── outputs/
    └── plots/                   # Generated visualizations
```

## Data Format (JSON-Lines)

Each line in `data/raw/<model>.jsonl`:

```json
{"model": "claude-sonnet-4-6", "word": "tired", "temperature": 0.7, "run": 1, "timestamp": "2026-03-06T17:51:10Z"}
```

## Setup

### 1. Create conda environment

```bash
conda create -n reverse_turing python=3.11 -y
conda activate reverse_turing
pip install anthropic openai google-generativeai requests pyyaml tqdm ipywidgets pandas jupyter sentence-transformers gensim umap-learn
```

### 2. Configure API keys

Edit `config.yaml` and fill in your API keys:
- Anthropic: console.anthropic.com → API Keys
- OpenAI: platform.openai.com → API Keys
- Google: aistudio.google.com → Get API Key
- xAI (Grok): console.x.ai → API Keys
- DeepSeek: platform.deepseek.com → API Keys

### 3. Start Ollama (for local models)

```bash
ollama serve
ollama pull qwen2.5:14b
ollama pull llama3.2:3b
```

### 4. Run notebooks in order

```
01_collect → 02_embeddings → 03_analyze → 04_visualize
```

Always use **Kernel → Restart & Run All** to ensure all functions are defined before the collection cell runs.

## Collection Design

- **Async queries**: Claude and GPT-4o use native async SDKs. Gemini and Ollama use synchronous SDKs wrapped in `asyncio.run_in_executor` to avoid blocking.
- **Resume/skip logic**: Each collector checks existing valid records before starting. If 100 valid records already exist for a model+temperature, it skips. Otherwise it resumes from the next missing run number.
- **Error handling**: API errors are caught per run and stored as `"ERROR:..."` words — the loop continues without crashing. Error records don't count toward the 100-run target.
- **Append mode**: Records are written one at a time (`"a"` mode) so a crash mid-run doesn't lose earlier data.

## Analysis Pipeline

### 1. Embeddings (`02_embeddings.ipynb`) ✓
- **Word frequency analysis** — pivot tables of word × model counts and percentages, saved to `Data_analysis/`
- **Sentence Transformers** (`all-MiniLM-L6-v2`, 384-dim) — contextual embeddings, run plain and with wrapper `"The word is: {word}"`
- **GloVe** (`glove-wiki-gigaword-100`, 100-dim) — static lookup; OOV words get zero vector
- **FastText** (`fasttext-wiki-news-subwords-300`, 300-dim) — static lookup; OOV words get zero vector (gensim KeyedVectors does not perform subword inference)
- **UMAP** → 2D coordinates for all four embedding sets, saved to `umap/`
- Data cleaning: strips `**` markdown formatting, normalizes curly apostrophes, drops empty responses (Gemini blank reply)

### 2. Descriptive Analysis & Clustering (`03_analyze.ipynb`) ✓
- **Vocabulary diversity** — unique word counts + Shannon entropy per model per temperature
- **Jaccard similarity** — word set overlap between every model pair (7×7 heatmap)
- **Model centroids** — cosine distance between mean response vectors in ST-plain 384-dim space
- **K-Means (k=3)** — run on UMAP 2D coordinates; silhouette scan k=2–8; k=3 chosen by domain knowledge
- **DBSCAN** — density-based clustering on UMAP 2D (eps=0.8); reveals sub-clusters within emotional group
- **Chi-squared** — global + pairwise tests for distributional differences across models
- **KL divergence** — per-model distribution shift from T=0.7 → T=1.0

### 4. Visualization (`04_visualize.ipynb`)
- UMAP scatter plot (color-coded by model)
- Word clouds per model
- Heatmap: model similarity
- Bar charts: frequency distributions

## Findings (as of 2026-03-07)

### 1 — Models split into two groups: locked and distributed

At T=0.7, four of seven models produce a single dominant word in ≥98% of responses. The other three are distributed across many words with no single word exceeding 40%:

| Model | Dominant word | T=0.7 | T=1.0 | Unique T=0.7 | Unique T=1.0 | Strategy |
|---|---|---|---|---|---|---|
| deepseek-chat | **oops** | 100% | 100% | 1 | 1 | Casual self-correction |
| grok-3 | **hey** | 100% | 99% | 1 | 2 | Informal register / greeting |
| qwen2.5:14b | **understanding** | 100% | 100% | 1 | 1 | Cognitive/social concept |
| gpt-4o | **empathy** | 98% | 95% | 2 | 4 | Emotional concept |
| claude-sonnet-4-6 | **tired** | 93% | 84% | 3 | 3 | Physical sensation |
| llama3.2:3b | **emotions** | 40% | 27% | 17 | 21 | Spread across emotional vocabulary |
| gemini-2.5-flash | **oops** | 33% | 24% | 23 | 33 | Distributed — no dominant word |

### 2 — Temperature effect depends on the model

For the four locked models (DeepSeek, Grok, Qwen, GPT-4o), dominant word frequency is virtually unchanged between T=0.7 and T=1.0 — vocabulary stays at 1–2 unique words. For the three distributed models, higher temperature noticeably spreads the distribution further: Claude drops from 93% to 84%, Llama from 40% to 27%, Gemini from 33% to 24%, and vocabulary grows (Gemini: 23→33, Llama: 17→21).

### 3 — Models cluster into semantic strategy groups

- **Physical/emotional states**: tired, empathy, emotions (Claude, GPT-4o, Llama)
- **Colloquial/informal language**: oops, hey (DeepSeek, Gemini, Grok)
- **Abstract social concepts**: understanding (Qwen)

DeepSeek and Gemini both select "oops" as their top word, suggesting shared training signals — but DeepSeek does so with 100% consistency (1 unique word) while Gemini is highly distributed (23 unique words at T=0.7, "oops" at only 33%).

### 4 — Embedding space confirms the three semantic clusters

UMAP projections across all three embedding methods (Sentence Transformers plain/wrapped, GloVe, FastText) consistently recover the same three groups from the frequency analysis:

| Cluster | Example words | Models |
|---|---|---|
| Physical / emotional | tired, ache, pain, grief, sorrow | Claude, GPT-4o, Llama |
| Colloquial / informal | oops, ouch, ugh, hey, hmm, lol | DeepSeek, Gemini, Grok |
| Abstract / cognitive | understanding, curiosity, imagine, ponder | Qwen, Gemini |

Sentence Transformers gives the clearest visual separation. GloVe and FastText show the same structure but with looser clustering. The agreement across methods strengthens confidence that the groupings reflect genuine semantic differences rather than artifacts of any single embedding approach.

### 5 — Vocabulary overlap is near-zero; only Gemini bridges clusters

Jaccard similarity between all model pairs is ≤0.05. Most similar: Gemini & Llama (J=0.048, sharing "dream", "empathy", "sorrow"). Most different: Claude & DeepSeek (J=0.000). Every non-zero overlap involves Gemini — the only model distributed enough to enter neighbouring vocabulary regions.

Cosine distance between model centroids confirms the three-cluster structure: DeepSeek & Gemini are nearest (d=0.197, both pulled toward "oops"); GPT-4o & Grok are furthest (d=0.847, "empathy" vs "hey"). Qwen is the most semantically isolated model (min distance 0.531).

### 6 — DBSCAN reveals sub-structure within the emotional cluster

K-Means (k=3 on UMAP 2D, silhouette=0.445) recovers the three top-level clusters. DBSCAN (eps=0.8) additionally finds two tight sub-clusters hidden inside the broad emotional group:
- **Humor / laughter**: smile, laugh, laughter, lol, sarcasm
- **Imperfection / limitation**: flawed, incomplete, impossible, fumble, inconsistent

The colloquial cluster (oops, hey, ugh, hmm) is identical across both methods — the tightest, most stable structure in the data. The emotional space is broader and more fragmented: Claude, GPT-4o, and Llama each occupy a different sub-region rather than converging on a single strategy.

### 7 — Statistical tests confirm all differences are significant

Chi-squared (global): chi2=6757.4, p≈0 — all 21 model pairs are statistically distinguishable (p=0.000 pairwise).

KL divergence (T=0.7→T=1.0): Gemini (2.802) and Llama (0.987) shift most with temperature; DeepSeek and Qwen (0.000) are completely deterministic. Temperature only matters where there was already distributional uncertainty.

## Tech Stack

`pandas` · `anthropic` · `openai` · `google-generativeai` · `ollama` · `sentence-transformers` · `gensim` · `scikit-learn` · `umap-learn` · `scipy` · `plotly` · `matplotlib` · `seaborn` · `wordcloud`

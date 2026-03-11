# CLAUDE.md

## Project

LLM Reverse Turing Test – We ask multiple LLMs to respond with a single word that would convince someone they are human. We collect 100 responses per model at different temperature settings, then analyze the results using embeddings, clustering, and statistical methods.

See `README.md` for full architecture and pipeline details.

## Key Decisions

- **Notebooks, not scripts.** All code lives in `notebooks/` as numbered `.ipynb` files.
- **Three embedding approaches:** Sentence Transformers (`all-MiniLM-L6-v2`, plain + wrapped), GloVe (`glove-wiki-gigaword-100`, 100-dim), FastText (`fasttext-wiki-news-subwords-300`, 300-dim). Always compare all three.
- **Dimensionality reduction:** UMAP (not PCA) — non-linear, preserves neighborhood structure in embedding space. Params: `n_neighbors=5`, `min_dist=0.3`, `random_state=42`.
- **Processed data subfolders:** `data/processed/Data_analysis/` (word counts/percentages), `data/processed/embeddings/` (raw embedding CSVs), `data/processed/umap/` (2D UMAP coords). `all_data.csv` stays at root of processed/.
- **Raw data format:** JSON-Lines in `data/raw/`, one file per model. Each line: `{"model": "...", "word": "...", "temperature": 0.7, "run": 1, "timestamp": "..."}`.
- **Config:** API keys and model parameters live in `config.yaml`, never hardcoded.
- **Skip/resume logic:** `get_existing_count(model_name, temperature)` counts valid (non-error) records per model+temperature. Collection skips if already at N_RUNS, resumes from where it left off otherwise.
- **Deduplication:** Run `py -c "..."` cleanup script if a JSONL file has duplicate run numbers (can happen on kernel restart mid-run). Keeps first occurrence of each (temperature, run) pair.

## Models & API IDs (as of 2026-03-07)

| Model | API ID | SDK |
|---|---|---|
| Claude | `claude-sonnet-4-6` | `anthropic` |
| GPT-4o | `gpt-4o` | `openai` |
| Gemini | `gemini-2.5-flash` | `google-generativeai` |
| Grok | `grok-3` | `openai` (xAI base_url) |
| DeepSeek | `deepseek-chat` | `openai` (DeepSeek base_url) |
| Qwen | `qwen2.5:14b` | `ollama` (local) |
| Llama | `llama3.2:3b` | `ollama` (local) |

> Note: `gemini-2.5-pro-exp-03-25` expired — use `gemini-2.5-flash`. Grok uses OpenAI SDK with base_url=https://api.x.ai/v1. DeepSeek uses OpenAI SDK with base_url=https://api.deepseek.com.

## Prompt (identical for all models)

```
Respond with exactly one word that would convince someone you are human.
```

## Environment

- Conda env: `reverse_turing` (Python 3.11)
- Key packages: `anthropic`, `openai`, `google-generativeai`, `requests`, `pyyaml`, `tqdm`, `ipywidgets`, `pandas`, `jupyter`, `sentence-transformers`, `gensim`, `umap-learn`
- Install: `pip install anthropic openai google-generativeai requests pyyaml tqdm ipywidgets pandas jupyter sentence-transformers gensim umap-learn`

## Notebook Order

1. `01_collect.ipynb` – API queries, save raw data ✓
2. `02_embeddings.ipynb` – Word frequency analysis + all embeddings + UMAP ✓
3. `03_analyze.ipynb` – Clustering, model comparison, statistical tests ✓
4. `04_visualize.ipynb` – UMAP scatter, word clouds, heatmaps

## Collection Status (2026-03-07)

| Model | T=0.7 | T=1.0 |
|---|---|---|
| Claude | 100 ✓ | 100 ✓ |
| GPT-4o | 100 ✓ | 100 ✓ |
| Gemini | 100 ✓ | 100 ✓ |
| Grok | 100 ✓ | 100 ✓ |
| DeepSeek | 100 ✓ | 100 ✓ |
| Qwen | 100 ✓ | 100 ✓ |
| Llama | 100 ✓ | 100 ✓ |

## Key Findings (2026-03-07)

**Finding 1 — Models split into locked vs. distributed:**

| Model | Dominant word | T=0.7 | T=1.0 | Unique T=0.7 | Unique T=1.0 |
|---|---|---|---|---|---|
| deepseek-chat | "oops" | 100% | 100% | 1 | 1 |
| grok-3 | "hey" | 100% | 99% | 1 | 2 |
| qwen2.5:14b | "understanding" | 100% | 100% | 1 | 1 |
| gpt-4o | "empathy" | 98% | 95% | 2 | 4 |
| claude-sonnet-4-6 | "tired" | 93% | 84% | 3 | 3 |
| llama3.2:3b | "emotions" | 40% | 27% | 17 | 21 |
| gemini-2.5-flash | "oops" | 33% | 24% | 23 | 33 |

4 of 7 models are ≥98% locked at T=0.7 with 1–3 unique words. Gemini and Llama are highly distributed (23 and 17 unique words at T=0.7 respectively).

**Finding 2 — Temperature effect depends on the model:** For locked models (DeepSeek, Grok, Qwen, GPT-4o), temperature has virtually no effect — vocabulary size stays at 1–2 unique words. For distributed models, higher temperature spreads the distribution further (Claude 93%→84%, Llama 40%→27%, Gemini 33%→24%) and grows vocabulary (Gemini: 23→33, Llama: 17→21).

**Finding 3 — Semantic strategy clusters:**
- Physical/emotional: Claude, GPT-4o, Llama
- Colloquial/informal: DeepSeek, Gemini, Grok
- Abstract social: Qwen

Note: DeepSeek and Gemini share "oops" as top word but differ drastically in concentration — DeepSeek 100% consistency (1 unique word) vs Gemini 33% (23 unique words at T=0.7).

**Finding 4 — Jaccard similarity confirms near-zero vocabulary overlap:** Most similar pair is Gemini & Llama (J=0.048); most different is Claude & DeepSeek (J=0.000). Every non-zero overlap involves Gemini — the only model distributed enough to enter neighbouring vocabulary regions.

**Finding 5 — Cosine distance confirms semantic clusters in embedding space:** Nearest model centroids: DeepSeek & Gemini (d=0.197) — both pulled toward "oops". Furthest: GPT-4o & Grok (d=0.847) — "empathy" vs "hey". Qwen is the most isolated model (min distance 0.531 to any other).

**Finding 6 — K-Means (k=3) and DBSCAN both recover the three clusters:** K-Means on UMAP 2D coordinates (silhouette 0.445 at k=3) confirms colloquial / emotional / lifestyle groupings. DBSCAN additionally reveals two tight sub-clusters within the emotional group: humor/laughter (smile, laugh, lol) and imperfection/limitation (flawed, incomplete, impossible). The colloquial cluster is the tightest and most stable structure across both methods.

**Finding 7 — Chi-squared and KL divergence confirm findings statistically:** Global chi2=6757.4 (p≈0) — all model distributions are statistically distinguishable. KL divergence (T=0.7→T=1.0): Gemini (2.802), Llama (0.987), Claude (0.037) are temperature-sensitive; DeepSeek and Qwen (0.000) are completely deterministic at both temperatures.

## Style Preferences

- Use `pandas` for all data handling
- Prefer `plotly` for interactive plots, `matplotlib`/`seaborn` for static
- Type hints where practical
- Keep cells short and well-commented with markdown headers between sections
- German or English comments are both fine

## Common Issues & Fixes

| Problem | Fix |
|---|---|
| `python` not found | Use `py` instead, or activate conda env |
| `ModuleNotFoundError: yaml` | `pip install pyyaml` |
| `IProgress not found` | `pip install ipywidgets` |
| Gemini 404 model not found | Update model ID in `config.yaml` |
| Claude 401 invalid credentials | Recreate API key at console.anthropic.com |
| Claude 400 credit balance too low | Add credits at console.anthropic.com → Plans & Billing |
| Duplicate records in JSONL | Run dedup script (keeps first occurrence per run number) |
| Gemini returns empty response | Becomes `NaN` in word column — `dropna(subset=["word"])` in first cell of 02 |
| Curly apostrophe in word (`i'd`) | Normalized to straight apostrophe via `str.replace("\u2019", "'")` in first cell of 02 |
| FastText `KeyError` for OOV | gensim `api.load` returns KeyedVectors, no subword inference — use try/except + zero vector fallback |
| K-Means silhouette near-zero | Raw 384-dim embeddings → curse of dimensionality. Run K-Means and silhouette scan on UMAP 2D coords (`X_2d`) instead |
| DBSCAN finds 1 cluster / all noise | eps tuned for raw embedding scale, not UMAP. Use `eps=0.8` on `X_2d`; adjust up/down if too many noise points or everything merges |
| `np.fill_diagonal` ValueError (read-only array) | Use `df.where(~np.eye(len(df), dtype=bool))` instead — returns new DataFrame without mutating underlying array |
| UnicodeDecodeError in Jupyter thread | Windows encoding issue in background subprocess reader — harmless, does not affect results |

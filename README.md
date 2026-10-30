# Group19_IDC409-Faculty-profiles-web-scraper-
Faculty profiles web scraper
(Please open the code in Google Colab or the graphs will not be visible)
# Faculty Profile Explorer

A reproducible Data Science and AI course project (IDC409). It collects the **public** faculty profile pages of IISER Mohali,
stores them in SQLite, explores them, groups faculty by research vocabulary (**TF-IDF → Truncated SVD → K-Means**) and offers a
**similarity search** ("zebrafish regeneration" → the most similar faculty profiles).

> Everything reported here comes from running the code on the real data. Nothing was invented. Where the results are weak, the
> report says so (see *Limitations*).

## Data source and ethics

* Source: the official IISER Mohali people directory (`https://www.iisermohali.ac.in/people`) and the linked faculty profile pages.
* The scraper reads `robots.txt` **before every run** and stops if the pages are not allowed. It identifies itself with a
  User-Agent that contains your contact e-mail, waits 2 seconds between requests and saves each page so nothing is downloaded twice.
* **E-mail addresses are never stored** (data minimisation). Only public research text, designation and department are used.
* Use the data for coursework only. Check the website's terms before redistributing it.
* Set your contact e-mail through an environment variable; it is not written into the repository:
  `export FPE_CONTACT_EMAIL="you@example.com"`

## Data at a glance

| Item | Value |
|---|---|
| Faculty profiles | 122 (DBS 28, DPS 28, DCS 25, DMS 21, EES 12, HSS 8) |
| Designations | 38 Professors, 40 Associate Professors, 44 Assistant Professors |
| Profiles with usable research text | 122 (113 `ok`, 9 `very_short` = fewer than 15 words) |
| TF-IDF search matrix (`min_df=1`) | 122 × 3,291, 98.41% sparse |
| TF-IDF cluster matrix (`min_df=2`) | 122 × 1,142, 96.97% sparse |

## Method in one picture

```
directory + profile pages ──► faculty_raw.csv ──► faculty_cleaned.csv ──► faculty.db (SQLite, CRUD)
                                                                             │
                              EDA figures ◄──────────────────────────────────┤
                                                                             ▼
                      profile_text ──► TF-IDF ──► Truncated SVD (k = 8) ──► K-Means (K = 9) ──► faculty_clusters.csv
                                          └────────────► cosine similarity search (full TF-IDF space)
```

| Step | Method | Why |
|---|---|---|
| Text → numbers | TF-IDF, sub-linear tf, L2 rows | rare, topic-specific words get more weight than common ones |
| Feature reduction | Truncated SVD (not PCA) | works directly on sparse matrices; PCA would centre the data and make it dense |
| Re-normalisation | rows scaled to length 1 | Euclidean K-Means then behaves like cosine similarity |
| Clustering | K-Means, 50 random starts, `random_state=42` | simple, explainable, centroids give keywords |
| Evaluation | silhouette (reduced and full space), stability, shuffled-label baseline, sensitivity to k | one score is not enough |
| Search | cosine similarity | ignores profile length |

## Results (from `scripts/07_evaluate.py` and the Day 9 notebook)

* **Chosen by a written rule:** K between 4 and 10, no cluster below 5 people, at least 8 SVD components, highest silhouette → **k = 8, K = 9**.
  The silhouette is flat for K = 7–10 (0.413–0.438), so K = 9 is *one of several comparable solutions*.
* **Silhouette:** 0.438 in the 8-D reduced space, **0.060** in the full TF-IDF space (shuffled labels: −0.110 and −0.024). Davies–Bouldin 0.882.
* **Stability:** mean ARI 0.942 for 80% subsamples (worst 0.752); 0.827 for single random starts (worst 0.694).
* **Sensitivity:** with 10, 15 or 20 SVD components the grouping agrees only moderately (ARI 0.656, 0.561, 0.645), so the result is a
  coarse, dimension-dependent map, not a unique partition. Three clusters persist at both 10 and 15 components (Jaccard ≥ 0.7).
* **Departments (sanity check, not ground truth):** ARI 0.503, NMI 0.694, purity 82.8%.

| C | n | Name (chosen after reading keywords and members) | Mean silhouette |
|---|---|---|---|
| 0 | 21 | Organic, inorganic and materials chemistry, including catalysis | 0.476 |
| 1 | 16 | Mixed: humanities and social sciences, hydrology/remote sensing, evolutionary ecology | 0.363 |
| 2 | 10 | Infection, host-pathogen interactions and immunity | 0.574 |
| 3 | 13 | Operator theory, group theory and algebra | 0.601 |
| 4 | 7 | Soft matter, statistical and biological physics | 0.648 |
| 5 | 13 | Quantum information, quantum thermodynamics, optics and gravity | 0.434 |
| 6 | 17 | Genetics, genomics, cell/developmental and plant biology | 0.385 |
| 7 | 7 | Algebraic geometry, commutative algebra and geometry | 0.727 |
| 8 | 18 | Mixed residual group: modelling, spectroscopy, atmospheric and observational sciences | 0.125 |

## Similarity search

```bash
python scripts/08_search.py "bacterial infection"
python scripts/08_search.py "zebrafish regeneration"
```
`tfidf` mode (default) matches exact words over all 3,291 terms. If fewer than five profiles contain the query words, the list is filled
with the people most similar to the best match and **clearly labelled "related to <name>"**. `--mode lsa` searches the coarse 8-D topic space.

## Repository layout

```
src/faculty_explorer/   config, scraping, cleaning, database, eda, features, clustering, evaluation, search, cluster_names
scripts/                01_scrape … 08_search, run_pipeline.py (one script per project day)
notebooks/              Colab notebook (Days 1–8) and Day 9 notebook (evaluation and search)
data/                   raw/  processed/  db/  features/   (see "Data files" below)
models/                 fitted vectorizers, SVD pipeline and K-Means (re-created by the scripts)
reports/figures/        01–11 PNG figures
docs/                   final report, viva preparation, presentation outline
tests/                  offline tests on SYNTHETIC data (no real faculty data, no internet)
```

## Reproduce it

```bash
python -m venv .venv && source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
export FPE_CONTACT_EMAIL="you@example.com"

python scripts/01_scrape.py --pilot        # optional: Day 2 (needs internet; a few minutes for the full run because of the 2-second delay)
python scripts/run_pipeline.py             # Days 3-9 from data/raw/faculty_raw.csv (clean, DB, EDA, TF-IDF, clusters, evaluation)
python scripts/08_search.py "quantum information"
```
Or run the notebooks in Google Colab (`notebooks/`).

**Data files.** The data folders are empty placeholders. To work without scraping, copy `faculty_raw.csv` (from your Drive `data_raw`) into
`data/raw/`, then run `python scripts/run_pipeline.py`. Re-running reproduces the saved results when the same library versions and
`random_state=42` are used; cluster *numbers* (and therefore `cluster_names.py`) can change if you change K, k or the seed.

## Tests

```bash
pip install -r requirements-dev.txt
pytest -q
```
The tests use invented people and invented text. They check the parsers, the CRUD round trip, TF-IDF row lengths, the K/k rule,
reproducibility, the evaluation code and the search fallback. They prove the code works, not that any result is correct.

## Limitations (also in the report)

* 122 short, self-written profiles: the results are exploratory.
* Silhouette is a heuristic; k and K were selected on the same data they were scored on.
* Bag-of-words ignores word meaning: single words such as "matter" or "metal" pull a few people into the wrong cluster, and high-energy
  physicists are spread over three clusters. There is no stemming, so "regeneration" does not match "regenerative".
* Cosine similarity rewards short profiles that consist only of the query words.
* The clusters describe similarity of research vocabulary, not research quality.

## credits

Built with pandas, NumPy, SciPy, scikit-learn,
matplotlib, requests, BeautifulSoup and SQLite.

## Group 19
Kartik Saini(MS24144)
Niharika Saini(Ms24101)
Tuneer Das(Ms24030)

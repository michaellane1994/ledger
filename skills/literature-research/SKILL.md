---
name: literature-research
description: Search academic literature and perform deep research reviews. Use when user asks to search PubMed, find academic papers, or do literature reviews.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Literature Research

## Goal
Search academic databases and perform comprehensive literature reviews. Two scripts handle different use cases: a broad search across PubMed and ClinicalTrials.gov, and a deep review pipeline that downloads full texts, matches CT.gov results, and filters by relevance.

## Scripts
- `./scripts/pubmed_literature_search.py` - Broad search of PubMed and ClinicalTrials.gov
- `./scripts/literature_deep_review.py` - Multi-step deep review pipeline with full text retrieval

---

## Script 1: pubmed_literature_search.py

### Usage
```bash
python3 ./scripts/pubmed_literature_search.py \
  --query "machine learning cancer diagnosis" \
  --min_year 2015 \
  --max_pubmed 2000 \
  --max_trials 500 \
  --output .tmp/papers.json
```

### Parameters
| Argument | Default | Description |
|----------|---------|-------------|
| `--query` | (required) | PubMed search query string |
| `--min_year` | 1994 | Earliest publication year to include |
| `--max_year` | current | Latest publication year to include |
| `--max_pubmed` | 2000 | Max results from PubMed |
| `--max_trials` | 500 | Max results from ClinicalTrials.gov |
| `--output` | `.tmp/literature_search_TIMESTAMP.json` | Output file path |
| `--pubmed_only` | false | Skip ClinicalTrials.gov |
| `--trials_only` | false | Skip PubMed |

### Filtering Options

Use PubMed's built-in query syntax for precision filtering. The query string is passed directly to the NCBI E-utilities API.

Filter by publication type:
```
"machine learning" AND randomized controlled trial[pt]
"machine learning" AND meta-analysis[pt]
"machine learning" AND systematic review[pt]
```

Filter by field:
```
"cancer diagnosis"[Title/Abstract] AND "deep learning"[Title]
```

Limit to recent years (alternative to `--min_year`):
```
"sepsis" AND 2020:2024[pdat]
```

Combine terms:
```
(HRT OR "hormone therapy" OR estrogen) AND (menopause OR postmenopausal) AND (hot flash OR vasomotor)
```

### Output Format
A JSON array of objects. Each object includes:

| Field | Description |
|-------|-------------|
| `source` | `"PubMed"` or `"ClinicalTrials.gov"` |
| `pmid` | PubMed ID (empty for CT.gov entries) |
| `nct_id` | ClinicalTrials.gov NCT ID (empty for PubMed) |
| `doi` | DOI |
| `title` | Article/study title |
| `authors` | Semicolon-separated (first 10 authors) |
| `journal` | Full journal name |
| `pub_date` | Publication date string |
| `pub_year` | Year only |
| `abstract` | Full abstract text |
| `pub_types` | Semicolon-separated publication types |
| `mesh_terms` | MeSH descriptors (PubMed) or conditions (CT.gov) |
| `keywords` | Author-supplied keywords |
| `study_type` | Classified type (see below) |
| `url` | Direct link to the record |

Study types are auto-classified from publication type tags and abstract text:
`RCT`, `Clinical Trial`, `Meta-Analysis`, `Systematic Review`, `Review`, `Cohort Study`, `Case-Control`, `Cross-Sectional`, `Retrospective Study`, `Prospective Study`, `Observational`, `Other`

Results are sorted by year descending. PubMed results come first; ClinicalTrials.gov entries are appended after deduplication by title.

At the end of output, the script prints a summary by study type and writes `__OUTPUT_FILE__:/path/to/file` for easy piping.

### No API Key Required
The script uses NCBI E-utilities which are free and do not require a key for low-volume use (up to 3 requests/second). For high-volume searches, set `NCBI_API_KEY` in `.env` to raise the limit to 10 requests/second.

---

## Script 2: literature_deep_review.py

A 10-step pipeline for systematic reviews that goes beyond metadata to retrieve full texts and extract intervention details.

### Usage
```bash
# Run with default menopause/HRT query
python3 ./scripts/literature_deep_review.py

# Custom query with options
python3 ./scripts/literature_deep_review.py \
  --query "fezolinetant menopause hot flashes randomized" \
  --min_year 2010 \
  --max_pubmed 2000 \
  --output_dir .tmp/my_review \
  --skip_unpaywall
```

### Parameters
| Argument | Default | Description |
|----------|---------|-------------|
| `--query` | Built-in menopause query | Custom PubMed query |
| `--min_year` | 1994 | Earliest year |
| `--max_pubmed` | 2000 | Max PubMed results |
| `--max_trials` | 500 | Max CT.gov results |
| `--max_full_texts` | 500 | Max PMC full texts to download |
| `--output_dir` | `.tmp/literature_review` | Output directory |
| `--skip_full_text` | false | Skip PMC full text download |
| `--skip_unpaywall` | false | Skip Unpaywall open-access check |

### Pipeline Steps
1. **PubMed search** - Retrieves PMIDs for the query
2. **Article detail fetch** - Gets abstracts, authors, MeSH terms, DOIs, PMC IDs
3. **PMC availability check** - Identifies which articles have free full text in PubMed Central
4. **PMC full text download** - Downloads XML full texts; saved to `output_dir/full_texts/`
5. **Unpaywall check** - For articles without PMC full text, checks if an open-access version exists
6. **ClinicalTrials.gov search** - Searches CT.gov with a companion query
7. **CT.gov article matching** - Scans abstracts for NCT IDs and fetches trial results data
8. **Intervention extraction** - Regex-based extraction of treatment names and comparators
9. **Relevance filtering** - Scores articles as High/Moderate/Low relevance
10. **Compilation** - Merges PubMed and CT.gov results, sorted by relevance then year

### Output Files
All output goes to `--output_dir` (default `.tmp/literature_review/`):

| File | Contents |
|------|----------|
| `deep_review_TIMESTAMP.json` | Full compiled dataset, all results |
| `highly_relevant_studies.json` | Subset: High relevance only (RCTs/trials with direct comparisons) |
| `full_texts/PMC*.xml` | Downloaded PMC full text XML files |

### How Relevance Filtering Works
The filter logic is hardcoded for HRT vs SSRI/SNRI/fezolinetant comparisons in menopause, but the categories apply generally:

- **High - Head-to-head comparison:** RCT or Clinical Trial that mentions both HRT and SSRI/SNRI (or fezolinetant)
- **High - Placebo-controlled:** RCT or Clinical Trial with a relevant intervention vs placebo
- **Moderate - Relevant intervention:** Trial mentioning any relevant intervention but no direct comparison
- **Moderate - Observational:** Non-trial study mentioning relevant interventions
- **Low:** Everything else

To use this pipeline for a different topic, modify the intervention/comparator regex patterns and the `filter_relevant_studies` method in the script.

### Environment
Create `.env` with:
```
UNPAYWALL_EMAIL=your@email.com
```

Unpaywall requires a valid email address for rate-limiting. Without `UNPAYWALL_EMAIL` set, it defaults to `research@example.com` which still works but is poor practice.

### Troubleshooting

**Script runs but finds 0 results:** Check your query syntax. Test it directly in the PubMed web interface at pubmed.ncbi.nlm.nih.gov first.

**NCBI rate limit errors (HTTP 429):** The script sleeps 0.35s between batches. If you still hit limits, set `NCBI_API_KEY` in `.env` to increase the allowed rate.

**PMC full text download is slow:** Use `--skip_full_text` to skip it. Full texts are only needed if you plan to feed the XML content to an AI for analysis.

**Unpaywall returns no results:** This is normal for paywalled journals. Use `--skip_unpaywall` if you only need abstract-level data.

**Memory issues with large result sets:** Reduce `--max_pubmed` and `--max_trials`. 2000 PubMed + 500 CT.gov results is the practical upper limit before the JSON becomes unwieldy.

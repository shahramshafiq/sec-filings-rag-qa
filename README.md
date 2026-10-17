<div align="center">

# SEC Filings RAG QA

A question answering service over SEC 10-K filings that cites its sources and declines to answer when the evidence is weak.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-REST%20API-009688?style=flat-square&logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-Corrective%20RAG-1C3C3C?style=flat-square&logo=langchain&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-gpt--4.1--mini-412991?style=flat-square&logo=openai&logoColor=white)

</div>

## Overview

Annual filings are long, repeat the same line items under slightly different names, and look alike across companies, so a single vector search often pulls the wrong figure. This service (titled "Ledger" in the FastAPI app) answers questions about the 10-K filings of Apple, Microsoft, Walmart and JPMorgan Chase for fiscal years 2022 and 2023. It runs a corrective RAG pipeline built as a LangGraph state machine: filter by company and year, retrieve, grade whether the evidence is sufficient, rewrite the query if it is not, and abstain if it still falls short. Every factual claim in an answer cites the retrieved chunk it came from, and each design choice was kept or dropped based on a run against a 25-question golden set.

## Key Features

- **Corrective RAG graph**: `app/retrieval/corrective_rag.py` wires retrieve, grade, generate, rewrite and abstain nodes in LangGraph, with at most 2 query rewrites before the service abstains.
- **Abstention**: when the grader judges the evidence insufficient after the rewrites, `POST /query` returns `"abstained": true` and a fixed insufficient-evidence message instead of a guess.
- **Claim-level citations**: each chunk gets a numeric id, the answer cites ids such as `[3]`, and `verify_citations()` reports any cited id that was never retrieved in `invalid_citations`.
- **Metadata filtering**: the company and fiscal year named in the question are turned into a vector-store filter, with negation handling ("fiscal year 2022, not fiscal year 2023"). Known companies are read from the database, not hardcoded.
- **Per-entity retrieval**: questions that span several companies or years are decomposed into one standalone question per entity, and each entity gets its own top-30 search so a larger filing cannot crowd out the others.
- **Structure-aware chunking**: 10-K HTML is split on "Item N." headings, tables stay whole with a "Table reporting: ..." caption, chunks are capped at 512 tokens, and page numbers are recovered from running page footers where the filing has them.
- **Prompt-injection defense**: retrieved text is wrapped in `<evidence>` tags and the prompt tells the model to treat it as data. `debug/prompt_injection_test.py` plants instructions in body text and in a table cell to test this.
- **Evaluation harness**: `eval/run_harness.py` scores recall@5, correctness, faithfulness, latency and cost for every question and appends one row per question per run to `eval/results.csv`.

## Tech Stack

| Area | Technology |
|------|------------|
| Language | Python 3.12 |
| API | FastAPI, Uvicorn, Server-Sent Events for ingestion progress |
| Orchestration | LangGraph, LangChain |
| Models | OpenAI `gpt-4.1-mini` (default) and `text-embedding-3-small` embeddings |
| Vector store | PostgreSQL with pgvector (`pgvector/pgvector:pg16`) through `langchain-postgres` and psycopg 3 |
| Parsing and chunking | BeautifulSoup4, tiktoken (`cl100k_base`) |
| Data source | `sec-edgar-downloader` (SEC EDGAR) |
| Retrieval experiments | sentence-transformers cross-encoder reranker, Postgres full-text search with reciprocal rank fusion |
| Testing | pytest |

## Project Structure

```text
app/
  main.py              FastAPI app, /health and /health/ready, startup DB check
  config.py            Settings loaded from .env (pydantic-settings)
  routes/              POST /query, POST /documents, SSE ingestion stream
  parsing/             10-K HTML to sections, paragraphs, tables and page markers
  chunking/            structure_aware.py (used), agentic.py (tested alternative)
  retrieval/           corrective_rag.py, metadata_filter.py, bm25.py, fusion.py
  reranker/            Cross-encoder reranker experiment
  services/            Ingestion pipeline and the single-pass baseline query
  vectorstore/         PGVector store factory
  utils/               Cost calculation and logging setup
eval/                  run_harness.py and results.csv (one row per question per run)
data/golden_dataset.json   25 hand-written questions with expected answers
tests/                 Citation verification and metadata filter tests
debug/                 Investigation, prompt-injection and citation-induction scripts
docs/NOTES.md          Measurement log: what was tried, kept and rejected
download_filings.py    Downloads the 8 filings from SEC EDGAR
```

## Getting Started

### Prerequisites

- Python 3.12
- Docker (for PostgreSQL with pgvector)
- An OpenAI API key

The commands below use the Windows virtual environment layout (`venv/Scripts/...`), matching the project's scripts.

### Installation

```bash
git clone https://github.com/shahramshafiq/sec-filings-rag-qa.git
cd sec-filings-rag-qa
python -m venv venv
venv/Scripts/pip install -r requirements.txt
venv/Scripts/pip install uvicorn python-multipart pytest
cp .env.example .env
```

`requirements.txt` does not list `uvicorn`, `python-multipart` or `pytest`, so they are installed separately. Then edit `.env`. The code reads these environment variables:

| Variable | Required | Default |
|----------|----------|---------|
| `OPENAI_API_KEY` | yes | none |
| `DATABASE_URL` | yes | none (see `.env.example` for the psycopg URL format) |
| `OPENAI_MODEL` | no | `gpt-4.1-mini` |
| `EMBEDDING_MODEL` | no | `text-embedding-3-small` |
| `INPUT_PRICE` / `OUTPUT_PRICE` | no | `0.40` / `1.60` (USD per 1M tokens, used for cost reporting) |

### Run

1. Start PostgreSQL with pgvector and enable the extension once:

   ```bash
   docker run --name ledger-postgres -e POSTGRES_PASSWORD=devpassword -e POSTGRES_DB=ledger -p 5432:5432 -d pgvector/pgvector:pg16
   docker exec -it ledger-postgres psql -U postgres -d ledger -c "CREATE EXTENSION IF NOT EXISTS vector;"
   ```

   Use the same password in `DATABASE_URL`.

2. Download the 8 filings (about 200 MB, not committed to the repo). SEC EDGAR asks for a contact name and email on every request, so set your own in `download_filings.py` first:

   ```bash
   venv/Scripts/python.exe download_filings.py
   ```

3. Ingest the filings into the vector store:

   ```bash
   venv/Scripts/python.exe -m app.ingest_all
   ```

4. Start the API:

   ```bash
   venv/Scripts/uvicorn app.main:app --reload --port 8000
   ```

## Usage

Ask a question:

```bash
curl -X POST http://localhost:8000/query -H "Content-Type: application/json" \
  -d '{"question": "What was Apple'"'"'s net income for fiscal year 2023?"}'
```

The response contains `answer`, `abstained`, `invalid_citations`, `chunks_used` (company, ticker, fiscal year, form type, section, page where known, and the chunk text), `latency_seconds`, `input_tokens` and `output_tokens`.

Other endpoints:

- `GET /health` and `GET /health/ready` (the second returns 503 if Postgres is unreachable).
- `POST /documents` takes a multipart upload with a `file` (.html or .htm) and the fields `company`, `ticker`, `fiscal_year` (for example `FY2023`) and optional `form_type` (default `10-K`). It returns a `job_id` and ingests in the background.
- `GET /documents/{job_id}/stream` streams ingestion status as Server-Sent Events until the job completes or fails.

Run the evaluation harness (results are appended to `eval/results.csv`):

```bash
venv/Scripts/python.exe -m eval.run_harness <run_label> <collection_name> [--rerank] [--filter] [--hybrid] [--corrective]
venv/Scripts/python.exe -m eval.run_harness corrective ledger_chunks --corrective
```

Results recorded in `eval/results.csv`, graded by an LLM judge against the golden answers (25 questions each):

| Configuration | Correct | Faithful | Avg latency | Total cost |
|---------------|---------|----------|-------------|------------|
| Baseline (structure-aware chunks, top-5 vector search) | 11/25 | 21/25 | 2.66 s | $0.017 |
| Agentic chunking | 12/25 | 22/25 | 2.40 s | $0.017 |
| Cross-encoder reranker | 10/25 | 23/25 | 7.55 s | $0.019 |
| Hybrid (vector + Postgres full-text) | 12/25 | 21/25 | 2.54 s | $0.015 |
| Metadata filtering | 13/25 | 21/25 | 2.23 s | $0.015 |
| Corrective RAG (final pipeline) | 25/25 | 25/25 | 8.59 s | $0.337 |

`docs/NOTES.md` explains what each experiment showed and why it was kept or rejected.

## Testing

```bash
venv/Scripts/pytest tests/ -v
```

There are 10 tests. The 6 in `tests/test_citation_verification.py` check citation-id verification. The 4 in `tests/test_metadata_filter.py` check that company and year filters are extracted correctly and that a filtered search never returns another company's chunks, so they need the running Postgres container with the corpus ingested. Both files need a configured `.env`, because settings are loaded on import.

## Limitations and Roadmap

- The golden set is small (25 questions, 4 companies, 2 fiscal years) and graded by an LLM judge, whose verdict flipped between runs on two close cross-document questions. Read the 25/25 as a measured result on this corpus, not a general accuracy figure.
- Corrective RAG is slower and costs more per question than single-pass retrieval, and per-entity retrieval runs sequentially. Caching the decomposition step and running searches concurrently are noted in `docs/NOTES.md` as not yet built.
- Ingestion job state is kept in memory, which suits a single process, and `app/ingest_all.py` uses Windows-style file paths.
- Page numbers are extracted only for Apple and JPMorgan filings; Microsoft and Walmart chunks have no page. Recursive and hierarchical chunking and a chunk-size sweep were left out of scope.

## Author

**Shahram Shafiq** - [GitHub](https://github.com/shahramshafiq) | [LinkedIn](https://www.linkedin.com/in/shahramshafiq/)

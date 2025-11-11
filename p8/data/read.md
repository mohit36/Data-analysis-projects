# Goal
Implement a production-safe ingestion pipeline that reads our **sitemap index** (e.g., `https://<org-domain>/sitemap_index.xml`), expands to all child sitemaps and URLs, fetches pages, **cleans text**, **chunks**, **embeds using embeddingGemma**, and **upserts vectors into Qdrant** so NLWeb can answer queries grounded on this content.

We already run NLWeb with:
- LLM: **Ollama** → `llama3.2:3b`
- Embeddings: **embeddingGemma**
- Vector DB: **Qdrant (local)**

**Important:** Qdrant is just the vector store; embeddings are computed by our code using embeddingGemma and then stored.

---

## Requirements (what Copilot should generate)
1. **New module & CLI**
   - Create: `nlweb/ingest/sitemap_ingest.py`
   - Provide a CLI entrypoint:
     ```bash
     python -m nlweb.ingest.sitemap_ingest --sitemap https://<org-domain>/sitemap_index.xml \
       --qdrant-url http://127.0.0.1:6333 \
       --collection nlweb_docs \
       --delay 1.2 \
       --max-pages 10 \
       --include '^https://<org-domain>/au/'
     ```
   - Flags:
     - `--sitemap` (str): sitemap or sitemap_index URL
     - `--qdrant-url` (str, default `http://127.0.0.1:6333`)
     - `--collection` (str, default `nlweb_docs`)
     - `--api-key` (optional)
     - `--delay` (float, polite crawl delay; default 1.0–1.5s)
     - `--include/--exclude` (regex list filters)
     - `--max-pages`/`--max-chunks` (ints; for testing)
     - `--state` (path for incremental state json; default `nlweb_sitemap_state.json`)
     - `--header` (repeatable) e.g. `X-Index-Token=abc123` (for WAF allow)

2. **Sitemap expansion**
   - Parse **sitemap index** → recurse into child sitemaps → yield `(url, lastmod)`.
   - Deduplicate URLs; stream results.
   - Respect `<lastmod>` for incremental decisions.

3. **Fetch + clean**
   - `requests.get` with headers:
     - `User-Agent: NLWebBot/1.0 (+org-internal)`
     - plus any `--header` values
   - Polite: throttle by `--delay`, backoff on 429/503.
   - Extract main text with fallback chain:
     1) `trafilatura.extract`
     2) `readability-lxml` summary → strip HTML
     3) BeautifulSoup fallback (remove `script/style/noscript/iframe/svg`)
   - Skip docs with < 200 chars of text.

4. **Chunking**
   - Word-based chunker: default size **450** words, overlap **60**.
   - Return list[str] chunks; deterministic, no tokenizers required.

5. **Embeddings: embeddingGemma**
   - Use the repo’s existing embeddingGemma wiring (do **not** switch to OpenAI).
   - Provide a small abstraction `embed_texts(texts: List[str]) -> List[List[float]]`.
   - Batch size 32–48. Keep input order stable.

6. **Qdrant upsert**
   - Auto-create collection with correct **dimension**:
     - Probe once via `embed_texts(["__probe__"])` → `dim = len(vector)`
     - If collection absent → create with `Distance.COSINE`, `size=dim`.
   - Stable point IDs: `md5(url + "|" + chunk)` truncated to int.
   - Payload must include: `url`, `title`, `chunk`, `lastmod`, `source="sitemap_crawl"`, `ingested_at` (epoch).
   - Upsert in batches; continue on per-page errors (log & skip).

7. **Incremental updates**
   - Maintain a JSON state file (default `nlweb_sitemap_state.json`):
     - `pages[url] = { "lastmod": "...", "hash": "sha1(html)" }`
   - If both `lastmod` unchanged **and** `hash` unchanged → skip re-embedding.

8. **Config alignment with NLWeb**
   - No change to NLWeb retrieval code.
   - Ensure `.env` or project config already points NLWeb to Qdrant:
     ```
     VECTOR_DB=Qdrant
     QDRANT_URL=http://127.0.0.1:6333
     QDRANT_COLLECTION=nlweb_docs
     LLM=ollama:llama3.2:3b
     EMBEDDINGS=embeddingGemma
     ```
   - This ingest tool only **adds** content; it does not change serving path.

9. **Testing**
   - Add tests under `tests/ingest/test_sitemap_ingest.py`:
     - unit test: chunker windowing correctness (size/overlap)
     - unit test: `ensure_collection` creates with right dim (mock `embed_texts`, mock Qdrant)
     - unit test: sitemap parser handles both sitemap index and urlset
   - Provide a minimal `README_ingest.md` with usage examples.

10. **Docs**
    - Add `docs/ingestion/sitemap.md` explaining:
      - required env vars (if any)
      - CLI examples
      - WAF header usage (`--header X-Index-Token=...`)
      - polite crawling guidelines
      - how to schedule (cron/Task Scheduler)

---

## Acceptance Criteria
- ✅ Running the CLI on our sitemap_index ingests at least 10 pages with chunks visible in Qdrant (`/collections/nlweb_docs`).
- ✅ Collection auto-creates with the embeddingGemma dimension (no dim mismatch errors).
- ✅ Re-running the CLI without changes skips all pages due to state match.
- ✅ NLWeb `/ask` answers ground to newly ingested URLs.
- ✅ Tests pass locally (`pytest -q`) for the added unit tests.
- ✅ No changes to existing serving/LLM codepaths; ingestion is additive.

---

## Implementation Hints (for Copilot)
- Prefer small, composable functions: `iter_sitemap_urls`, `fetch_page`, `clean_text`, `chunk_words`, `embed_texts`, `ensure_collection`, `upsert_points`.
- Keep external deps minimal: `requests`, `bs4`, `trafilatura`, `readability-lxml`, `qdrant-client`.
- Handle exceptions per-page; never crash the whole run on one bad URL.
- Log succinctly: `INFO upserted N chunks from URL`, `INFO unchanged skip URL`, `WARN fetch failed URL (status/exception)`.



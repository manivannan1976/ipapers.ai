# Cache Coverage & TTL Alignment

_Source: review of `main` plus remote branches `feature/redis-caching-api`, `feature/db-query-caching`,
`feature/in-memory-caching-request-scooped-data-design` (as of 2026-11-24). Branches are unmerged, so values may change._

## 1. Coverage table

| Endpoint / function | Frontend cache | Backend cache | Cache type | TTL (frontend / backend) | Metrics |
|---|---|---|---|---|---|
| `searchPapers()` → `POST /api/search/` → `keyword_search` | Partial: `searchStore` holds the last query's results in memory (no TTL; cleared on a new search) | Yes (branch `redis-caching-api`: cache-aside in `db_service.keyword_search`) | Redis | none / **1800 s** (±10% jitter) | No |
| `getRecentPapers()` → `GET /api/papers` | **None on main** (`api.ts` calls the API every time) | No `get_recent_papers` / LRU code found on any branch | – | none / none | No |
| `countPapers()` → `GET /api/papers/count` | None | Yes (`redis-caching-api`, `api/paper.py`) | Redis | none / **300 s** | No |
| `get_paper_by_id` → `GET /api/papers/{id}` | Partial: `paperStore.getCurrentPaper()` (current paper only, no TTL) | Yes (`redis-caching-api`). No `lru_cache` found. | Redis | none / **3600 s** (invalidated on `click_paper`) | No |
| `get_papers_by_ids` | N/A | Yes (`redis-caching-api`) | Redis | N/A / **3600 s** | No |
| LLM responses (chat / paper / search / quick summary) | N/A | Yes (`llm_cache.py`) | Redis | N/A / **300 / 3600 / 7200 / 3600 s** (env-overridable) | **Yes**: `llm_cache_hit_total`, `llm_cache_miss_total`, latency (`get_metrics()`) |
| Pure helpers `tokenize_query`, `preprocess_search_query` | N/A | Yes (`app_cache.py`, branch `in-memory-caching-…`) | In-process `lru_cache`, max 500 | N/A / none (process lifetime) | **Yes**: hits/misses/ratio via `get_cache_stats()` |

Also: `db_cache.py` (branch `db-query-caching`) defines `TTL_KEYWORD_SEARCH_SECONDS=1800` and `TTL_PAPERS_BY_IDS_SECONDS=3600`
via env `DB_CACHE_TTL_*`. `redis-caching-api` hardcodes the same numbers inline instead.

## 2. TTL alignment answers

1. **Redis TTLs documented?** Now: `keyword_search` = 1800 s, `get_papers_by_ids` = 3600 s (jitter ±10%).
2. **Does the frontend 5-min TTL match the backend?** No. No frontend TTL exists yet (the 5-min value is only planned). Backend values range from 5 min (count) to 60 min (paper lookups), 30 min (search).
3. **4-hour update cycle?** (cron `0 */4 * * *`, `setup_cron.sh`; APScheduler on branch `python-scheduler-for-cron-jobs`.)
   All TTLs are shorter than 4 h, so staleness is bounded, which is safe. Nothing invalidates caches when the job finishes, so after an update a user can see up to 60 min of old data from backend plus 5 min from frontend.
4. **Where should TTLs live?** Recommended: one module, `backend/app/config/cache_ttl.py` (env-overridable), imported by every cache. This replaces the inline literals (`300`, `3600`, `1800`). Frontend TTL goes in one constant in `frontend/src/config/cache.ts`. This document is the human-readable index.

### Recommended target values

| Item | Frontend | Backend | Reason |
|---|---|---|---|
| Search results | 5 min | 30 min | Frontend ≤ backend ≤ 4 h |
| Recent papers | 5 min | 15 min | Changes on each ingest |
| Count | 5 min | 5 min | Cheap, visible number |
| Paper by id | n/a | 60 min | Rarely changes |
| After ingest job | n/a | Flush `search`/`papers` keys at end of `fetch_papers` | Removes the stale window |

## 3. Metrics

Exists: LLM cache hit/miss/latency; in-process LRU hit/miss ratios. Both are only readable in-process, and no endpoint exposes them.
Missing: hit/miss for Redis API/DB caches (`api_cache.py`, `db_cache.py`), endpoint latency, request frequency.

Proposed (not yet implemented):
- Add hit/miss counters to `RedisAPICache.get_json` and `DBCache.get`, keyed by namespace.
- Add a `GET /api/admin/cache-stats` endpoint that merges `get_metrics()` and `get_cache_stats()`.
- Add timing middleware logging `endpoint, ms, cache_hit` to measure latency before and after, and request frequency.

## 4. Open items for the team
- PRs #17–#20 are unmerged; re-verify this table after merge. PR #18 (LRU for `get_paper_by_id`, `get_recent_papers`, `get_search_suggestions`) is not on any remote branch I could see.
- Decide the frontend cache implementation (Pinia vs. the existing `reactive` stores).

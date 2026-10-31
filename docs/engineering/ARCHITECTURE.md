# Runtime architecture

This is a code-level map of MarginHash as inspected on 1 November 2026. The private codebase contains deployment configuration, but the state of the current production server and its installed commit were not verified for this document.

## Process boundary

The web entrypoint is wsgi.py, which imports the Flask application from catalog.py. A separate scheduler.py drives source collection, merge, price-history writes and read-model refresh. The private codebase's web and scheduler unit files set distinct process roles and disable startup background jobs in the web process. The parser-group unit is present as configuration; this does not establish which units are enabled on a server.

Manual web commands are written as deduplicated JSON files through runtime_boundary.py. The scheduler can consume those files without importing Flask route code. This keeps ordinary requests from directly running a scrape. The boundary is deliberately simple and file-based; it is not a distributed task queue.

The web-facing path normally reads prepared files:

~~~text
source adapters → per-source caches → normalization → ASIC store
                                           ↓
                                  read-model builder
                                           ↓
                         API, catalog and landing snapshots
                                           ↓
                              Flask routes → browser
~~~

The builder publishes the API snapshot with an atomic file replacement (read_model_builder.py, runtime_boundary.py). The catalog syncs a newer snapshot into bounded process-local caches (catalog.py). Some routes still have legacy merged-JSON fallbacks, and an import of catalog.py can start local background jobs unless the disabling environment flag is set. The process boundary is therefore an implemented design with compatibility paths, not a claim that every request is a pure snapshot lookup.

## Storage choices

- asics_store.py uses SQLite tables for algorithms, normalized models and seller offers. Frequently queried fields are columns; less frequently used specifications are stored as JSON. The merger writes the store, while some readers retain JSON fallback.
- prices_history_store.py uses SQLite WAL for price positions and change/hold points. The JSON history is retained for explicit legacy import. Local migration checks exist, but production migration is unverified.
- runtime_boundary.py places source caches, rates and read-model snapshots in separate runtime directories. Snapshots carry timestamps or build versions so web processes can detect changes.

SQLite and JSON are used for different purposes. The current system has neither a single all-SQLite data plane nor a network database.

## HTTP and frontend

catalog.py serves pages and API routes. The public catalog payload endpoint returns a weak ETag and cache headers, allowing a browser to receive 304 when the version has not changed (catalog.py). Routes such as raw data and search suggestions check internal access (catalog.py). The history API builds cached payloads by period and source revision (catalog.py).

HTML is assembled on the server, with page-specific code in landing.py, detail_page.py and public_assets/. The catalog inserts an initial subset of grouped data and preloads the full payload, then vanilla JavaScript handles filters, sorting and search (catalog.py). Catalog SSR cards are currently disabled because server and client recalculation had disagreed; the page uses a skeleton followed by the client render. The model page loads its history chart when the chart area enters view (detail-page-runtime.js).

## Performance and failure behavior

Prepared read models, bounded in-memory payload caches, ETag responses and lazy chart loading are concrete latency-oriented choices in the code. The Nginx configuration includes static-file handling and HTTP cache directives. No repeatable benchmark or verified production load figure is included here, so this document does not call the system high-load or fault-tolerant.

There are known limits. A parser timeout does not terminate its worker thread. Some adapters can return stale source cache without a distinct failure status. One source-cache writer is not atomic. A long merge and a single local task queue can become bottlenecks. These are reasons to measure and harden the next iteration, not properties to hide in a portfolio.

The private codebase has targeted unittest cases for calculations, offer identity, source fallback, snapshots, history and release staging. No CI workflow was found in that checkout at review time. The staged WinSCP/deploy path is implemented locally but has not been validated end to end against a confirmed production base commit.

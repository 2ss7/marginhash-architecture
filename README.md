# MarginHash

MarginHash turns inconsistent ASIC listings from sellers serving the Russian market into a comparable catalog and historical price data. I built the source adapters, ingestion and matching pipeline, profitability calculations, and the Flask application that presents the resulting data. The private codebase registers 20 source adapters; this is a count of configured integrations, not a claim that 20 sellers are currently available in production.

The engineering problem is preserving the identity of an individual offer while combining data from sites that disagree on model names, units, availability and specifications. The catalog is designed so a displayed model card takes its price, hashrate, power and calculated return from a compatible selected offer.

This public repository contains architecture notes only. The application source code, seller caches, price data and deployment configuration are private. The descriptions below refer to the local code inspected on 1 November 2026; they are not claims about the currently deployed version.

## System at a glance

~~~mermaid
flowchart LR
    Sites[Seller sites] --> Adapters[Source-specific adapters]
    Adapters --> Cache[Per-source caches]
    Cache --> Merge[Normalize and match offers]
    Merge --> ASIC[(ASIC SQLite store)]
    ASIC --> Builder[Read-model builder]
    Builder --> Snapshots[Versioned JSON snapshots]
    Snapshots --> Web[Flask pages and API]
    Web --> Browser[Vanilla JS catalog and charts]
    Merge --> History[(SQLite price history)]
    History --> Web
    Rates[Network rates and exchange rate] --> Builder
~~~

The scheduler runs parser groups and read-model jobs outside the web request path. The runtime boundary carries commands and prepared files between background work and Flask. The web process normally reads snapshots; the current code retains legacy fallbacks for some data paths.

## Data collection

The source registry assigns 20 adapters to three groups. Each adapter handles a seller's actual page structure and emits a common offer shape with fields such as model name, hashrate, unit, power, price, status, source and direct URL. The extraction methods vary by source:

- MiningMoon reads a WooCommerce JSON API.
- MineStream reads AJAX fragments with an HTML fallback.
- Geometria has a browser automation fallback when direct HTTP is insufficient.
- PushMiner combines catalog HTML with Product/Offer JSON-LD. It keeps the exact offer URL so a detail page cannot silently replace one variant with another.

The rate limiter applies intervals per source and request mode. The parser runtime tracks source failures and retry cooldowns. Cache fallback exists, but not every adapter reports fallback as a distinct scheduler failure, so the code does not guarantee that every cached offer is fresh. [Data collection details](docs/engineering/DATA_PIPELINE.md).

## Normalization and read model

The merger converts currencies and hashrate units, rejects unusable listings, fills missing specifications from compatible sources and keeps seller-specific offers. Fuzzy matching in the shared specification index requires an exact model token, for example S21 does not match S21e; cooling variants have a separate guard. This is rule-based entity resolution rather than an unrestricted fuzzy join.

The read-model builder removes exact duplicates and non-miner listings, marks inferred power, and quarantines extreme price outliers while retaining the listing. The grouping layer separates incompatible variants and chooses a representative priced offer for model cards. It requires independent source agreement before replacing suspicious group specifications. These rules matter because mixing one seller's price with another variant's hashrate would make the calculated return misleading. [Pipeline and matching rules](docs/engineering/DATA_PIPELINE.md).

## Calculations and history

The calculation engine derives daily net revenue from a rate per hashrate unit, pool and P2P fees, power in kW and an electricity tariff:

~~~text
net_RUB_per_day = rate_RUB_per_unit_day * hashrate * (1 - pool_fee) * (1 - P2P_fee)
                  - power_kW * 24 * electricity_RUB_per_kWh
~~~

The rate refresh combines network data from WhatToMine with a USD/RUB input. The longer scenario accumulates daily results until the fixed halving date in the current code, applies annual tariff changes and shows base, +20% and -20% revenue cases. Payback is the first day cumulative net reaches the price of the selected offer. These are scenario calculations, not income forecasts.

The local price-history store uses SQLite WAL. It records price changes and periodic unchanged-price observations, updates a history revision in the same transaction, and serves the web through a version-aware cache. The old JSON file is an explicit legacy import source, not the automatic read path. Production migration has **not** been verified. [Calculation and history details](docs/engineering/ANALYTICS.md).

## Web application and operational choices

The backend is Python, Flask and Gunicorn with sqlite3, requests and Beautiful Soup. Pages are rendered by Python plus vanilla JavaScript; the model page loads Chart.js when needed. The catalog uses a prepared JSON payload for client-side search, filtering and sorting. The catalog endpoint uses a versioned payload and HTTP cache headers; history endpoints cache by data version and selected period. Some API routes are internal or protected, so this is not advertised as an unrestricted public API.

Bounded in-memory caches and prepared snapshots reduce repeated work in requests. The private codebase also contains Nginx and systemd configuration and a staged deploy implementation. Their presence is not evidence of measured throughput or a completed rollout of the latest local code. No CI configuration was found in the private checkout at review time. [Architecture and trade-offs](docs/engineering/ARCHITECTURE.md).

## Private code map and verification

| Area | Main modules |
| --- | --- |
| Source collection | source/, parser_runtime.py, scheduler.py |
| Normalization and offer matching | data_merger.py, read_model_builder.py |
| Storage and published views | asics_store.py, prices_history_store.py, read_model_snapshots.py, runtime_boundary.py |
| Web and UI | catalog.py, detail_page.py, landing.py, public_assets/ |
| Regression checks | tests/ |

The tests cover model/offer consistency, profitability assumptions, grouping, snapshots, price history and source-specific parser failures. For this documentation review, four focused unittest modules ran under Python 3.13 with background startup disabled: 20 tests passed. This is not a full-suite or production validation. The private audit maps these claims to code; it is not included in this documentation repository.

This README describes the local code as inspected on 1 November 2026. It does not assert production freshness, total market coverage or deployment of unfinished local changes. Market-wide supplier share and seller deviation from a model's median price are planned ideas, not implemented metrics.

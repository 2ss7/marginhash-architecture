# Data collection and normalization

This document describes the local code inspected on 1 November 2026. It explains how offers move from seller pages to the catalog read model. It does not measure the current availability of external sources.

## Adapter contract

parser_runtime.py registers 20 adapters in three groups: fast/stable, slower sources, and detail/image-heavy sources. The registry is the source of truth for configured integrations. Each adapter is responsible for its source's format and returns seller-level offers through a common shape assembled by finalize_item:

~~~text
name, hashrate, hashrate_unit, price_rub, power_watts,
algorithm, direct_url, image_url, status, status_text,
_source, date, specs
~~~

Each row carries a seller URL. A variant-specific URL is validated where the source exposes a variant selector, as in PushMiner; the common adapter contract alone does not prove that every seller link identifies a unique variant. Unknown values can stay empty; downstream code should not invent a price or treat unknown profitability as zero.

The extraction strategy depends on the seller. MiningMoon reads WooCommerce Store API data. MineStream handles AJAX fragments and has an HTML fallback. Geometria can use Playwright after a direct-request failure. PushMiner parses catalog cards and then reads Product/Offer JSON-LD for a selected variant. This is a set of source-specific adapters, not one universal scraper.

## Scheduling and source failures

scheduler.py runs groups with a configurable maximum number of active groups. Sources inside a group run sequentially, with an inter-source delay; the group path uses a thread pool when more than one group is active. The value of parsers.worker_group_workers controls group concurrency and has a legacy fallback in parser_runtime.py. It is separate from the per-source detail concurrency cap.

The shared rate limiter spaces requests by source and request mode with jitter. Per-source state records outcomes, failure streak and the next allowed retry. A failed source can therefore back off without forcing every other adapter to stop. This is failure management in the scheduler, not process-level isolation: its timeout stops waiting for a daemon thread but cannot kill that thread (scheduler.py).

Adapters persist source-specific JSON caches. A scheduled merge reads those caches after the groups finish (scheduler.py). Some adapters explicitly signal that a result came from stale cache; the general fallback helper does not do so for every source. The common cache writer also writes directly to a file, unlike the later atomic snapshot writer. These are known limits of the current freshness and crash-safety model.

## From offers to comparable models

data_merger.py performs a separate normalization pass:

1. Build a shared specification index from source records in configured priority order. Missing fields may be filled from compatible records.
2. Clean names, convert currency and hashrate units, reject category headings and unusable prices.
3. Keep seller, URL, status and specifications on each normalized offer.
4. Remove cache/parser duplicates with the same source, normalized name and price. This is a bounded duplicate rule, not a claim that every semantically duplicate listing is found.

The shared specification index uses guarded fuzzy matching. At least one alphanumeric model identifier must match exactly, so S21 and S21e are different. Air, hydro and immersion words prevent cross-variant matches. A secondary specification source can fill gaps, but the source name is kept when a loose reference match would rename the product incorrectly (data_merger.py).

The read-model builder then removes non-miners and exact duplicate offers, keeps different hashrate/URL variants, and marks values filled from peers or estimated power. Price outliers above 3× or below one third of a model median are quarantined only when at least four comparable priced offers exist (read_model_builder.py). The listing remains available without its suspect purchase price.

The grouping layer separates clear specification variants. It accepts a replacement for a grossly inconsistent representative only when compatible priced offers are confirmed by at least two independent sellers. The resulting model card retains a representative offer identity, allowing its displayed price and economic metrics to refer to the same variant.

## A concrete source-change example

PushMiner's newer catalog has multiple offer variants under a model URL. Its parser follows only the next sequential page of the same catalog (source/pushminer.py), validates that a card's model and purchase links contain the same offer selector, and preserves that direct link (source/pushminer.py). Detail enrichment accepts exactly one Product/Offer pair whose two JSON-LD URLs match the chosen offer (source/pushminer.py). An out-of-stock offer has no purchase price even when JSON-LD includes a numeric price. Generic seller images are excluded instead of being presented as photos of a specific ASIC variant. The focused regression tests are in test_pushminer_new_catalog.py and test_pushminer_new_detail.py.

## Output and boundaries

The merge writes ASIC rows to asics_store.py; the current read path also retains a legacy JSON fallback (read_model_builder.py). read_model_builder.py calculates prepared values and publishes snapshots. Flask normally reads those snapshots rather than running the parsers during a public request.

Configured adapters, successful local parser tests and source caches do not prove that all sellers are live or that production data is fresh. Source-specific fallback and the date attached to an offer need to be evaluated separately before making freshness claims.

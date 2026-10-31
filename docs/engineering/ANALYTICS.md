# Calculation and historical data

MarginHash computes scenario metrics from seller offers and external network inputs. The numbers are estimates under explicit assumptions. This document describes the local implementation, not a verified production data feed.

## From network inputs to daily net revenue

refresh_rates reads WhatToMine network data and a USD/RUB rate. For each supported coin, it derives an expected RUB/day value for one hashrate unit from network hashrate, block interval, block reward and exchange rate. Missing or failed inputs can use previously stored or default rates. A fallback rate is an assumption, not a fresh observation.

simulate_single then uses the ASIC's algorithm, hashrate, power and the calculation settings:

~~~text
gross = rate_RUB_per_unit_day × hashrate
net_day = gross × (1 - pool_fee_pct / 100) × (1 - p2p_fee_pct / 100)
          - (power_watts / 1000) × 24 × electricity_RUB_per_kWh
~~~

The read-model rebuild calculates a fast daily figure for each cleaned offer (read_model_builder.py). The full path accumulates day-by-day base, optimistic and pessimistic scenarios until the fixed halving date in the current code. Optimistic and pessimistic revenue multipliers are 1.20 and 0.80. Electricity can rise annually on 1 July. Scenario ROI is cumulative net divided by the offer price; payback is the first day cumulative net meets that price. These calculations do not model every future change in network difficulty, coin price, fees or availability.

The published snapshot stores the calculation settings alongside the derived figures. Web code rebases offer economics when a user changes a tariff or fee so the same saved basis is used across catalog, detail and comparison. If a raw fallback has no known profitability basis, the result stays unknown instead of being rendered as a real zero (catalog.py, catalog.py).

## Offer-level and market-level aggregates

The grouped read model calculates separate model-level aggregates from comparable offers: total offer count, priced seller count, minimum and arithmetic mean of eligible priced offers, best net and best ROI. Those maxima need not come from the minimum-price offer. The rendered catalog card chooses an offer and uses that offer for its displayed price, net and ROI (catalog.py). Price outliers can remain in the offer list while being excluded from the purchase-price calculation.

The supplier analytics works over observed offers. It calculates each priced offer's percentage deviation from its model's **arithmetic mean**, then averages those deviations per supplier. It also calculates hidden-price share and summary ROI/payback statistics under the current settings. These are statistics of the sources MarginHash observes, not estimates of every seller in the market. In particular, the implemented deviation metric is not a deviation from a model's median, and the code does not calculate market share.

## Price history

The scheduler records priced merged offers through save_price_history. A stable history ID associates a position with source, model, algorithm, hashrate, power and link. prices_history_store.py stores positions and points in SQLite WAL.

The point writer uses two states (prices_history_store.py):

- A change point records the first observation or a changed price.
- A hold point moves forward while the price is unchanged. The previous plateau boundary is retained when a later change arrives.

Batch writes, position metadata and the history revision commit in one transaction (prices_history_store.py). The web checks that revision at most once per second per process before reusing its history cache (prices_history_store.py). The old JSON file can be imported explicitly; ordinary reads and writes do not rebuild SQLite from it.

The history endpoint builds period-specific payloads and caches them against the source version. Historical profitability combines prices with historical rates when available, but avoids carrying a rate through a long missing interval. Long series are reduced before drawing charts. A price chart and an income scenario answer different questions and should not be read as a realized return.

## Current limits

The SQLite price-history read/write path and migration checks have been verified locally, but the production migration and freshness of production history remain unverified. ASIC storage still has a JSON fallback. Broader ideas, including supplier market share and median-based supplier pricing, are not implemented.

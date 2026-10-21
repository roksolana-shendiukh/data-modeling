# Late-Arriving Data

Production data does not arrive in order. Devices go offline, upstream APIs fail, and corrections are issued after a partition has closed. A model that assumes events arrive on time and dimensions exist before facts produces wrong results. Late data is the normal case and is designed for from the start.

## Event time and processing time

* Event time is when something happened. Processing time is when the pipeline sees it. The gap between the two is lateness.
* Every fact stores both: a business date used for reporting and a load timestamp used for operations.
* Analytics partition and aggregate by event time. Processing time is used for pipeline operations only.

## Late-arriving facts

* The event belongs to a historical partition that may already be closed.
* Three strategies exist:
  * Insert into the historical partition. Most correct, but downstream jobs must be idempotent.
  * Land rows in a separate late-data partition and move them with a periodic reconciliation job. Less disruptive, more moving parts.
  * Partition by processing date and keep the event date as a column. Simple, but consumers must filter on event date.
* Inserting the row is only the first step. Every aggregate and cache built on that partition is stale until it is recomputed. Only the affected partitions are recomputed, never the whole table.
* A late-data SLA defines how late is acceptable. Events inside the window are handled automatically. Older events go to a reconciliation queue. The cutoff is a business decision.
* In streaming pipelines a watermark marks the point before which all events are believed to have arrived. Events behind it go to a side output (dead letter queue) for batch reconciliation.

## Late-arriving dimensions

* A fact can arrive before its dimension row, so the foreign key lookup fails. The fact is neither skipped nor queued.
* An inferred member is a placeholder dimension row with default values, an `is_inferred` flag and an `inferred_at` timestamp. The fact loads at once with a valid surrogate key.
* When the real dimension arrives, the placeholder is backfilled. The surrogate key does not change, so no fact row is rewritten.
* Under SCD Type 2 the inferred member is the first version. At backfill it is expired and a new row with real attributes is inserted. Facts keep the key that was valid in their period.
* The natural key of the dimension is what lets the real record find its placeholder.

## Corrections

* A correction is never a destructive `UPDATE` or a delete-and-reinsert. Both destroy the audit trail and leave downstream aggregates ambiguous.
* In an immutable fact table corrections are additive.
* Reversal pattern: a row that negates the original and a row that carries the new value. `SUM` returns the correct result without special handling. Best for financial and regulated data.
* Version column: each correction inserts a row with an incremented version, and queries keep only the latest version per entity. Best for high correction volume, but every consumer must apply the filter.
* A correction row references the row it changes through the identifier of the original.

## Reprocessability

* Loads are idempotent: running the same partition twice gives the same result. Delete-insert inside one transaction, or a merge on a dedup key.
* Reprocessing targets one partition and does not depend on other partitions being processed first.
* Audit timestamps (`loaded_at`, optionally `reprocessed_at`) record when the pipeline touched a row, separate from business time.
* Aggregates are derived and can always be rebuilt from facts. A control table records what arrived and when, and each aggregate records when it was last rebuilt. A period is stale when something arrived after its last rebuild.

## Where the pieces live

* Staging holds raw landed records, which makes a rebuild from scratch possible.
* The core star holds facts and dimensions.
* Aggregate tables sit on top of the star as the layer the dashboard reads.
* Control tables (sync logs, audit tables) are pipeline metadata and stay outside the star.

## Monitoring

* Late-fact volume: share of events arriving more than one hour after event time.
* Inferred-member age: count of inferred members older than 24 hours. A growing count signals a degrading upstream dimension source.
* Reconciliation reports compare base fact totals with source-system totals per date. Discrepancies reveal missed late data.

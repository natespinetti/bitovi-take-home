## Idempotency

The application should be the primary owner of the idempotency guarantee because it
would have the deepest understanding of how to process duplicates. The messaging platform
should be treated as providing at-least-once delivery. In production, we could add a
processed-events table that tracks a unique ID and handles the order. If a duplicate appears,
check against the table, and skip if a match is found.

## Scaling & backpressure

I would scale the workers horizontally based on Kafka consumer lag and
the age of the oldest unprocessed event. The most likely bottleneck would be Postgres.
If traffic spikes overwhelm workers, there's a possibility that they are opening hundreds of
db connections and bogging down Postgres. Once a maximum amount of connections is found,
we could split that pool between the workers to handle their minimum-maximum events.
Workers can also reduce or pause consumption of events while the db catches up.

## Breaking rollout

I'd create a v2 of the current schema and deploy worker code to handle processing
v1 and v2 schema. Once you verify workers are capable of processing both, adjust the
system to publish to v2. Once v1 events drain and process, remove the connection to v1.
To prevent double-processing, each schema would use the same stable event ID and idempotency
practices would prevent dupes.

## What I'd improve

With more time and if we were making this for production:

- Add real Kafka connection for true event processing
- Add db connection to simulate idempotency dupe managment
- Expand Prometheus metrics into a curated dashboard
- True secret rotations

## Questions

- **Idempotency.** This worker consumes order events and will sometimes receive
  the same event more than once. How do you make processing idempotent, and
  where does that responsibility live — the app, the platform, or both?
- **Scaling & backpressure.** Traffic is spiky. How would you scale this safely?
  What breaks first, and how do you keep the worker from overwhelming its
  downstream Postgres when it does?
- **Breaking rollout.** You need to ship a change to the order event schema
  that is **not** backward-compatible, with zero downtime and no dropped or
  double-processed events. Walk us through the rollout.

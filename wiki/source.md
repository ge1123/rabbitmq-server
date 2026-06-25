# Wiki Source and Coverage

## Provenance

 * Repository: `rabbitmq-server`
 * Verified commit at skeleton creation: `e3005f0ba2b699b31b667b0214a2cddfb7ba07dc`
 * Primary code relationship source: `codebase-memory-mcp`
 * Final behavioral authority: repository source code

## Coverage

The wiki currently covers graph-native orientation for these durable areas:

 * Broker core ownership, boot, boot-step execution, and dynamic supervision
 * AMQP 0-9-1 connection/channel lifecycle and method dispatch
 * Queue/exchange declaration, binding, routing, and queue-type delivery
 * Authentication/authorization coordinator, backend contracts, and protocol
   adapter handoff
 * Management HTTP API/UI routing to broker operations
 * Management plugin ownership for listener setup, dynamic route extension,
   HTTP auth layers, stats/cache boundaries, management-agent collection, and
   UI request plumbing
 * MQTT, STOMP, stream, and WebSocket protocol adapter boundaries
 * Federation links, shovel worker handoff, and stream queue/storage
   interaction boundaries
 * Build, release, test, packaging, Selenium, CI, and release-note routing
 * Shared `rabbit_common` records/types, generated AMQP framing, helper
   boundaries, SASL/password helper routing, and common Make glue
 * CLI command discovery, validation, output formatting, printer selection, and
   node RPC handoff routing

It intentionally does not cover exhaustive component maps, caller/callee lists,
full route inventories, full AMQP method inventories, CI matrices, release-note
summaries, or generated source summaries. Those should be regenerated from the
graph or searched in source when needed.

The wiki preserves business logic, investigation conclusions, routing decisions,
source-vs-graph pitfalls, and reusable query strategy. Use source verification
where behavior depends on Erlang multi-clause dispatch, dynamic callbacks,
module attributes, RPC calls, Make/YAML/shell behavior, generated/static
assets, or other evidence the graph cannot prove.

## Graph Artifact

If a graph artifact is present, prefer the indexed project whose root path
matches this checkout. If `index_status` is missing or stale, run or request
`index_repository` before relying on relationship results.

## Tool Notes

During the 2026-06-25 batch ingest, `list_projects` showed the matching
`Users-wuchuni-Desktop-system_design-rabbitmq-server` project and the
coordinator initially confirmed `index_status` as `ready`. Later worker and
coordinator calls saw intermittent `index_status` transport/project lookup
failures while `search_graph` and `trace_path` still worked. Treat this as a
tool-state warning: reconfirm the graph project at the start of each
investigation, and record the failure if graph discovery proceeds after an
`index_status` inconsistency.

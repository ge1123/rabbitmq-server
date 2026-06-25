# Wiki Log

## 2026-06-25

 * Added `ingest-queue.md` with repository-wide deep ingest work items,
   worker write scopes, graph query plans, source verification strategy,
   durable-knowledge goals, queue status, and first-batch dispatch plan
 * Added `components/common-libraries.md` covering `deps/rabbit_common`
   routing for shared records/types, generated AMQP framing, shared helpers,
   SASL/password helpers, data coercion, common Make glue, and graph pitfalls
   around records, macros, generated code, behaviours, and dynamic calls
 * Added `components/cli.md` covering `deps/rabbitmq_cli` command discovery,
   parser and command-module routing, validation phases, distribution setup,
   output formatter/printer selection, streaming RPC helpers, and node RPC
   pitfalls
 * Added `components/management.md` covering management listener setup,
   dynamic route extension, Webmachine resources, HTTP auth layers, broker
   handoff paths, stats/cache boundaries, management-agent collection, and UI
   request plumbing
 * Deepened `queries/build-test-release.md` with build/test/release/package
   routing across Make, dependency pins, source-run nodes, Common Test,
   packaging, Selenium, CI, compatibility, and release dispatch
 * Updated component, query, and source indexes with new page links and
   coverage notes
 * Recorded an intermittent 2026-06-25 `index_status` MCP transport/project
   lookup failure in `source.md` while preserving source-verified conclusions

## 2026-06-24

 * Created graph-native wiki skeleton
 * Added component routing, query recipes, flow and QA entry points, source
   provenance, and templates
 * Documented that the wiki should preserve reusable routing and query strategy
   instead of copying graph-generated relationship inventories
 * Hardened wiki guidance around wiki-vs-graph responsibilities, source
   verification, and judgment-based text search fallback
 * Added Batch 1 broker-core overview covering `deps/rabbit` ownership,
   node-level startup entry points, dynamic `rabbit_sup` supervision,
   boot-step execution, graph trace starts, and boot-step graph pitfalls
 * Added broker-core query note with effective graph searches, trace starts,
   source verification points, and text-search fallback for
   `-rabbit_boot_step` declarations
 * Added Batch 2 connection/channel lifecycle flow covering AMQP 0-9-1
   connection supervisor setup, reader state, channel supervisor topology,
   channel creation, and cleanup behavior
 * Added Batch 2 AMQP method-handling flow covering channel-0 dispatch,
   command assembler handoff, channel cast handling, interceptors, and
   multi-clause `rabbit_channel:handle_method` dispatch
 * Added reusable connection/channel query note with graph trace starts,
   source verification points, and graph/source pitfalls for supervisor child
   specs and Erlang multi-clause method handlers
 * Added Batch 3 queue/exchange/routing/delivery flow covering AMQP 0-9-1
   queue declaration, exchange declaration, binding checks, exchange routing,
   queue-type delivery dispatch, and classic queue delivery handoff
 * Added reusable queue/exchange/routing query note with graph trace starts,
   source verification points, text-search fallbacks, and a graph/source
   mismatch note for `deliver_to_queues/3` to `rabbit_queue_type:deliver/4`
 * Added Batch 4 authentication/authorization flow covering
   `rabbit_access_control`, authN/authZ backend callback contracts, backend
   chain semantics, SASL mechanism handoff, protocol adapter routing, and
   credential-refresh reauthorization
 * Added reusable authentication/authorization query note with graph trace
   starts, source verification points, text-search fallbacks for dynamic Erlang
   callback names, and a noisy `query_graph` pattern to avoid
 * Improved component routing for auth work by pointing first to
   `rabbit_access_control` and then to internal, callback-contract, external
   backend, and SSL mechanism modules
 * Added Batch 5 management HTTP-to-broker flow covering listener registration,
   `/api` route construction, Webmachine resource callbacks, UI request helper
   paths, direct AMQP-method broker mutations, and publish/get channel paths
 * Improved component routing for management work with stable trace starts,
   HTTP callback naming conventions, and graph pitfalls for dynamic extension
   routes and RPC broker handoff
 * Added reusable management HTTP query note with handler callback searches,
   UI literal text-search fallback, and route-extension source verification
   points
 * Added `flows/protocol-adapters.md` covering MQTT, STOMP, stream, and
   WebSocket protocol adapter ownership, core auth/resource handoff points, and
   publish/subscribe/stream boundaries
 * Added `queries/protocol-adapters.md` with reusable graph search, trace, and
   source verification strategy for protocol adapter investigations
 * Updated flow, query, and component indexes with protocol adapter routing and
   links back to core auth and AMQP routing flows
 * Added Batch 7 federation/shovel/streams interaction flow covering exchange
   and queue federation link boundaries, shovel worker-to-protocol callback
   handoff, stream manager queue-type declaration handoff, and
   `rabbit_stream_queue` interaction with stream coordinator and Osiris
 * Added reusable federation/shovel/streams query note with graph searches,
   trace starts, source verification points, and pitfalls for dynamic callback,
   boot-step, AMQP, and Osiris call edges
 * Improved component routing for federation, shovel, and stream queue/storage
   investigations without duplicating protocol-adapter details
 * Added Batch 8 build/test/release/package query note covering Make entry
   points, plugin/component version routing, Common Test and source-run
   verification points, packaging delegation, Selenium orchestration, CI
   workflows, release-note search strategy, and text-search pitfalls
 * Improved component routing with concise build, test, release, packaging,
   Selenium, CI, script, and release-history entry points

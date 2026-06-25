# Wiki Ingest Queue

## Purpose

This queue coordinates deep wiki ingest work for `rabbitmq-server`. It keeps
work items explicit enough for parallel workers while preserving the wiki as a
durable routing and investigation layer, not a source mirror.

## Coordinator Rules

 * Use project `Users-wuchuni-Desktop-system_design-rabbitmq-server`
 * Confirm `index_status` is `ready` before graph work
 * Keep at most 3 to 5 workers active at once
 * Assign disjoint wiki targets when workers write directly
 * Ask workers to produce draft notes when multiple items would edit the same
   wiki page
 * Merge worker output through a consistency pass before updating index pages
 * Record source-vs-graph conflicts in the relevant page and `wiki/source.md`
   when reusable

## Phase 0 Snapshot

 * Repository root: `/Users/wuchuni/Desktop/system_design/rabbitmq-server`
 * Matching graph project: `Users-wuchuni-Desktop-system_design-rabbitmq-server`
 * Index status: `ready`
 * Graph size: 48421 nodes, 180876 edges
 * Existing wiki entry points: `index.md`, `components/index.md`,
   `queries/index.md`, `flows/index.md`, `qa/index.md`, `source.md`,
   `log.md`
 * Existing flow coverage: auth, connection/channel, AMQP method handling,
   queue/exchange/routing, management HTTP, protocol adapters,
   federation/shovel/streams
 * Existing query coverage: auth, broker core, build/test/release,
   connection/channel, management HTTP, protocol adapters,
   queue/exchange/routing, federation/shovel/streams

## High Priority Batch

Batch 1 should deepen low-conflict pages and produce reusable routing notes
without competing over the same wiki file:

 * `A-build-routing`: write `wiki/queries/build-test-release.md` and draft
   `wiki/components/index.md` merge notes only if needed
 * `C-common-libraries`: write `wiki/components/common-libraries.md`
 * `D-cli`: write `wiki/components/cli.md`
 * `E-management`: write `wiki/components/management.md` and update
   `wiki/queries/management-http.md` only after coordinator review

## Queue

### A-build-routing

 * Scope: repository routing, build system, dependency pins, compatibility,
   test entry points, packaging and CI ownership
 * Wiki target file: `wiki/queries/build-test-release.md`
 * Graph queries to run:
   * `get_architecture(aspects=["entry_points","clusters"])`
   * `search_graph(query="make common test release package", limit=20)`
   * `search_code(pattern="ENABLED_PLUGINS|RABBITMQ_ENABLED_PLUGINS", path_filter="^(Makefile|deps/|packaging/)", regex=true)`
   * `search_code(pattern="parallel-ct|ct-", path_filter="^(Makefile|deps/|.github/)", regex=true)`
 * Source verification strategy: read `Makefile`, `erlang.mk`,
   `rabbitmq-components.mk`, `plugins.mk`, `CONTRIBUTING.md`,
   `docs/compatibility.json`, selected `deps/rabbit_common/mk/*.mk`,
   selected `packaging/**/Makefile`, and selected workflow YAML
 * Expected durable knowledge: how to route build/test/release questions,
   where source-run targets differ from package targets, how dependency and
   compatibility evidence is verified, and where graph is less useful than
   text search
 * Status: done

### B-core-lifecycle

 * Scope: broker core application lifecycle, boot phases, boot steps,
   supervision and listener readiness
 * Wiki target file: `wiki/components/broker-core.md`
 * Graph queries to run:
   * `search_graph(name_pattern="^(boot|start|start_it|start_apps)$", qn_pattern=".*deps.rabbit.src.rabbit.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_boot_steps.*", limit=50)`
   * `trace_path(function_name=<rabbit:start/2 qualified name>, direction="outbound", depth=2)`
   * `query_graph` for functions in `deps/rabbit/src/rabbit_sup.erl` with
     high fan-in
 * Source verification strategy: verify `rabbit.erl`, `rabbit_sup.erl`,
   `rabbit_boot_steps.erl`, selected boot-step attributes, and listener boot in
   `rabbit_networking.erl`
 * Expected durable knowledge: runtime startup phases, dynamic supervision
   attachment, boot-step graph limits, and where plugin startup crosses into
   core
 * Status: pending

### B-core-connection-channel

 * Scope: AMQP 0-9-1 reader/channel ownership, channel supervision, method
   dispatch, cleanup and process relationships
 * Wiki target file: `wiki/flows/connection-channel-lifecycle.md`
 * Graph queries to run:
   * `search_graph(name_pattern=".*rabbit_reader.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_channel.*", limit=50)`
   * `search_graph(name_pattern="handle_method", qn_pattern=".*rabbit_channel.*", limit=50)`
   * `trace_path(function_name=<rabbit_reader start/connection function>, direction="outbound", depth=2)`
 * Source verification strategy: verify `rabbit_reader.erl`,
   `rabbit_channel.erl`, channel supervisor modules, and direct connection
   helper behavior where management or protocol adapters reuse AMQP channels
 * Expected durable knowledge: stable trace starts for connection questions,
   how method dispatch reaches channel code, and multi-clause verification
   pitfalls
 * Status: pending

### B-core-routing-queues

 * Scope: queues, exchanges, bindings, publish routing, delivery handoff,
   classic/quorum/stream queue boundaries
 * Wiki target file: `wiki/flows/queue-exchange-routing-delivery.md`
 * Graph queries to run:
   * `search_graph(query="exchange route binding queue declare publish", limit=50)`
   * `search_graph(name_pattern=".*rabbit_exchange.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_binding.*", limit=50)`
   * `trace_path(function_name=<rabbit_exchange route qualified name>, direction="both", depth=2)`
 * Source verification strategy: verify selected exchange, binding, queue
   type, routing and delivery modules in `deps/rabbit/src`
 * Expected durable knowledge: routing ownership, declaration paths, queue-type
   boundary decisions, and when graph relationships need source clause checks
 * Status: pending

### B-core-resource-flags

 * Scope: alarms, memory/disk/resource limits, feature flags and compatibility
   gates
 * Wiki target file: `wiki/components/broker-core-resource-flags.md`
 * Graph queries to run:
   * `search_graph(query="memory alarm disk alarm resource limit", limit=50)`
   * `search_graph(name_pattern=".*rabbit_alarm.*", limit=50)`
   * `search_graph(name_pattern=".*feature.*flag.*", limit=50)`
   * `trace_path(function_name=<feature flag controller qualified name>, direction="both", depth=2)`
 * Source verification strategy: verify `rabbit_alarm*.erl`,
   `rabbit_disk_monitor.erl`, memory monitor modules, `rabbit_ff_controller`
   and selected compatibility checks
 * Expected durable knowledge: where alarms originate, how feature flag state
   is coordinated, and which checks are runtime behavior versus config/schema
 * Status: pending

### C-common-libraries

 * Scope: `deps/rabbit_common`, shared records/types/helpers, framing,
   config/schema helpers, shared auth utility boundaries
 * Wiki target file: `wiki/components/common-libraries.md`
 * Graph queries to run:
   * `search_graph(query="framing table record config schema auth helper", file_pattern="deps/rabbit_common/.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_framing.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_data_coercion.*|.*rabbit_parameter_validation.*", limit=50)`
   * `query_graph` to summarize high fan-in functions under
     `deps/rabbit_common`
 * Source verification strategy: verify selected `deps/rabbit_common/src`
   modules, include files, and `deps/rabbit_common/mk` only where build
   behavior is in scope
 * Expected durable knowledge: reusable route from protocol/core questions into
   shared libraries, shared helper ownership, and common graph pitfalls around
   records/includes/generated framing
 * Status: done

### D-cli

 * Scope: `deps/rabbitmq_cli`, command routing, command modules, output
   formatting, validators and node RPC patterns
 * Wiki target file: `wiki/components/cli.md`
 * Graph queries to run:
   * `search_graph(query="command router formatter validator rpc node", file_pattern="deps/rabbitmq_cli/.*", limit=50)`
   * `search_graph(name_pattern=".*Command.*|.*command.*", file_pattern="deps/rabbitmq_cli/.*", limit=50)`
   * `trace_path(function_name=<CLI command dispatch qualified name>, direction="outbound", depth=2)`
   * `query_graph` for `deps/rabbitmq_cli` functions with high fan-out
 * Source verification strategy: verify command behavior modules, command
   discovery/dispatch code, formatter modules, validator modules and RPC helper
   modules
 * Expected durable knowledge: how CLI commands are located and executed, how
   output formatting is selected, and how node RPC calls cross into broker
 * Status: done

### E-management

 * Scope: `deps/rabbitmq_management`, HTTP API routing, resource auth,
   permission checks, stats collection, UI asset ownership
 * Wiki target file: `wiki/components/management.md`
 * Graph queries to run:
   * `search_graph(name_pattern=".*rabbit_mgmt_dispatcher.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_mgmt_wm_.*", limit=100)`
   * `search_graph(query="stats database management cache collector", file_pattern="deps/rabbitmq_management/.*", limit=50)`
   * `trace_path(function_name=<rabbit_mgmt_util direct_request qualified name>, direction="outbound", depth=2)`
 * Source verification strategy: verify dispatcher/listener modules,
   selected `rabbit_mgmt_wm_*` resource callbacks, auth helper modules, stats
   database/cache modules, and UI request helper files under `priv/www/js`
 * Expected durable knowledge: HTTP route construction, mutation handoff to
   broker, stats ownership, UI/API boundary, and dynamic extension route
   pitfalls
 * Status: done

### F-protocol-plugins

 * Scope: stream, MQTT, STOMP, AMQP 1.0 if present, WebSocket adapters and
   handoff into core auth/routing
 * Wiki target file: `wiki/components/protocol-plugins.md`
 * Graph queries to run:
   * `search_graph(query="mqtt stomp stream protocol connection publish subscribe", limit=80)`
   * `search_graph(name_pattern=".*rabbit_mqtt.*|.*rabbit_stomp.*|.*rabbit_stream.*", limit=100)`
   * `search_graph(query="web mqtt web stomp cowboy websocket", limit=50)`
   * `trace_path(function_name=<adapter connection entry qualified name>, direction="outbound", depth=2)`
 * Source verification strategy: verify adapter app/supervisor/reader modules,
   auth handoff, publish/subscribe handoff, stream queue manager boundaries and
   WebSocket dispatch modules
 * Expected durable knowledge: protocol plugin ownership, boundary with core
   auth/routing, WebSocket transport separation, and AMQP 1.0 routing if
   present
 * Status: pending

### G-authn-authz

 * Scope: core access control, internal and external auth backends, SSL
   mechanism, permissions, topic/resource checks, user/vhost path
 * Wiki target file: `wiki/components/authn-authz.md`
 * Graph queries to run:
   * `search_graph(name_pattern=".*rabbit_access_control.*", limit=50)`
   * `search_graph(name_pattern=".*rabbit_authn_backend.*|.*rabbit_authz_backend.*", limit=50)`
   * `search_graph(query="check user login permission vhost resource topic", limit=80)`
   * `trace_path(function_name=<check_resource_access qualified name>, direction="both", depth=2)`
 * Source verification strategy: verify `rabbit_access_control.erl`, internal
   backend modules, backend behaviour modules, external backend apps and SSL
   auth mechanism modules
 * Expected durable knowledge: authN/authZ coordinator responsibilities,
   backend contracts, adapter handoff points and permission-check pitfalls
 * Status: pending

### H-federation-shovel

 * Scope: federation exchange/queue links, upstream runtime behavior, shovel
   worker lifecycle, protocol-module handoff and supervision
 * Wiki target file: `wiki/components/federation-shovel.md`
 * Graph queries to run:
   * `search_graph(name_pattern=".*federation.*link.*|.*shovel.*worker.*", limit=100)`
   * `search_graph(query="upstream link status shovel protocol reconnect", limit=80)`
   * `trace_path(function_name=<rabbit_shovel_worker init qualified name>, direction="outbound", depth=2)`
   * `trace_path(function_name=<rabbit_federation_exchange_link init qualified name>, direction="outbound", depth=2)`
 * Source verification strategy: verify federation link modules, common helpers,
   shovel worker modules, protocol callback modules and status tables
 * Expected durable knowledge: lifecycle ownership, handoff contracts, status
   and runtime parameter behavior, and reconnect/policy pitfalls
 * Status: pending

### I-persistence-clustering

 * Scope: metadata store, mnesia/khepri usage, ra/osiris integration, quorum
   queue and stream persistence boundaries, cluster membership and recovery
 * Wiki target file: `wiki/components/persistence-clustering.md`
 * Graph queries to run:
   * `search_graph(query="mnesia khepri metadata store cluster recovery", limit=80)`
   * `search_graph(query="ra osiris quorum stream replica recovery", limit=80)`
   * `search_graph(name_pattern=".*rabbit_db.*|.*khepri.*|.*quorum.*", limit=100)`
   * `trace_path(function_name=<metadata store selected function>, direction="both", depth=2)`
 * Source verification strategy: verify selected metadata store modules,
   quorum queue modules, stream queue modules, cluster membership modules and
   recovery paths where graph evidence is not enough
 * Expected durable knowledge: metadata ownership boundaries, persistence
   engine boundaries, recovery entry points, and version/feature flag caveats
 * Status: pending

### J-tests-qa

 * Scope: Common Test layout, targeted suite execution, test helpers, logs,
   selenium/browser tests and QA routing
 * Wiki target file: `wiki/qa/testing.md`
 * Graph queries to run:
   * `search_graph(query="common test suite helper config broker test", limit=80)`
   * `search_code(pattern="init_per_suite|end_per_suite|groups\\(", path_filter="^deps/", regex=true, limit=50)`
   * `search_code(pattern="suite_template|run-suites|selenium", path_filter="^(selenium|.github/)", regex=true, limit=50)`
   * `query_graph` for TESTS edges grouped by source file where useful
 * Source verification strategy: verify `CONTRIBUTING.md`, selected
   component test directories, `deps/rabbit_ct_helpers`, `selenium/README.md`,
   selenium runner scripts and relevant workflow YAML
 * Expected durable knowledge: how to choose targeted tests, where helper
   behavior lives, how to inspect logs/artifacts and how browser tests differ
   from CT
 * Status: pending

### K-packaging-runtime-ci

 * Scope: runtime wrappers, packaging, release notes, generated scripts, CI
   workflows and release automation
 * Wiki target file: `wiki/components/packaging-runtime-ci.md`
 * Graph queries to run:
   * `search_code(pattern="rabbitmq-server|rabbitmqctl|rabbitmq-plugins", path_filter="^(scripts|deps/rabbit/scripts|packaging)/", regex=true, limit=50)`
   * `search_code(pattern="workflow_dispatch|release|package|docker", path_filter="^.github/workflows/", regex=true, limit=50)`
   * `search_graph(query="runtime script wrapper package release", limit=20)`
   * `get_architecture(aspects=["entry_points"])`
 * Source verification strategy: verify top-level scripts, `deps/rabbit/scripts`,
   packaging makefiles, selected workflow YAML and release note conventions
 * Expected durable knowledge: installed wrapper ownership, package build
   routing, CI entry points, release-note search strategy and generated artifact
   pitfalls
 * Status: pending

## First Worker Dispatch Plan

### Worker 1: A-build-routing

 * Write scope: `wiki/queries/build-test-release.md`
 * Draft-only scope: any `wiki/components/index.md` navigation changes
 * Reason: broad but mostly non-code evidence, unlikely to conflict with
   component-specific pages

### Worker 2: C-common-libraries

 * Write scope: `wiki/components/common-libraries.md`
 * Draft-only scope: `wiki/queries/index.md` link text
 * Reason: fills a missing shared-library routing page that many later items
   can link to

### Worker 3: D-cli

 * Write scope: `wiki/components/cli.md`
 * Draft-only scope: `wiki/queries/index.md` link text
 * Reason: CLI is a separate component and can be ingested without touching
   broker-core flow pages

### Worker 4: E-management

 * Write scope: `wiki/components/management.md`
 * Draft-only scope: `wiki/queries/management-http.md` and
   `wiki/flows/management-http-to-broker.md`
 * Reason: management already has initial coverage, so worker should deepen
   ownership and avoid overwriting existing flow/query conclusions

## Batch Consistency Checklist

 * Check all new links from `wiki/index.md`, `wiki/components/index.md`,
   `wiki/queries/index.md` and `wiki/flows/index.md`
 * Remove duplicated explanation that belongs in exactly one page
 * Preserve graph query recipes without pasting large result sets
 * Ensure behavioral claims cite source verification, not graph evidence alone
 * Confirm Markdown list items do not end with final full stops
 * Update `wiki/source.md` coverage and `wiki/log.md` only after reviewed
   wiki changes land

## Batch 1 Results

 * Completed `A-build-routing` in `wiki/queries/build-test-release.md`
 * Completed `C-common-libraries` in `wiki/components/common-libraries.md`
 * Completed `D-cli` in `wiki/components/cli.md`
 * Completed `E-management` in `wiki/components/management.md`
 * Updated `wiki/components/index.md`, `wiki/queries/index.md`,
   `wiki/source.md`, and `wiki/log.md` during coordinator consistency pass

## Batch 1 Open Items

 * Re-run CLI `trace_path` and high fan-out `query_graph` patterns when the MCP
   transport is stable; Worker 3 completed source-verified routing but those
   graph calls were interrupted
 * Reconfirm `index_status` at the start of the next batch; Worker 4 and the
   coordinator saw intermittent transport/project lookup failures after the
   initial ready status
 * Consider folding management auth-helper and stats/cache searches into
   `wiki/queries/management-http.md` in a later management-specific pass
 * Consider adding a short authorization prelude to
   `wiki/flows/management-http-to-broker.md` during a later flow-specific pass

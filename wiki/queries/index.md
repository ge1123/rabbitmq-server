# Graph Query Recipes

Record graph strategies that worked, including enough detail to repeat them.
Avoid pasting complete result sets.

## Broker Core Symbol Lookup

Use for questions about channel, queue, exchange, connection, or core broker
behavior.

1. Read `wiki/components/index.md` and route broker-core questions to `deps/rabbit`
2. Run `list_projects` and select the `rabbitmq-server` project for this checkout
3. Run `index_status` and confirm the graph is ready
4. Start with `search_graph(name_pattern=".*rabbit_channel.*", limit=20)`
5. Narrow from returned `qualified_name` and `file_path`
6. For channel method handling, use
   `search_graph(name_pattern="handle_method", qn_pattern=".*rabbit_channel.*", limit=20)`
7. Use `trace_path(function_name=<qualified name>, direction="both", depth=1)`
8. Use `get_code_snippet` or source reads for clause-specific behavior

## Erlang Multi-Clause Functions

Graph relationships identify function-level callers and callees, not every
semantic branch inside multi-clause Erlang functions. Use `trace_path` to find
relationship boundaries, then inspect neighboring source for pattern-matched
clauses, guards, and fall-through behavior.

## Avoiding Broad File Patterns

Prefer `name_pattern`, `qn_pattern`, `query`, and returned `file_path` checks.
Use broad `file_pattern` filters only after a symbol search is too noisy or when
the component route is already known.

## Template

Copy `wiki/templates/query.md` for query notes that should be preserved.

## Query Notes

 * `authentication-authorization.md`: auth coordinator, backend callback,
   SASL mechanism, and protocol adapter handoff trace strategy
 * `broker-core.md`: broker-core startup, boot-step, and dynamic supervision
   discovery strategy
 * `build-test-release.md`: build, test, release, packaging, Selenium, CI, and
   release-note routing and text-search strategy
 * `wiki/components/common-libraries.md`: shared records/types, generated AMQP
   framing, shared helper, auth mechanism, password hashing, and common Make
   glue routing
 * `wiki/components/cli.md`: CLI command discovery, validation, formatter/printer,
   and node RPC routing strategy
 * `connection-channel.md`: AMQP 0-9-1 connection/channel lifecycle and method
   dispatch trace strategy
 * `management-http.md`: management HTTP API handler, UI route, and broker
   handoff trace strategy
 * `wiki/components/management.md`: management listener, dynamic route extension,
   auth helper, stats/cache, management-agent, and static UI routing strategy
 * `protocol-adapters.md`: MQTT, STOMP, stream, and WebSocket adapter entry
   point and handoff trace strategy
 * `queue-exchange-routing.md`: AMQP 0-9-1 queue/exchange declaration,
   binding, routing, and queue-type delivery trace strategy
 * `federation-shovel-streams.md`: federation link, shovel worker, and stream
   queue/storage boundary trace strategy

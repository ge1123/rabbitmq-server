# Common Libraries

## Purpose

Use `deps/rabbit_common` for shared Erlang records, types, AMQP 0-9-1 framing
types and codecs, binary frame parsing/generation, environment/config context
helpers, SASL mechanism behaviour, password hashing helpers, registry/runtime
parameter behaviours, shared metrics helpers, and utility modules reused by
broker core, protocol plugins, CLI-adjacent code, management, tests, and build
glue.

This component is a routing target when the question is about shared data
shapes or helper boundaries. It is usually not the final owner for broker
semantics: queue, exchange, authz, channel, listener, and protocol adapter
behavior generally crosses from `rabbit_common` records or behaviours into
`deps/rabbit` or a plugin component.

## Entry Points

 * Shared records and canonical cross-component data shapes:
   `deps/rabbit_common/include/rabbit.hrl`,
   `deps/rabbit_common/include/resource.hrl`, and
   `deps/rabbit_common/src/rabbit_types.erl`
 * AMQP 0-9-1 method/property records, exception constants, and generated
   framing type definitions:
   `deps/rabbit_common/include/rabbit_framing.hrl`,
   `deps/rabbit_common/src/rabbit_framing.erl`, and
   `deps/rabbit_common/src/rabbit_framing_amqp_0_9_1.erl`
 * AMQP field-table and frame binary helpers:
   `deps/rabbit_common/src/rabbit_binary_parser.erl`,
   `deps/rabbit_common/src/rabbit_binary_generator.erl`, and AMQP table helpers
   in `deps/rabbit_common/src/rabbit_misc.erl`
 * Runtime environment and source-run configuration context:
   `deps/rabbit_common/src/rabbit_env.erl`
 * General shared utility and error construction:
   `deps/rabbit_common/src/rabbit_misc.erl`
 * Shared auth boundary types and behaviours:
   `deps/rabbit_common/src/rabbit_auth_mechanism.erl`,
   `deps/rabbit_common/src/rabbit_password.erl`,
   `deps/rabbit_common/src/rabbit_password_hashing*.erl`, and auth-related
   types in `rabbit_types.erl`
 * Data coercion and user-supplied value normalization:
   `deps/rabbit_common/src/rabbit_data_coercion.erl`
 * RabbitMQ-specific make glue shared by components:
   `deps/rabbit_common/mk/*.mk` and generated framing rules in
   `deps/rabbit_common/development.post.mk`

## Graph Query Strategy

Start with graph lookup, but expect `rabbit_common` to need narrower follow-up
than ordinary component routing because it contains records, macros, generated
code, behaviours, and broad utility modules.

Useful graph starts:

 * `search_graph(query="framing table record config schema auth helper", file_pattern="deps/rabbit_common/.*", limit=50)` to discover broad shared-library vocabulary, then manually check returned `file_path` because full-text ranking can surface tests and callers outside `rabbit_common`
 * `search_graph(name_pattern=".*rabbit_framing.*", limit=50)` to find the type wrapper module, generated AMQP 0-9-1 module, generated include target, and generated file nodes
 * `search_graph(name_pattern=".*rabbit_data_coercion.*|.*rabbit_parameter_validation.*", limit=50)` to distinguish shared coercion ownership in `rabbit_common` from broker-specific parameter validation in `deps/rabbit`
 * `search_graph(qn_pattern=".*deps\\.rabbit_common\\.src\\.rabbit_password.*|.*deps\\.rabbit_common\\.src\\..*hashing.*", limit=50)` for password hashing helper ownership
 * `search_graph(qn_pattern=".*deps\\.rabbit_common\\.src\\.rabbit_env.*|.*deps\\.rabbit_common\\.src\\.rabbit_misc.*|.*deps\\.rabbit_common\\.src\\.rabbit_binary.*", limit=80)` for high-use shared environment, utility, and binary helpers
 * `search_graph(qn_pattern=".*deps\\.rabbit_common\\.src\\..*(auth|credentials|permission).*", limit=50)` for shared auth behaviours, auth attempt metrics, and auth type definitions
 * `query_graph` over functions whose `file_path` starts with `deps/rabbit_common/` to rank by `in_degree` and `out_degree`; use the results as hints, then verify exact behavior from source
 * `trace_path` selectively for representative helpers such as
   `rabbit_misc:amqp_error/4`, `rabbit_binary_parser:parse_table/1`, or
   `rabbit_env:get_context/1`; do not expect complete cross-component traces for
   macros, records, generated code, behaviours, or dynamic module calls

## Behavioral Notes

`rabbit_common` owns shared data shape more than business behavior.
`include/rabbit.hrl` defines records passed across broker core and plugins,
including users, permissions, connections, content, exchanges, bindings,
deliveries, listeners, plugins, and AMQP errors. `rabbit_types.erl` exports
typed aliases over those records and points many message/content/binding types
back to `rabbit_framing` types.

AMQP 0-9-1 framing is generated. `rabbit_framing.erl` is a small type wrapper
around the generated `rabbit_framing_amqp_0_9_1` module, while
`rabbit_framing.hrl` and `rabbit_framing_amqp_0_9_1.erl` are generated from
`rabbitmq_codegen` JSON specs through `development.post.mk`. Edit questions
about method records, class IDs, exception constants, or encode/decode clauses
must account for code generation rather than treating those files as hand-owned
source.

Binary parsing and generation are AMQP 0-9-1 support helpers, not protocol
adapter dispatch. Use `rabbit_binary_parser` and `rabbit_binary_generator` for
frame/table encoding details, then route method handling, channel lifecycle,
and queue/exchange semantics to broker core.

`rabbit_env` centralizes runtime context for paths, node names, default user and
vhost, enabled plugin files, config file paths, and source-run environment
conversion. It is shared by broker startup, CLI config reads, and build/run
support, so source verification is usually needed before changing assumptions
about config precedence or generated environment values.

`rabbit_auth_mechanism` defines the shared SASL mechanism behaviour used by
AMQP 0-9-1, AMQP 1.0, Stream, and MQTT 5.0 protocol readers. It deliberately
stops at extracting credentials and delegating authentication through
`rabbit_access_control`; backend semantics live in `deps/rabbit` and
`deps/rabbitmq_auth_backend_*`.

`rabbit_password` and `rabbit_password_hashing*` own shared password hashing
helpers. The graph shows `rabbit_password:hash/2` building a salted binary from
`generate_salt/0` and `salted_hash/3`, but auth backend policy and user storage
remain broker-core concerns.

`rabbit_data_coercion` is a small conversion boundary for values that need to
move between binaries, lists, atoms, maps, proplists, booleans, and Unicode
forms. Broker-specific validation such as parameter validation belongs outside
`rabbit_common`.

## Cross-Component Links

 * Broker core uses `rabbit_common` records/types heavily, but queue, exchange,
   authz, boot, connection, and channel semantics belong in `deps/rabbit`
 * Protocol adapters use `rabbit_auth_mechanism`, `rabbit_types`, and AMQP table
   helpers, then hand authentication, routing, or delivery work to broker core
 * Management and CLI paths often format or coerce shared AMQP tables and
   runtime context, but route HTTP API and command behavior to their own
   components
 * Auth mechanisms and password hashing helpers are shared library boundaries;
   route access-control decisions to `rabbit_access_control` and backend modules
 * Build/run questions involving common Make glue can start in
   `deps/rabbit_common/mk`, but release/dependency/version ownership remains in
   top-level build files

## Investigation Pitfalls

 * Full-text `search_graph` with `file_pattern="deps/rabbit_common/.*"` can
   still return high-ranking tests and callers outside `rabbit_common`; check
   `file_path` before using a hit as ownership evidence
 * Generated AMQP framing files contain real records, types, and encode/decode
   clauses, but durable edit ownership is the codegen input and
   `development.post.mk` generation path
 * Records and macros from `.hrl` files do not behave like ordinary function
   call edges in the graph, so include-file ownership often requires source
   reads
 * `trace_path` can under-report cross-component use for shared helpers reached
   through macros, behaviours, dynamic module calls, generated code, or Elixir
   callers
 * `rabbit_misc` is intentionally broad; do not infer business ownership from a
   helper name without tracing or source verification
 * Graph degree is a routing hint, not proof of runtime importance; verify
   multi-clause Erlang functions and generated clauses from source when behavior
   matters

## Related Wiki Pages

 * `components/index.md` for coarse ownership routing
 * `queries/index.md` for reusable graph query patterns
 * `flows/authentication-authorization.md` for auth handoff after shared SASL
   mechanisms
 * `flows/connection-channel-lifecycle.md` for AMQP reader/channel behavior
   after framing decode
 * `flows/queue-exchange-routing-delivery.md` for core queue, exchange, binding,
   routing, and delivery behavior
 * `flows/protocol-adapters.md` for MQTT, STOMP, stream, AMQP adapter, and
   WebSocket handoff boundaries
 * `queries/build-test-release.md` for build, test, release, packaging, and
   source-run routing

# Query Note: Queue, Exchange, Routing, and Delivery

## Question

How should agents find AMQP 0-9-1 queue declaration, exchange declaration,
binding, routing, and delivery paths without copying large `rabbit_channel`
method inventories?

## Recommended Graph Tools

 * `search_graph`
 * `trace_path`
 * `get_code_snippet`
 * `query_graph`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern="^(handle_method|declare|bind|route|deliver|process_routing_mandatory)$",
  qn_pattern=".*deps\\.rabbit\\.src\\.(rabbit_channel|rabbit_amqqueue|rabbit_exchange|rabbit_binding|rabbit_queue_type|rabbit_classic_queue).*",
  limit=100)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern=".*(binding|route|publish|deliver|lookup|add|remove).*",
  qn_pattern=".*deps\\.rabbit\\.src\\.(rabbit_binding|rabbit_exchange|rabbit_channel|rabbit_queue_type|rabbit_classic_queue).*",
  limit=100)

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_channel.handle_method",
  direction="outbound",
  depth=1,
  mode="calls")

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_channel.binding_action_with_checks",
  direction="both",
  depth=1,
  mode="calls")

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_exchange.route",
  direction="both",
  depth=2,
  mode="calls")

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_channel.deliver_to_queues",
  direction="both",
  depth=2,
  mode="calls")

get_code_snippet(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_amqqueue.declare",
  include_neighbors=true)

get_code_snippet(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_exchange.route",
  include_neighbors=true)

get_code_snippet(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_queue_type.deliver0",
  include_neighbors=true)
```

## Failed or Noisy Queries

```text
search_graph(query="queue declare exchange declare queue bind exchange bind route publish deliver basic get consume", limit=60)
```

This is too broad and returns many test helpers and plugin/client utilities.
Prefer core-module `qn_pattern` filters after routing the work to `deps/rabbit`.

```text
query_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="MATCH (f) WHERE f.file_path CONTAINS 'deps/rabbit/src/rabbit_channel.erl' AND f.name IN ['handle_method','binding_action_with_checks','binding_action','deliver_to_queues'] RETURN f.name, f.qualified_name")
```

This returned no rows in this index even though `search_graph` found the same
symbols. Use `search_graph` for symbol discovery; reserve Cypher for patterns
after confirming property shapes in the active graph.

## Source Verification Points

 * `deps/rabbit/src/rabbit_channel.erl`: `#'basic.publish'`,
   `#'queue.declare'`, `#'exchange.declare'`, `#'queue.bind'`,
   `#'exchange.bind'`, `binding_action_with_checks/10`, `binding_action/4`,
   and `deliver_to_queues/3`
 * `deps/rabbit/src/rabbit_amqqueue.erl`: `declare/6`, `declare/7`
 * `deps/rabbit/src/rabbit_exchange.erl`: `declare/7`, `route/2`,
   `route/3`, `route1/4`, `process_alternate/2`, `process_route/2`,
   `validate_binding/2`, `type_to_route_fun/1`
 * `deps/rabbit/src/rabbit_binding.erl`: `add/3`, `remove/3`,
   `binding_checks/2`
 * `deps/rabbit/src/rabbit_queue_type.erl`: `declare/2`, `deliver/4`,
   `deliver0/4`
 * `deps/rabbit/src/rabbit_classic_queue.erl`: `declare/2`, `deliver/3`

## Text Search Notes

Use source text search for AMQP record clauses and dynamic dispatch that graph
edges can understate:

```text
rg -n "#'queue.declare'|#'exchange.declare'|#'queue.bind'|#'exchange.bind'|#'basic.publish'|deliver_to_queues\\(|rabbit_exchange:route|rabbit_queue_type:deliver|binding_action_with_checks" deps/rabbit/src/rabbit_channel.erl
rg -n "^declare\\(|^deliver\\(|^deliver0\\(" deps/rabbit/src/rabbit_queue_type.erl deps/rabbit/src/rabbit_amqqueue.erl deps/rabbit/src/rabbit_classic_queue.erl
rg -n "^declare\\(|^route\\(|^route1\\(|^process_route\\(|^process_alternate\\(|^type_to_route_fun\\(|^validate_binding\\(" deps/rabbit/src/rabbit_exchange.erl
rg -n "^add\\(|^remove\\(|^binding_checks\\(" deps/rabbit/src/rabbit_binding.erl
```

## Notes

Graph evidence is good for finding stable trace starts, but final behavior for
this area depends on Erlang pattern-matched clauses, exchange type callbacks,
queue type callbacks, and metadata-store callbacks. Verify those branches in
source before documenting business behavior.

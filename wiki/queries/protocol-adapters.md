# Query Note: Protocol Adapters

## Question

How should future investigations find MQTT, STOMP, stream, and WebSocket
adapter entry points without copying a generated module inventory?

## Recommended Graph Tools

 * `search_graph`
 * `trace_path`
 * `get_code_snippet`
 * `query_graph`
 * `get_architecture`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")
search_graph(query="MQTT processor publish subscribe check vhost resource topic websocket", limit=40)
search_graph(query="STOMP processor connect send subscribe resource permission websocket", limit=40)
search_graph(query="stream reader auth publish subscribe create stream metadata vhost access", limit=50)
search_graph(query="web mqtt cowboy websocket init rabbit mqtt reader", limit=40)
search_graph(query="web stomp cowboy websocket init rabbit stomp reader", limit=40)
trace_path(function_name="...rabbit_mqtt_processor.process_connect", direction="outbound", depth=2)
trace_path(function_name="...rabbit_mqtt_processor.publish_to_queues_with_checks", direction="outbound", depth=2)
trace_path(function_name="...rabbit_stomp_processor.process_connect", direction="outbound", depth=2)
trace_path(function_name="...rabbit_stomp_processor.do_send", direction="outbound", depth=2)
trace_path(function_name="...rabbit_stream_reader.handle_frame_post_auth", direction="outbound", depth=2)
trace_path(function_name="...rabbit_stream_manager.create", direction="outbound", depth=2)
```

Use `get_code_snippet` on exact qualified names for narrow functions such as
`publish_to_queues_with_checks`, `check_write_permitted`, or WebSocket callback
starts, then use source reads for multi-clause command handlers.

## Failed or Noisy Queries

```text
query_graph:
MATCH (f)
WHERE f.file_path STARTS WITH 'deps/rabbitmq_stream/src/'
  AND (f.name CONTAINS 'permitted'
       OR f.name IN ['handle_frame_post_auth','create','do_create_stream'])
RETURN f.name, f.qualified_name, f.file_path, f.start_line
ORDER BY f.file_path, f.start_line
LIMIT 50
```

This returned no rows in the current graph shape even though `search_graph`
finds the same functions. Prefer `search_graph` and check returned `file_path`
for protocol adapter discovery.

Broad natural-language protocol searches return tests and management modules
alongside adapter code. Narrow by returned `file_path` and qualified name before
tracing.

## Source Verification Points

 * `deps/rabbitmq_mqtt/src/rabbit_mqtt_processor.erl`: MQTT CONNECT,
   publish, subscribe/binding, resource permission, and topic permission
   clauses
 * `deps/rabbitmq_stomp/src/rabbit_stomp_processor.erl`: STOMP CONNECT, SEND,
   SUBSCRIBE, resource permission, and topic authorization clauses
 * `deps/rabbitmq_stream/src/rabbit_stream_reader.erl`: stream pre-auth and
   post-auth command clauses, vhost checks, stream command permission checks,
   and reader initialization
 * `deps/rabbitmq_stream/src/rabbit_stream_utils.erl`: stream permission
   wrapper functions
 * `deps/rabbitmq_stream/src/rabbit_stream_manager.erl`: stream create/delete
   metadata handoff
 * `deps/rabbitmq_stream_common/src/rabbit_stream_core.erl`: stream frame
   parsing and encoding
 * `deps/rabbitmq_web_mqtt/src/rabbit_web_mqtt_*.erl` and
   `deps/rabbitmq_web_stomp/src/rabbit_web_stomp_*.erl`: Cowboy routes,
   subprotocol negotiation, WebSocket initialization, and frame handoff

## Text Search Notes

Use text search for literal WebSocket subprotocol names, configured paths, and
protocol command atoms because these are often embedded in callback clauses or
configuration rather than easy graph relationships.

Useful literals include `sec-websocket-protocol`, `mqtt`, `mqttv3.1`,
`ws_path`, `handle_frame_post_auth`, `declare_publisher`, `create_stream`,
`delete_stream`, `check_write_permitted`, `check_read_permitted`, and
`check_read_or_write_permitted`.

## Notes

Protocol adapter investigations should stop at stable handoff boundaries unless
the question is specifically about core auth, AMQP exchange routing, queue-type
delivery, or stream storage internals. Link to `authentication-authorization.md`
and `queue-exchange-routing-delivery.md` for those core flows.

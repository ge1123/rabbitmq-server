# Query Note: Federation, Shovel, and Streams

## Question

How should future investigations find federation links, shovel worker handoff,
and stream queue/storage boundaries without copying broad module inventories?

## Recommended Graph Tools

 * `list_projects`
 * `index_status`
 * `search_graph`
 * `trace_path`
 * `get_code_snippet`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")
search_graph(query="federation link upstream exchange queue", limit=20)
search_graph(query="shovel worker source destination publish ack", limit=20)
search_graph(query="stream queue storage osiris broker core publish", limit=25)
search_graph(name_pattern=".*rabbit_federation.*", limit=30)
search_graph(name_pattern=".*rabbit_shovel.*", limit=30)
search_graph(name_pattern=".*rabbit_stream.*", limit=30)
trace_path(function_name="...rabbit_federation_exchange_link.go", direction="outbound", depth=1)
trace_path(function_name="...rabbit_federation_queue_link.go", direction="outbound", depth=1)
trace_path(function_name="...rabbit_shovel_worker.handle_msg", direction="outbound", depth=1)
trace_path(function_name="...rabbit_stream_manager.do_create_stream", direction="outbound", depth=1)
trace_path(function_name="...rabbit_stream_queue.deliver", direction="outbound", depth=1)
trace_path(function_name="...rabbit_stream_queue.begin_stream", direction="outbound", depth=1)
```

For exact source reads, first use `search_graph` to get qualified names for
`go`, `consume_from_upstream_queue`, `handle_msg`, `do_create_stream`,
`create_stream`, `deliver`, and `begin_stream`, then use `get_code_snippet` or
direct source reads for multi-clause behavior.

## Failed or Noisy Queries

```text
search_graph(query="shovel worker source destination publish ack", limit=20)
```

This is useful for vocabulary discovery but returns many tests and unrelated
ack/publish helpers. Follow it with `name_pattern=".*rabbit_shovel.*"` and
returned `file_path` checks.

```text
trace_path(function_name="...rabbit_shovel_worker.handle_msg", direction="outbound", depth=1)
```

This only exposes local wrapper calls. The important protocol handoff goes
through dynamic `Mod:*` calls in `rabbit_shovel_behaviour`, so verify the
selected protocol module in source.

## Source Verification Points

 * `deps/rabbitmq_exchange_federation/src/rabbit_federation_exchange_link.erl`:
   exchange-federation link startup, upstream queue declaration, upstream
   internal exchange/binding setup, delivered-message forwarding, and cleanup
 * `deps/rabbitmq_queue_federation/src/rabbit_federation_queue_link.erl`:
   queue-federation startup, upstream suitability check, consume/pause behavior,
   default-exchange republish, and visit-count headers
 * `deps/rabbitmq_shovel/src/rabbit_shovel_worker.erl`: worker lifecycle,
   source/destination connect ordering, topology initialization, message
   dispatch, and termination
 * `deps/rabbitmq_shovel/src/rabbit_shovel_behaviour.erl`: dynamic callback
   dispatch and callback contract
 * `deps/rabbitmq_shovel/src/rabbit_amqp091_shovel.erl`,
   `deps/rabbitmq_shovel/src/rabbit_amqp10_shovel.erl`, and
   `deps/rabbitmq_shovel/src/rabbit_local_shovel.erl`: representative protocol
   module registration, source handling, destination handling, ack/nack, and
   forward behavior
 * `deps/rabbitmq_stream/src/rabbit_stream_manager.erl`: stream command to
   stream-queue declaration handoff and super-stream exchange routing
 * `deps/rabbit/src/rabbit_stream_queue.erl`: queue-type registration,
   declaration, coordinator handoff, Osiris write/read interaction, and replica
   operations

## Text Search Notes

Use text search for `-rabbit_boot_step`, `shovel_protocol`,
`x-queue-type`, `x-internal-purpose`, `x-stream-*` arguments, and test cases
that demonstrate stream or quorum queue interaction because these literals are
often more direct than graph edges.

## Notes

Keep federation/shovel investigations separate from ordinary protocol adapter
work. Federation and shovel are broker-to-broker or in-node transfer components;
stream protocol adapter work belongs in `protocol-adapters.md`, while stream
queue/storage interaction starts at `rabbit_stream_manager` and
`rabbit_stream_queue`.

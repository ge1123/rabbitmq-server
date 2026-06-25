# Flow: Federation, Shovel, and Streams Interaction

## Scope

Cross-component boundaries for federation links and upstreams, shovel worker
handoff, and stream queue/storage interaction with broker core.

This page distinguishes stream protocol adapter behavior from the stream queue
and storage path. Use `protocol-adapters.md` for stream wire parsing,
authentication, and command dispatch before the stream manager boundary.

## Components

 * `deps/rabbitmq_federation_common`
 * `deps/rabbitmq_exchange_federation`
 * `deps/rabbitmq_queue_federation`
 * `deps/rabbitmq_shovel`
 * `deps/rabbitmq_stream`
 * `deps/rabbit/src/rabbit_stream_queue.erl`
 * `deps/rabbit/src/rabbit_stream_coordinator.erl`

## Core Symbols

 * `rabbit_federation_exchange_link:go/1`
 * `rabbit_federation_exchange_link:consume_from_upstream_queue/1`
 * `rabbit_federation_queue_link:go/1`
 * `rabbit_federation_queue_link:consume/3`
 * `rabbit_federation_link_util:forward/8`
 * `rabbit_shovel_worker:init/1`
 * `rabbit_shovel_worker:handle_msg/2`
 * `rabbit_shovel_behaviour:handle_source/2`
 * `rabbit_shovel_behaviour:handle_dest/2`
 * `rabbit_shovel_behaviour:forward/3`
 * `rabbit_stream_manager:create/4`
 * `rabbit_stream_manager:do_create_stream/4`
 * `rabbit_stream_queue:declare/2`
 * `rabbit_stream_queue:create_stream/1`
 * `rabbit_stream_queue:deliver/3`
 * `rabbit_stream_queue:begin_stream/7`

## Trace Starts

 * `rabbit_federation_exchange_link:go` with `direction="outbound"` to find
   exchange-federation link startup, upstream command channel setup, upstream
   queue consumption, and upstream binding synchronization
 * `rabbit_federation_queue_link:go` with `direction="outbound"` to find
   queue-federation link startup, upstream suitability checks, and consumption
   from the upstream queue
 * `rabbit_shovel_worker:handle_msg` with `direction="outbound"` to find the
   worker-to-protocol callback boundary, then verify dynamic module calls in
   `rabbit_shovel_behaviour`
 * `rabbit_stream_manager:create` and `rabbit_stream_manager:do_create_stream`
   with `direction="outbound"` to find the stream protocol command handoff
   into broker queue declaration
 * `rabbit_stream_queue:deliver`, `rabbit_stream_queue:begin_stream`, and
   `rabbit_stream_queue:create_stream` with `direction="outbound"` to inspect
   stream queue interaction with Osiris and the stream coordinator

## Flow

Federation is not a new broker routing path. Federation link processes act as
AMQP clients that connect to an upstream broker, consume from upstream-side
resources, and republish into a downstream exchange or queue through AMQP
channels. Common link utilities own shared connection failure, ack/nack, and
forwarding mechanics.

Exchange federation creates and consumes an upstream queue for the downstream
exchange link. `rabbit_federation_exchange_link:consume_from_upstream_queue/1`
declares a durable upstream queue with federation purpose arguments, applies
prefetch unless the upstream is in `no-ack` mode, and subscribes the link
process. Delivered upstream messages are republished to the downstream exchange
with updated federation routing headers and a cycle/max-hop check before
`rabbit_federation_link_util:forward/8` handles publish/ack coordination.
Binding synchronization creates an upstream internal exchange, binds the
upstream queue, records the active suffix, and deletes the old internal
exchange during normal rotation.

Queue federation is similar but targets a downstream queue. The queue link
checks that the upstream supports consumer priorities, consumes from the
upstream queue, and republishes delivered messages to the default exchange with
the downstream queue name as routing key. Its headers preserve original
exchange/routing-key information on the first forward and maintain visit-count
metadata for loop detection.

Shovel uses a generic worker plus protocol modules rather than hard-coding one
transport path. `rabbit_shovel_worker` starts from static or dynamic
configuration, connects source first, then destination, then initializes
destination and source topology through `rabbit_shovel_behaviour`. Runtime
messages are handed to `handle_source/2` first and `handle_dest/2` second; each
protocol module can return updated state, `not_handled`, or stop reasons.
`rabbit_shovel_behaviour` dispatches all operations through the configured
source or destination module stored in the state map.

AMQP 0-9-1, AMQP 1.0, and local shovel modules register as `shovel_protocol`
providers at boot. The AMQP 0-9-1 source path converts incoming
`basic.deliver` plus content into `mc` message state and calls
`rabbit_shovel_behaviour:forward/3`; destination confirms then ack or nack the
source according to the shovel ack mode. The local shovel path uses core queue
type APIs directly: source initialization calls `rabbit_queue_type:consume/3`
and credits the queue type, while destination publishing is handled inside the
local protocol module. This is the key boundary when a shovel source or
destination is a stream queue: shovel still owns the worker/protocol handoff,
but stream queue behavior is owned by `rabbit_stream_queue`.

Stream creation from the stream plugin is a queue-type declaration. After the
stream protocol adapter reaches `rabbit_stream_manager:create/4`, the manager
translates stream command arguments into AMQP queue arguments, adds
`x-queue-type = stream`, validates supported stream arguments, builds an
`amqqueue` record with `rabbit_stream_queue` as its type, and calls
`rabbit_queue_type:declare/2`. The broker queue-type registry then dispatches
to `rabbit_stream_queue:declare/2`.

`rabbit_stream_queue` is broker core queue-type code, not the stream wire
adapter. It registers the stream queue type by boot step, validates stream queue
compatibility, selects a leader and followers, persists the queue record through
`rabbit_amqqueue:internal_declare/2`, and asks
`rabbit_stream_coordinator:new_stream/2` to create the backing stream. Deletes,
policy changes, replica changes, and leadership transfers also route through
`rabbit_stream_coordinator`.

Stream message storage and delivery are backed by Osiris. Publish delivery to a
stream queue writes the converted message to the leader pid with `osiris:write`;
stateful publish tracks correlation so queue-type actions can settle publisher
confirms and apply block/unblock flow actions. Consumption requires a local
stream replica, initializes an Osiris reader with `osiris:init_reader/4`,
registers offset listeners, and converts offset events into queue delivery
actions.

## Source Evidence

 * `deps/rabbitmq_exchange_federation/src/rabbit_federation_exchange_link.erl`:
   verified delivered-message republish through
   `rabbit_federation_link_util:forward/8`, upstream queue declaration and
   subscription, upstream internal exchange/binding rotation, and cleanup
   behavior on normal shutdown
 * `deps/rabbitmq_queue_federation/src/rabbit_federation_queue_link.erl`:
   verified queue-federation run/pause casts, upstream consumer-priority
   suitability check, downstream default-exchange publish method, and
   federation header visit-count handling
 * `deps/rabbitmq_shovel/src/rabbit_shovel_worker.erl`: verified dynamic shovel
   parsing through the runtime-parameter registry, source-then-destination
   connect ordering, topology initialization ordering, source/destination
   message handoff, and termination cleanup
 * `deps/rabbitmq_shovel/src/rabbit_shovel_behaviour.erl`: verified callback
   contract and state-map dispatch to configured source and destination
   protocol modules
 * `deps/rabbitmq_shovel/src/rabbit_amqp091_shovel.erl`,
   `deps/rabbitmq_shovel/src/rabbit_amqp10_shovel.erl`, and
   `deps/rabbitmq_shovel/src/rabbit_local_shovel.erl`: verified
   `shovel_protocol` boot-step registration and representative source,
   destination, ack/nack, and forward boundaries
 * `deps/rabbitmq_stream/src/rabbit_stream_manager.erl`: verified stream
   manager argument translation, `x-queue-type = stream`, stream queue record
   construction, queue-type declaration handoff, stream deletion, super-stream
   exchange routing, and offset reset through Osiris tracking
 * `deps/rabbit/src/rabbit_stream_queue.erl`: verified stream queue type
   registration, declare/create/delete paths, coordinator handoff, publish
   writes to Osiris, local-reader consume setup, offset event handling,
   replica/leadership operations, and management/info reads from Osiris
   counters

## Graph Notes

`search_graph` works well for first-pass discovery with natural-language
queries such as `federation link upstream exchange queue`, `shovel worker source
destination publish ack`, and `stream queue storage osiris broker core publish`.

Use `name_pattern` searches for module families after the natural-language pass:
`.*rabbit_federation.*`, `.*rabbit_shovel.*`, and `.*rabbit_stream.*`. Narrow by
returned `file_path` because tests and management helpers appear in the same
result sets.

Known graph pitfall: many important Erlang calls in this area are dynamic
callback dispatches, boot-step MFAs, or remote AMQP/Osiris calls. `trace_path`
finds local wrapper boundaries but source verification is required for
`amqp_channel:call`, `amqp_channel:subscribe`, `Mod:handle_source`,
`Mod:forward`, `rabbit_queue_type:declare`, and `osiris:*` behavior.

## Open Questions

 * Full `rabbit_federation_link_util:forward/8` publish/ack internals are a
   useful follow-up if changing federation confirmation behavior
 * `rabbit_stream_coordinator` internals, Ra membership behavior, and Osiris log
   storage internals are intentionally left to a stream storage deep dive
 * Management and Prometheus views for federation/shovel/stream status are
   outside this interaction-boundary page

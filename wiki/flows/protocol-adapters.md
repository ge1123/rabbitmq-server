# Flow: Protocol Adapters

## Scope

MQTT, STOMP, stream protocol, and WebSocket adapter ownership from connection
entry points through core auth/vhost/resource checks and publish, subscribe, or
stream handoff boundaries.

This page intentionally names, but does not duplicate, the core broker flows in
`authentication-authorization.md` and `queue-exchange-routing-delivery.md`.

## Components

 * `deps/rabbitmq_mqtt`
 * `deps/rabbitmq_stomp`
 * `deps/rabbitmq_stream`
 * `deps/rabbitmq_stream_common`
 * `deps/rabbitmq_web_mqtt`
 * `deps/rabbitmq_web_stomp`

## Core Symbols

 * `rabbit_mqtt_processor:process_connect/1`
 * `rabbit_mqtt_processor:publish_to_queues_with_checks/2`
 * `rabbit_mqtt_processor:binding_action_with_checks/5`
 * `rabbit_stomp_processor:process_connect/3`
 * `rabbit_stomp_processor:do_send/3`
 * `rabbit_stomp_processor:do_subscribe/4`
 * `rabbit_stream_reader:handle_inbound_data_pre_auth/4`
 * `rabbit_stream_reader:handle_inbound_data_post_auth/4`
 * `rabbit_stream_reader:handle_frame_post_auth/4`
 * `rabbit_stream_utils:check_resource_access/4`
 * `rabbit_stream_manager:create/4`
 * `rabbit_stream_core:incoming_data/2`
 * `rabbit_stream_core:frame/1`
 * `rabbit_web_mqtt_handler:init/2`
 * `rabbit_web_mqtt_handler:websocket_init/1`
 * `rabbit_web_stomp_handler:init/2`
 * `rabbit_web_stomp_handler:websocket_init/1`

## Trace Starts

 * `rabbit_mqtt_processor:process_connect` with `direction="outbound"` to
   inspect MQTT connection setup, credential validation, vhost checks,
   connection limits, client-id registration, and session initialization
 * `rabbit_mqtt_processor:publish_to_queues_with_checks` with
   `direction="outbound"` to inspect MQTT publish authorization before exchange
   routing and queue delivery
 * `rabbit_stomp_processor:process_connect` with `direction="outbound"` to
   inspect CONNECT credential, vhost, loopback, limit, and connection
   registration checks
 * `rabbit_stomp_processor:do_send` and `rabbit_stomp_processor:do_subscribe`
   with `direction="outbound"` to inspect SEND/SUBSCRIBE permission checks and
   broker handoff
 * `rabbit_stream_reader:handle_frame_post_auth` with `direction="outbound"`
   to inspect stream command handling after authentication
 * `rabbit_stream_manager:create` with `direction="outbound"` to inspect stream
   queue declaration handoff
 * `rabbit_web_mqtt_handler:websocket_init` and
   `rabbit_web_stomp_handler:websocket_init` with `direction="outbound"` to
   inspect WebSocket setup, but verify transport handoff in source because
   Cowboy callback edges can be sparse

## Flow

Protocol plugins own wire parsing, protocol negotiation, connection-local
state, and protocol error mapping. They do not own final auth or broker
resource policy. Once credentials, vhost, client identity, destination, stream,
or routing context are extracted, the adapters hand off to the core access
control flow and then to the broker routing, queue, or stream subsystems.

MQTT is centered on `rabbit_mqtt_processor`. Connection processing checks clean
start/session state, client id, configured vhost, node/vhost/user limits,
credential login, vhost access, loopback restrictions, and credential expiry
timers before registering the non-AMQP connection and initializing
subscriptions. MQTT passes `vhost`, `client_id`, and `password` in login auth
properties, and passes `client_id` in the vhost authz context.

MQTT publish is exchange based. `publish_to_queues_with_checks/2` first checks
write permission on the configured exchange and topic write permission; only
then does `publish_to_queues/2` convert MQTT message state into core message
annotations, look up the exchange, call `rabbit_exchange:route/3`, trace the
publish, and deliver to queues. Retained messages are handed to the retainer
after successful publish. Unauthorized publish is mapped to disconnect or
protocol-level acknowledgement/drop behavior according to MQTT settings and
QoS. MQTT subscribe/unsubscribe binding changes require queue write, exchange
read, and topic read checks before calling `rabbit_binding`.

STOMP is centered on `rabbit_stomp_processor`. `process_connect/3` negotiates
version and heartbeats, derives credentials from CONNECT headers, defaults, or
TLS login state, calls `rabbit_access_control:check_user_login/2`, then checks
vhost existence, vhost access, vhost connection limits, loopback restrictions,
and connection registration before returning CONNECTED. STOMP maps these
failures to STOMP ERROR/BAD CONNECT responses rather than exposing AMQP errors.

STOMP SEND is also exchange based. `do_send/3` resolves the destination,
checks write access on the target exchange, rejects internal exchanges, checks
topic authorization when the exchange is topic-like, builds a core message,
calls `rabbit_exchange:route/3`, resolves queue targets, traces the publish,
and hands routed messages to the adapter delivery helper. STOMP SUBSCRIBE
checks topic read authorization for topic destinations, ensures or creates the
endpoint queue, starts consumption, and binds the queue when the destination
requires a binding.

The stream protocol uses `rabbit_stream_reader` for connection state and frame
command handling, `rabbit_stream_core` in `rabbitmq_stream_common` for stream
wire frame parsing/encoding, `rabbit_stream_utils` for shared permission and
stream helpers, and `rabbit_stream_manager` for stream metadata operations.
`rabbit_stream_core:incoming_data/2` converts binary input into parsed commands
with frame-size enforcement; `rabbit_stream_reader` consumes those commands in
pre-auth and post-auth handlers.

After stream authentication, `rabbit_stream_reader:handle_frame_post_auth/4`
checks permissions per command: publisher declaration and publish paths require
write access to the stream queue resource, subscription and offset/query paths
require read access, metadata can require read or write access, and
create/delete stream paths require configure access. Stream creation is a queue
type handoff: `rabbit_stream_manager:create/4` validates stream queue arguments,
builds an `amqqueue` record with `rabbit_stream_queue` as the type, and calls
`rabbit_queue_type:declare/2`. Message append and read paths hand off to
stream storage/replication libraries such as `osiris` through stream helpers
and reader state, not through the AMQP exchange-routing path.

WebSocket variants are transport adapters over the same protocol processors.
`rabbitmq_web_mqtt` installs Cowboy routes for the configured Web MQTT path,
selects the `mqtt` or `mqttv3.1` WebSocket subprotocol, sets unauthenticated
frame limits, and lets `rabbit_web_mqtt_handler` manage WebSocket state before
MQTT processing. `rabbitmq_web_stomp` installs a Cowboy WebSocket route,
selects supported STOMP subprotocols when offered, builds a
`rabbit_stomp_processor` state in `init_processor_state/1`, and forwards text
or binary WebSocket frames into STOMP data handling. These plugins own HTTP
origin/subprotocol/frame concerns; MQTT/STOMP authorization and broker handoff
remain in the underlying protocol processors.

## Source Evidence

 * `deps/rabbitmq_mqtt/src/rabbit_mqtt_processor.erl`: verified MQTT connect
   setup, core login/vhost/loopback handoff, publish permission checks,
   exchange routing handoff, retained-message handoff, binding permission
   checks, resource permission cache, and topic permission cache
 * `deps/rabbitmq_stomp/src/rabbit_stomp_processor.erl`: verified STOMP
   CONNECT auth/vhost/limit handoff, SEND exchange routing and queue delivery,
   SUBSCRIBE endpoint/binding handoff, resource permission cache, and topic
   authorization context
 * `deps/rabbitmq_stream/src/rabbit_stream_reader.erl`: verified stream
   pre-auth/post-auth frame handling, vhost access check, secret-update vhost
   recheck, command-level permission checks, stream create/delete command
   handoff, and reader initialization through `osiris`
 * `deps/rabbitmq_stream/src/rabbit_stream_utils.erl`: verified stream
   configure/write/read permission wrappers delegate to
   `rabbit_access_control:check_resource_access/4`
 * `deps/rabbitmq_stream/src/rabbit_stream_manager.erl`: verified stream
   creation validates stream arguments, builds a stream queue record, and
   delegates declaration to `rabbit_queue_type:declare/2`
 * `deps/rabbitmq_stream_common/src/rabbit_stream_core.erl`: verified frame
   max enforcement, binary frame parsing into commands, and command encoding
   through `frame/1`
 * `deps/rabbitmq_web_mqtt/src/rabbit_web_mqtt_app.erl`,
   `deps/rabbitmq_web_mqtt/src/rabbit_web_mqtt_handler.erl`, and
   `deps/rabbitmq_web_mqtt/src/rabbit_web_mqtt_stream_handler.erl`: verified
   Cowboy route setup, WebSocket subprotocol selection, max-frame setup,
   connection initialization, and stream-handler header injection
 * `deps/rabbitmq_web_stomp/src/rabbit_web_stomp_listener.erl` and
   `deps/rabbitmq_web_stomp/src/rabbit_web_stomp_handler.erl`: verified Cowboy
   route setup, WebSocket subprotocol selection, processor-state
   initialization, and WebSocket frame forwarding

## Graph Notes

`search_graph` worked well with natural-language queries such as
`MQTT processor publish subscribe check vhost resource topic websocket`,
`STOMP processor connect send subscribe resource permission websocket`, and
`stream reader auth publish subscribe create stream metadata vhost access`.

`trace_path` was useful from the selected processor functions, but do not treat
it as exhaustive for Erlang multi-clause command handlers. Source verification
is needed around specific MQTT packet, STOMP frame, and stream command clauses.

Known graph pitfall: WebSocket Cowboy callbacks can have sparse call edges.
For Web MQTT and Web STOMP, verify the route, subprotocol, and
handler-to-processor handoff in source.

Known graph pitfall: a Cypher query filtering `f.file_path STARTS WITH
'deps/rabbitmq_stream/src/'` returned no rows for stream permission functions
even though `search_graph` found them. Prefer `search_graph` plus returned
`file_path` checks for protocol adapter discovery.

## Open Questions

 * MQTT retained-message storage internals and Web MQTT management examples are
   intentionally out of scope
 * STOMP destination parsing and all temporary queue cleanup cases should be
   investigated separately when changing STOMP destination behavior
 * Stream storage internals in `osiris`, `ra`, and stream queue implementation
   modules are intentionally outside this adapter boundary page

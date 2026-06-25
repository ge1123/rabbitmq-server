# Flow Entry Points

Use this page for cross-component concepts where a single module route is not
enough. Keep each flow as an entry point plus trace strategy; do not pre-fill
large call graphs.

## Flow Notes

 * `authentication-authorization.md`: authN/authZ coordinator, auth backend
   dispatch, SASL mechanisms, protocol adapter handoff, and source verification
   points
 * `connection-channel-lifecycle.md`: AMQP 0-9-1 network connection startup,
   channel supervisor topology, channel creation, and connection/channel cleanup
 * `amqp-method-handling.md`: AMQP 0-9-1 frame decoding and method dispatch
   from `rabbit_reader` to `rabbit_channel`
 * `queue-exchange-routing-delivery.md`: AMQP 0-9-1 queue declaration,
   exchange declaration, binding, publish routing, and queue-type delivery
 * `management-http-to-broker.md`: management HTTP API and UI request routing,
   Webmachine resource callbacks, direct broker mutations, and publish/get
   operation paths
 * `protocol-adapters.md`: MQTT, STOMP, stream, and WebSocket adapter
   ownership, auth/resource handoff, and publish/subscribe/stream boundaries
 * `federation-shovel-streams.md`: federation links/upstreams, shovel worker
   handoff, and stream queue/storage interaction with broker core

## Template

Copy `wiki/templates/flow.md` when a flow has stable trace starts and source
evidence worth preserving.

# Flow: Management HTTP API to broker operations

## Scope

Management plugin HTTP API and UI request paths from Cowboy/Webmachine routing
to broker reads, AMQP-like broker mutations, and message publish/get operations.

## Components

 * `deps/rabbitmq_management/src`: management application, dispatcher, HTTP
   resource modules, and shared request helpers
 * `deps/rabbitmq_management/priv/www`: bundled management UI and API reference
 * `deps/rabbit`: broker operation targets such as `rabbit_channel`

## Core Symbols

 * `rabbit_mgmt_app:start_configured_listeners/2`
 * `rabbit_mgmt_app:start_listener/3`
 * `rabbit_mgmt_app:register_context/3`
 * `rabbit_mgmt_dispatcher:build_dispatcher/1`
 * `rabbit_mgmt_dispatcher:build_routes/1`
 * `rabbit_mgmt_dispatcher:build_module_routes/1`
 * `rabbit_mgmt_dispatcher:dispatcher/0`
 * `rabbit_mgmt_util:direct_request/6`
 * `rabbit_mgmt_util:with_channel/4`
 * `rabbit_mgmt_wm_queue:accept_content/2`
 * `rabbit_mgmt_wm_bindings:accept_content/2`
 * `rabbit_mgmt_wm_exchange_publish:do_it/2`
 * `rabbit_mgmt_wm_queue_get:do_it/2`

## Trace Starts

 * `rabbit_mgmt_app:start_listener/3` with `direction="outbound"` to see
   listener setup, context registration, and dispatcher construction
 * `rabbit_mgmt_dispatcher:dispatcher/0` as the stable API route inventory for
   built-in `rabbit_mgmt_wm_*` resources
 * `rabbit_mgmt_util:direct_request/6` with `direction="outbound"` for
   AMQP-method style HTTP mutations that become
   `rabbit_channel:handle_method/6` RPCs
 * `rabbit_mgmt_wm_exchange_publish:do_it/2` with `direction="outbound"` for
   HTTP message publish semantics through a direct AMQP channel
 * `rabbit_mgmt_wm_queue_get:do_it/2` with `direction="outbound"` for HTTP
   message get semantics through a direct AMQP channel

## Source Evidence

 * `deps/rabbitmq_management/src/rabbit_mgmt_app.erl`: listener startup chooses
   TCP vs TLS context, registers the context handler, and builds the Cowboy
   dispatcher with `rabbit_mgmt_dispatcher:build_dispatcher/1`
 * `deps/rabbitmq_management/src/rabbit_mgmt_dispatcher.erl`:
   `build_module_routes/1` calls `dispatcher/0` from management extension
   modules and prefixes each returned API path with `/api`; `build_routes/1`
   also registers redirects and the static `www` route
 * `deps/rabbitmq_management/src/rabbit_mgmt_dispatcher.erl`: built-in routes
   map `/api/...` paths to `rabbit_mgmt_wm_*` Webmachine resource modules such
   as exchanges, queues, bindings, users, permissions, health checks, reset,
   feature flags, and login/version endpoints
 * `deps/rabbitmq_management/src/rabbit_mgmt_util.erl`:
   `direct_request/6` decodes request properties, builds an AMQP method record,
   selects a target node, and calls `rabbit_channel:handle_method/6` by RPC with
   the request vhost and authenticated user
 * `deps/rabbitmq_management/src/rabbit_mgmt_util.erl`: `with_channel/4` starts
   an `#amqp_params_direct{}` connection as the authenticated user, opens a
   channel, runs the handler callback, maps server-initiated close errors to
   HTTP responses, and closes the channel/connection
 * `deps/rabbitmq_management/src/rabbit_mgmt_wm_queue.erl`: queue creation
   validates the queue name and then calls `direct_request/6` with
   `'queue.declare'`
 * `deps/rabbitmq_management/src/rabbit_mgmt_wm_bindings.erl`: binding creation
   decodes the body, chooses `'exchange.bind'` or `'queue.bind'`, calls
   `direct_request/6`, and returns a Location header for the created binding
 * `deps/rabbitmq_management/src/rabbit_mgmt_wm_exchange_publish.erl`: message
   publish uses `with_channel/4`, enables confirms, publishes with
   `amqp_channel:cast/3`, and reports whether the message was routed
 * `deps/rabbitmq_management/src/rabbit_mgmt_wm_queue_get.erl`: queue get uses
   `with_channel/4`, parses ack mode/count/encoding, issues AMQP
   `basic.get`, optionally returns/rejects messages, and serializes the reply
 * `deps/rabbitmq_management/priv/www/js/dispatcher.js`: UI hash routes call
   `sync_put`, `sync_delete`, `sync_post`, `publish_msg`, and `get_msgs` for
   exchange, queue, binding, vhost, user, permissions, limits, feature flag,
   reset, publish, and get actions
 * `deps/rabbitmq_management/priv/www/js/main.js`: UI request helpers prepend
   `api` to paths, add auth and current-vhost headers, and send synchronous or
   asynchronous JSON requests
 * `deps/rabbitmq_management/priv/www/api/index.html`: the bundled API
   reference documents the public `/api` paths and HTTP verbs; use it for
   literal route/verb confirmation, not as the implementation authority

## Known Patterns and Pitfalls

 * Reads usually start at `to_json/2` callbacks in `rabbit_mgmt_wm_*` modules
   and often use `rabbit_mgmt_db` or management stats helpers rather than
   mutating broker state directly
 * Mutations that correspond to AMQP 0-9-1 methods often use
   `rabbit_mgmt_util:direct_request/6`; source verification is needed to see
   the exact method name and extra fields passed by a handler
 * Message publish and get are not high-throughput paths: they create a direct
   AMQP connection/channel from the HTTP handler path and use confirms or
   `basic.get` semantics
 * Extension plugins can add API routes by exposing modules with the
   `rabbit_mgmt_extension` behaviour; do not treat
   `rabbit_mgmt_dispatcher:dispatcher/0` as the complete route set for an
   enabled broker
 * The graph can find resource callbacks and local helper calls, but dynamic
   Erlang calls such as `Module:dispatcher()` and RPC targets such as
   `rabbit_channel:handle_method/6` need source verification

## Open Questions

 * Plugin-specific management extensions should be investigated from their own
   `rabbit_mgmt_extension` modules when a question is about non-core API paths
 * Authorization behavior for individual endpoints should be traced through the
   relevant resource callback and `rabbit_mgmt_util` authorization helper rather
   than inferred from route names

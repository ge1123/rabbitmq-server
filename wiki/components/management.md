# Component: Management

## Purpose

Use this page to route questions about `deps/rabbitmq_management`: listener
startup, HTTP API dispatch, Webmachine resource ownership, authorization gates,
broker operation handoff, statistics reads, management DB caching, and bundled
UI assets under `priv/www`

The management plugin is both an HTTP API surface and the owner of the classic
management UI. It does not own core broker semantics for queues, exchanges,
permissions, users, vhosts, feature flags, or message transfer; management
resources either read broker state, call core auth/internal modules, or hand
mutations to broker code through AMQP-method or direct-channel paths

## Entry Points

 * Listener setup starts in `deps/rabbitmq_management/src/rabbit_mgmt_app.erl`
   at `rabbit_mgmt_app:start_configured_listeners/2`,
   `rabbit_mgmt_app:start_listener/3`, and
   `rabbit_mgmt_app:register_context/3`
 * Cowboy route construction starts in
   `deps/rabbitmq_management/src/rabbit_mgmt_dispatcher.erl` at
   `rabbit_mgmt_dispatcher:build_dispatcher/1`,
   `rabbit_mgmt_dispatcher:build_routes/1`,
   `rabbit_mgmt_dispatcher:build_module_routes/1`, and
   `rabbit_mgmt_dispatcher:dispatcher/0`
 * HTTP API resources are usually `rabbit_mgmt_wm_*` modules under
   `deps/rabbitmq_management/src`
 * Shared HTTP request, response, authorization wrapper, body decode,
   vhost/id extraction, and broker handoff helpers live in
   `deps/rabbitmq_management/src/rabbit_mgmt_util.erl`
 * Permission and security headers are installed by
   `deps/rabbitmq_management/src/rabbit_mgmt_headers.erl` from each resource
   `init/2`
 * Management auth checks delegate through
   `deps/rabbitmq_web_dispatch/src/rabbit_web_dispatch_access_control.erl`,
   which calls `rabbit_access_control` and enforces management tag classes
 * Statistics query aggregation starts in
   `deps/rabbitmq_management/src/rabbit_mgmt_db.erl`; adaptive query caching is
   in `deps/rabbitmq_management/src/rabbit_mgmt_db_cache.erl`
 * Stats sample formatting lives in
   `deps/rabbitmq_management/src/rabbit_mgmt_stats.erl`
 * Per-node metrics collection, aggregation into management ETS tables, and GC
   are owned by `deps/rabbitmq_management_agent`, especially
   `rabbit_mgmt_agent_sup`, `rabbit_mgmt_metrics_collector`, and
   `rabbit_mgmt_metrics_gc`
 * Bundled UI routes and actions live mainly in
   `deps/rabbitmq_management/priv/www/js/dispatcher.js`
 * UI request plumbing lives mainly in
   `deps/rabbitmq_management/priv/www/js/main.js`

## Graph Query Strategy

Start by confirming the graph project whose `root_path` matches this checkout,
then check `index_status` before graph work. If `index_status` reports an
inconsistent project lookup but `search_graph` works for the listed project
name, record that tool inconsistency in the investigation notes before
continuing

Useful graph starts:

```text
search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern=".*rabbit_mgmt_dispatcher.*",
  limit=50)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern=".*rabbit_mgmt_wm_.*",
  limit=100)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="stats database management cache collector",
  file_pattern="deps/rabbitmq_management/.*",
  limit=50)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern="^(direct_request|with_channel)$",
  qn_pattern=".*rabbit_mgmt_util.*",
  limit=50)
```

Trace broker handoff helpers after finding exact qualified names:

```text
trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbitmq_management.src.rabbit_mgmt_util.direct_request",
  direction="outbound",
  depth=2,
  mode="calls")

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbitmq_management.src.rabbit_mgmt_util.with_channel",
  direction="both",
  depth=2,
  mode="calls")
```

For authorization questions, graph search can be noisy because management
delegates to `rabbitmq_web_dispatch` and core `rabbit_access_control`. Use a
focused search for the resource module first, then verify the selected
`is_authorized/2` callback and helper function in source

For UI questions, use graph only to identify parsed JavaScript functions when
available. Fall back to text search for literal API paths, Sammy hash routes,
and request helper names such as `sync_get`, `sync_put`, `sync_post`,
`sync_delete`, `publish_msg`, `get_msgs`, and `with_req`

## Behavioral Notes

Route construction is extension-driven. `rabbit_mgmt_dispatcher:build_routes/1`
collects modules with the `rabbit_mgmt_extension` behaviour from RabbitMQ
related apps, calls each module's `dispatcher/0`, prefixes returned API paths
with `/api`, then adds redirects, OAuth bootstrap, login, static UI routes, and
the `www` static catch-all

`rabbit_mgmt_dispatcher:dispatcher/0` is the built-in management API route
table, not the complete route set for a running node. Other enabled plugins can
contribute management routes and UI assets through the same extension behaviour

Most resource modules follow Webmachine/Cowboy REST callback ownership:
`allowed_methods/2` declares verbs, `resource_exists/2` performs object lookup,
`to_json/2` serves reads, `accept_content/2` handles PUT or POST payloads, and
`delete_resource/2` handles DELETE

Each resource usually installs common headers in `init/2` by calling
`rabbit_mgmt_headers:set_common_permission_headers/2`. That helper owns CORS,
CSP, HSTS, cache, content-type, XSS, and frame-option response headers; it is
not the same as request authorization

Management authorization is layered. Resource `is_authorized/2` callbacks call
`rabbit_mgmt_util:is_authorized_*`, which delegates to
`rabbit_web_dispatch_access_control`. The web-dispatch helper authenticates
Basic or Bearer credentials through configured auth backends, verifies loopback
rules, requires one of the management tags, and then applies endpoint-specific
predicates such as administrator, monitoring, policymaker, visible vhost, login
vhost, or own connection/channel object

Vhost filtering is not just path parsing. `rabbit_mgmt_util:filter_vhost/4`,
`list_visible_vhosts_names/2`, and `list_login_vhosts_names/2` call
`rabbit_access_control:check_vhost_access/4`; monitor users can see more global
state than ordinary management users, while non-admin list results are filtered
by vhost access

AMQP-method-style mutations use `rabbit_mgmt_util:direct_request/6`. The helper
decodes the body, converts JSON fields into an AMQP 0-9-1 method record,
resolves an optional target node, then RPCs to `rabbit_channel:handle_method/6`
with the request vhost and authenticated user. Queue and exchange declaration
and deletion are examples of this path

Message publish and queue get are separate direct-channel paths. The
publish/get resources call `rabbit_mgmt_util:with_channel/4`, which opens a
direct AMQP connection and channel as the authenticated HTTP user, runs a
callback, maps server close errors into HTTP responses, and closes the channel
and connection afterward

Read endpoints usually do not call broker mutation paths. They read object
state through core modules such as `rabbit_amqqueue`, `rabbit_exchange`,
`rabbit_vhost`, or internal auth modules, then augment list/detail payloads via
`rabbit_mgmt_db` when statistics are enabled

`rabbit_mgmt_db` is the management query aggregator. It fetches node-local data
from running nodes, reads metrics written by management-agent collectors,
combines immutable object facts with simple and detailed stats, and formats
rates at query time with `rabbit_mgmt_stats`

`rabbit_mgmt_db_cache` is an adaptive cache for expensive no-range query paths,
not a source of metrics. It caches per key only while the generating function's
duration multiplied by `management_db_cache_multiplier` justifies keeping the
result, and invalidates when function arguments change

The management agent owns periodic collection. `rabbit_mgmt_agent_sup` starts
one `rabbit_mgmt_metrics_collector` per core metrics table when metrics
collection is enabled, plus storage, delegates, management GC, and metric GC
workers. Collectors read core metrics ETS tables and aggregate into management
ETS tables used by `rabbit_mgmt_db`

The static UI is an API client, not a second backend. `main.js` owns login,
OAuth bootstrapping, request helpers, vhost header propagation, and the
`with_req` path that prepends `api` to UI paths. `dispatcher.js` owns Sammy hash
routes and invokes helpers such as `sync_put`, `sync_delete`, `sync_post`,
`publish_msg`, and `get_msgs`

## Cross-Component Links

 * `deps/rabbitmq_web_dispatch` owns the shared Cowboy/Webmachine dispatch
   context registration and HTTP access-control helper used by management
 * `deps/rabbit/src/rabbit_access_control.erl` owns core authentication,
   authorization backend invocation, loopback checks, and vhost access checks
 * `deps/rabbit/src/rabbit_channel.erl` is the broker-side target for
   AMQP-method-style management mutations through `direct_request/6`
 * `deps/rabbit/src/rabbit_amqqueue.erl`, `deps/rabbit/src/rabbit_exchange.erl`,
   and `deps/rabbit/src/rabbit_vhost.erl` own object lookup and core object
   state read by management resources
 * `deps/rabbit/src/rabbit_auth_backend_internal.erl` owns internal user,
   permission, topic permission, and tag state that management permission/user
   resources expose and mutate
 * `deps/rabbitmq_management_agent` owns node-local metrics collection and
   management ETS sample population; `deps/rabbitmq_management` owns HTTP
   query aggregation and formatting over those samples
 * `wiki/queries/management-http.md` has reusable graph query recipes for
   finding handlers, resource callbacks, utility handoffs, and UI literals
 * `wiki/flows/management-http-to-broker.md` has the HTTP-to-broker mutation
   flow for direct AMQP method calls and direct AMQP channel operations
 * `wiki/flows/authentication-authorization.md` is the next stop when a
   management question depends on auth backend behavior rather than HTTP
   resource routing

## Investigation Pitfalls

 * Do not treat `rabbit_mgmt_dispatcher:dispatcher/0` as a full route inventory
   for an enabled broker; extension modules can add API routes and UI assets
 * Do not infer authorization from route names; inspect the resource
   `is_authorized/2` callback and the exact `rabbit_mgmt_util:is_authorized_*`
   helper it calls
 * Do not confuse common permission headers with permission checks; headers are
   response policy, while request authorization goes through web-dispatch access
   control and core auth modules
 * Do not assume management list endpoints expose all broker state to every
   management user; vhost and user filters are applied after authentication and
   can differ for administrators, monitors, policymakers, and ordinary
   management users
 * Do not route stats questions only to `rabbit_mgmt_db`; collection and GC are
   in `deps/rabbitmq_management_agent`, while query aggregation and response
   formatting are in `deps/rabbitmq_management`
 * Do not treat UI JavaScript route strings as backend authority; use them to
   understand UI behavior and then verify backend modules under `src`
 * Graph traces can miss dynamic Erlang calls such as `Module:dispatcher()` and
   remote calls through `rabbit_misc:rpc_call/4`; verify those paths in source
 * The graph may index JavaScript helper functions from static assets, but text
   search is still the right tool for UI literal paths, templates, and bundled
   reference pages

## Related Wiki Pages

 * `wiki/components/index.md`
 * `wiki/queries/management-http.md`
 * `wiki/flows/management-http-to-broker.md`
 * `wiki/flows/authentication-authorization.md`
 * `wiki/queries/authentication-authorization.md`
 * `wiki/source.md`

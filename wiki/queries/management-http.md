# Query Note: Management HTTP API

## Question

How should agents find management HTTP API handlers, UI request routes, and
broker operation handoffs without treating tests or generated UI bundles as the
implementation source?

## Recommended Graph Tools

 * `search_graph`
 * `trace_path`
 * `get_code_snippet`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern="^(start_listener|build_dispatcher|build_routes|dispatcher|direct_request|with_channel)$",
  qn_pattern=".*deps\\.rabbitmq_management\\.src\\.(rabbit_mgmt_app|rabbit_mgmt_dispatcher|rabbit_mgmt_util).*",
  limit=50)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  name_pattern="^(to_json|accept_content|delete_resource|do_it)$",
  qn_pattern=".*deps\\.rabbitmq_management\\.src\\.rabbit_mgmt_wm_.*",
  limit=100)

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbitmq_management.src.rabbit_mgmt_util.direct_request",
  direction="outbound",
  depth=1,
  mode="calls")

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbitmq_management.src.rabbit_mgmt_util.with_channel",
  direction="outbound",
  depth=1,
  mode="calls")
```

## Failed or Noisy Queries

```text
search_graph(query="rabbitmq management HTTP API handler broker operation queue exchange user vhost", limit=40)
```

This is too broad and returns many management tests, Selenium helpers, and
unrelated HTTP-like symbols. Prefer `rabbit_mgmt_wm_*` callback names and
specific management utility entry points.

## Source Verification Points

 * `deps/rabbitmq_management/src/rabbit_mgmt_dispatcher.erl`: route
   construction and dynamic management extension dispatch
 * `deps/rabbitmq_management/src/rabbit_mgmt_util.erl`: `direct_request/6`,
   `with_channel/4`, request decoding, vhost/user extraction, and HTTP error
   conversion
 * `deps/rabbitmq_management/src/rabbit_mgmt_wm_*.erl`: endpoint-specific
   `to_json/2`, `accept_content/2`, `delete_resource/2`, and `do_it/2`
   behavior
 * `deps/rabbitmq_management/priv/www/js/dispatcher.js` and
   `deps/rabbitmq_management/priv/www/js/main.js`: UI route and request helper
   literals
 * `deps/rabbitmq_management/priv/www/api/index.html`: route and verb
   reference, not implementation authority

## Text Search Notes

Use text search for route literals, UI paths, and generated/static assets:

```text
rg -n "dispatcher\\(|direct_request\\(|with_channel\\(|accept_content\\(|delete_resource\\(|to_json\\(" deps/rabbitmq_management/src
rg -n "sync_put|sync_post|sync_delete|publish_msg|get_msgs|api/" deps/rabbitmq_management/priv/www/js deps/rabbitmq_management/priv/www/api
```

## Notes

Management routes can be extended by plugins through dynamic
`rabbit_mgmt_extension` modules. Use source reads when route completeness,
broker-side RPC targets, or endpoint authorization matters.

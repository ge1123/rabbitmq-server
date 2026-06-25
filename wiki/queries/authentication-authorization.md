# Query Note: Authentication and Authorization

## Question

How should future investigations find authentication, authorization, auth
backend, and protocol-adapter handoff code without copying a broad module
inventory?

## Recommended Graph Tools

 * `search_graph`
 * `trace_path`
 * `get_code_snippet`
 * `query_graph`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")
search_graph(name_pattern=".*rabbit_access_control.*", limit=50)
search_graph(query="authentication authorization backend check user access resource", limit=50)
trace_path(function_name="...rabbit_access_control.check_user_login", direction="both", depth=2)
trace_path(function_name="...rabbit_access_control.check_vhost_access", direction="both", depth=2)
trace_path(function_name="...rabbit_access_control.check_resource_access", direction="both", depth=2)
trace_path(function_name="...rabbit_access_control.check_topic_access", direction="both", depth=2)
search_graph(query="auth mechanism ssl user_login_authentication", limit=50)
search_graph(query="check_vhost_access MQTT STOMP stream web dispatch AMQP session", limit=80)
```

Use `get_code_snippet` on the exact qualified names returned for
`check_user_login`, `try_authenticate`, `try_authorize`,
`try_authenticate_and_try_authorize`, `check_vhost_access`,
`check_resource_access`, and `check_access`.

## Failed or Noisy Queries

```text
query_graph:
MATCH (f)
WHERE f.name IN ['user_login_authentication','user_login_authorization',
                 'check_vhost_access','check_resource_access','check_topic_access']
  AND f.file_path CONTAINS 'auth_backend'
RETURN f.name, f.qualified_name, f.file_path
```

This returned no rows in the current graph shape even though `search_graph`
finds the callback implementations. Prefer `search_graph` by callback name or
natural-language query for backend inventories.

Broad `search_graph(query="authentication authorization backend ...")` returns
release-note and test sections as well as source functions. Narrow by returned
`file_path` and qualified name before tracing.

## Source Verification Points

 * `deps/rabbit/src/rabbit_access_control.erl`: backend chain semantics,
   dynamic backend callbacks, loopback, vhost/resource/topic authorization, and
   credential update
 * `deps/rabbit/src/rabbit_authn_backend.erl` and
   `deps/rabbit/src/rabbit_authz_backend.erl`: callback contracts
 * `deps/rabbit/src/rabbit_reader.erl` and
   `deps/rabbit/src/rabbit_amqp_reader.erl`: AMQP 0-9-1 and AMQP 1.0 SASL and
   vhost handoff
 * `deps/rabbitmq_mqtt/src/rabbit_mqtt_processor.erl`,
   `deps/rabbitmq_stomp/src/rabbit_stomp_processor.erl`,
   `deps/rabbitmq_stream/src/rabbit_stream_reader.erl`, and
   `deps/rabbitmq_web_dispatch/src/rabbit_web_dispatch_access_control.erl`:
   protocol-specific auth error conversion and context construction
 * `deps/rabbit/src/rabbit_auth_mechanism_plain.erl`,
   `deps/rabbit/src/rabbit_auth_mechanism_amqplain.erl`, and
   `deps/rabbitmq_auth_mechanism_ssl/src/rabbit_auth_mechanism_ssl.erl`:
   SASL mechanism handoff into access control

## Text Search Notes

Use text search for exact dynamic callback names and config keys such as
`auth_backends`, `auth_mechanisms`, `user_login_authentication`,
`user_login_authorization`, `check_vhost_access`, `check_resource_access`, and
`check_topic_access`. These are dynamic Erlang module calls and may not appear
as explicit graph edges from `rabbit_access_control` to every backend.

## Notes

Auth investigations often need both graph and source evidence. The graph is
good at finding the coordinator functions and protocol wrappers; source is the
authority for backend-chain ordering, dynamic callback dispatch, and protocol
error conversion.

# Flow: Authentication and Authorization

## Scope

Authentication, authorization, auth backend dispatch, and protocol-adapter
handoff into core access control for broker clients and HTTP entry points.

## Components

 * `deps/rabbit/src/rabbit_access_control.erl`
 * `deps/rabbit/src/rabbit_authn_backend.erl`
 * `deps/rabbit/src/rabbit_authz_backend.erl`
 * `deps/rabbit/src/rabbit_auth_backend_internal.erl`
 * `deps/rabbit/src/rabbit_auth_mechanism_*.erl`
 * `deps/rabbitmq_auth_backend_*`
 * `deps/rabbitmq_auth_mechanism_ssl`
 * Protocol adapters in `deps/rabbit`, `deps/rabbitmq_mqtt`,
   `deps/rabbitmq_stomp`, `deps/rabbitmq_stream`, and
   `deps/rabbitmq_web_dispatch`

## Core Symbols

 * `rabbit_access_control:check_user_login/2`
 * `rabbit_access_control:check_user_login/3`
 * `rabbit_access_control:check_user_pass_login/2`
 * `rabbit_access_control:check_user_loopback/2`
 * `rabbit_access_control:check_vhost_access/4`
 * `rabbit_access_control:check_resource_access/4`
 * `rabbit_access_control:check_topic_access/4`
 * `rabbit_access_control:update_state/2`
 * `rabbit_authn_backend:user_login_authentication/2`
 * `rabbit_authz_backend:user_login_authorization/2`
 * `rabbit_authz_backend:check_vhost_access/3`
 * `rabbit_authz_backend:check_resource_access/4`
 * `rabbit_authz_backend:check_topic_access/4`

## Trace Starts

 * `rabbit_access_control:check_user_login` with `direction="both"` to inspect
   configured auth backend chain behavior
 * `rabbit_access_control:check_vhost_access` with `direction="outbound"` to
   inspect vhost authorization and backend callback dispatch
 * `rabbit_access_control:check_resource_access` with `direction="outbound"` to
   inspect queue/exchange permission checks
 * `rabbit_access_control:check_topic_access` with `direction="outbound"` to
   inspect topic authorization
 * `rabbit_reader:auth_phase` and `rabbit_amqp_reader:auth_phase` with
   `direction="outbound"` to inspect AMQP 0-9-1 and AMQP 1.0 SASL handoff
 * `rabbit_mqtt_processor:check_user_login`,
   `rabbit_stomp_processor:process_connect`, and
   `rabbit_stream_utils:auth_mechanism_to_module` for protocol adapter starts
 * `rabbit_web_dispatch_access_control:is_authorized` for HTTP auth handoff

## Flow

`rabbit_access_control` is the core auth coordinator. Protocol adapters collect
credentials and connection context, then call `check_user_login/2` or
`check_user_login/3`. The two-argument form reads `rabbit.auth_backends`; the
three-argument form is used when the caller supplies a specific backend list,
such as HTTP access-control configuration.

Login dispatch walks the configured backend list until a backend succeeds. A
single backend module performs both authentication and authorization. A tuple
`{AuthNBackend, AuthZBackends}` authenticates with the first module, then runs
passwordless authorization against one or more authorization backends.
`rabbit_auth_backend_cache` is special-cased because its own settings can wrap
both authN and authZ. The resulting `#user{}` stores tags plus
`{Module, Impl}` authorization backend state for later vhost, resource, topic,
and credential-refresh checks.

Backend callback contracts are split between `rabbit_authn_backend` and
`rabbit_authz_backend`. AuthN backends implement
`user_login_authentication/2`; authZ backends implement
`user_login_authorization/2`, `check_vhost_access/3`,
`check_resource_access/4`, `check_topic_access/4`, `update_state/2`, and
`expiry_timestamp/1`. Internal, HTTP, LDAP, OAuth2, cache, dummy, and
internal-loopback backends all plug into this contract.

Vhost, resource, and topic checks run against every authorization backend stored
on the authenticated user. The folds continue while prior checks return `ok`
and stop on the first denial or backend error. Backend `false` and
`{false, Reason}` results become protocol errors; `{error, E}` is logged and
also becomes a protocol error. Vhost checks first require
`rabbit_vhost:exists/1`; resource checks normalize the default exchange name
from `<<>>` to `<<"amq.default">>`.

AMQP 0-9-1 and AMQP 1.0 share the SASL mechanism registry pattern. Readers
advertise configured mechanisms whose modules return true from
`should_offer/1`, resolve the selected mechanism through `rabbit_registry`, and
call the mechanism's `handle_response/2`. PLAIN and AMQPLAIN build password
auth props; the SSL EXTERNAL plugin derives the username from the TLS peer
certificate and calls `check_user_login/2` without a password.

After AMQP 0-9-1 authentication succeeds, `rabbit_reader:auth_phase/2` checks
loopback restrictions, sends `connection.tune`, and stores the authenticated
user. `connection.open` then enforces node, vhost, and user connection limits,
calls `check_vhost_access/4`, checks vhost liveness, and only then moves the
connection to `running`. AMQP 1.0 follows the same split: SASL success stores
the user, and the later open path checks vhost access before sessions start.

MQTT, STOMP, Streams, and HTTP adapt core auth errors into protocol-specific
responses. MQTT adds `vhost`, `client_id`, and `password` auth props and passes
`client_id` in the vhost/topic authorization context. STOMP derives credentials
from CONNECT headers or TLS login state, checks vhost access, and caches
resource permissions. Streams uses the shared auth mechanism registry and
checks vhost access during the open frame. Web dispatch calls
`check_user_login/3`, applies loopback and management-user checks, and uses
`check_vhost_access/4` for request-scoped vhost authorization.

Credential refresh paths reuse the same authorization state. AMQP 0-9-1
`connection.update_secret` and AMQP 1.0 credential update call
`rabbit_access_control:update_state/2`, re-check vhost access with the updated
user, and then propagate the updated auth state to active channels or sessions.

## Source Evidence

 * `deps/rabbit/src/rabbit_access_control.erl`: verified backend-chain login
   semantics, split authN/authZ tuple handling, cache special case, vhost,
   resource, topic, loopback, default-exchange, and credential-update helpers
 * `deps/rabbit/src/rabbit_authn_backend.erl`: verified authN callback
   contract and return shapes
 * `deps/rabbit/src/rabbit_authz_backend.erl`: verified authZ callback
   contract, resource/topic/vhost decisions, state update, and expiry contract
 * `deps/rabbit/src/rabbit_reader.erl`: verified AMQP 0-9-1 SASL auth phase,
   loopback check, `connection.open` vhost authorization, and secret update
 * `deps/rabbit/src/rabbit_amqp_reader.erl`: verified AMQP 1.0 SASL auth
   phase, later vhost checks, and credential update
 * `deps/rabbit/src/rabbit_auth_mechanism_plain.erl`,
   `deps/rabbit/src/rabbit_auth_mechanism_amqplain.erl`, and
   `deps/rabbitmq_auth_mechanism_ssl/src/rabbit_auth_mechanism_ssl.erl`:
   verified PLAIN, AMQPLAIN, and EXTERNAL handoff into access control
 * `deps/rabbitmq_mqtt/src/rabbit_mqtt_processor.erl`: verified MQTT auth
   props, vhost authorization context, resource permission cache, and topic
   context
 * `deps/rabbitmq_stomp/src/rabbit_stomp_processor.erl`: verified STOMP
   CONNECT auth handoff, vhost check, and resource permission cache
 * `deps/rabbitmq_stream/src/rabbit_stream_reader.erl` and
   `deps/rabbitmq_stream/src/rabbit_stream_utils.erl`: verified stream vhost
   access and auth mechanism lookup/resource wrappers
 * `deps/rabbitmq_web_dispatch/src/rabbit_web_dispatch_access_control.erl`:
   verified HTTP login, loopback, management tag, and vhost authorization
   handoff
 * `deps/rabbit/src/rabbit_auth_backend_internal.erl`,
   `deps/rabbitmq_auth_backend_http/src/rabbit_auth_backend_http.erl`,
   `deps/rabbitmq_auth_backend_cache/src/rabbit_auth_backend_cache.erl`,
   `deps/rabbitmq_auth_backend_ldap/src/rabbit_auth_backend_ldap.erl`, and
   `deps/rabbitmq_auth_backend_oauth2/src/rabbit_auth_backend_oauth2.erl`:
   verified representative backend implementations of the shared contracts

## Graph Notes

`trace_path` from `rabbit_access_control:check_vhost_access`,
`check_resource_access`, and `check_topic_access` shows only the local
coordinator and helper calls, because backend module callbacks are dynamic
Erlang calls. Use `search_graph` for callback implementations by name and then
verify representative backend behavior in source.

`trace_path` from protocol adapter wrappers is useful for finding where each
adapter catches or converts core auth errors, but do not copy exhaustive
protocol caller lists into the wiki. Keep this page focused on stable handoff
points and backend semantics.

## Open Questions

 * Management UI OAuth bootstrap and token-login details are intentionally out
   of scope; this page covers the core auth handoff and HTTP access-control
   boundary
 * Authorization behavior inside each external backend's own service, LDAP
   query, or OAuth scope configuration is backend configuration policy, not
   broker-core flow

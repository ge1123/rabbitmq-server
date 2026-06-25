# Component Routing

Use this page to choose an ownership area before graph lookup. Keep entries
coarse; let the graph provide symbol-level detail.

## Core

 * Broker core behavior: `deps/rabbit`
 * Shared internal modules: `deps/rabbit_common`; use
   `common-libraries.md` for records/types, AMQP framing, shared helpers,
   SASL/password helpers, and generated-framing pitfalls
 * Build, release, dependency, and plugin inclusion: `Makefile`, `plugins.mk`,
   `rabbitmq-components.mk`, `erlang.mk`

### Build, Test, Release, and Packaging Overview

Start build/test/release/package questions with
`queries/build-test-release.md`. This area is mostly Make, shell, YAML,
Markdown, and package metadata, so ordinary text search is often the right
evidence tool after confirming graph project status.

Stable routing:

 * Top-level build, source distributions, install, package delegation, and
   generated workflow targets: `Makefile`
 * Release plugin inclusion: `plugins.mk`; verify actual release-pipeline
   inclusion separately because the server-release pipeline overrides this list
 * Component/dependency/version pinning: `rabbitmq-components.mk`
 * Generic Erlang.mk target behavior, Common Test, Dialyzer, Xref, relx, cover,
   and Hex support: `erlang.mk`
 * RabbitMQ-specific source-run and Common Test behavior:
   `deps/rabbit_common/mk/rabbitmq-run.mk` and
   `deps/rabbit_common/mk/rabbitmq-early-plugin.mk`
 * Local packaging: `packaging/Makefile`, `packaging/generic-unix/Makefile`,
   and `packaging/docker-image/Makefile`
 * Installed runtime scripts: top-level `scripts/` for wrappers/autocomplete,
   and `deps/rabbit/scripts/` for RabbitMQ command scripts copied by install
   targets
 * Browser/end-to-end tests: `selenium/README.md`,
   `selenium/run-suites.sh`, `selenium/bin/suite_template`,
   `selenium/test/utils.js`, and suite-list files
 * CI: `.github/workflows/test-make*.yaml`,
   `.github/workflows/test-management-ui*.yaml`,
   `.github/workflows/test-upgrades.yaml`, and release dispatch workflows
 * Release history: `release-notes/`, searched by version, component, issue, or
   feature name rather than summarized

Known pitfalls:

 * `ENABLED_PLUGINS` is converted to `RABBITMQ_ENABLED_PLUGINS` for source-run
   targets but only affects a fresh node; an existing `enabled_plugins` file is
   preserved
 * Top-level OS package targets require `RABBITMQ_PACKAGING_REPO`; local
   generic Unix and Docker image builds delegate under `packaging/`
 * New or moved Common Test suites must be checked against component
   `parallel-ct-sanity-check` grouping when CI coverage is the question
 * Selenium CI builds a package and Docker image before running suite files;
   inspect logs, screenshots, generated config, and artifacts before changing
   browser tests

### Broker Core Overview

Use `deps/rabbit` as the ownership root for node-level broker behavior: the OTP
application callback, core boot orchestration, dynamic broker supervision,
metadata stores, AMQP 0-9-1 reader/channel/queue/exchange behavior, recovery,
vhost reconciliation, alarms, metrics, and listener start coordination.

Stable trace starts for broker startup are:

 * `deps/rabbit/src/rabbit.erl`: `rabbit:boot/0`, `rabbit:start/0`,
   `rabbit:start/2`, `rabbit:start_apps/2`
 * `deps/rabbit/src/rabbit_sup.erl`: `rabbit_sup:start_link/0`,
   `rabbit_sup:start_child/*`, `rabbit_sup:start_supervisor_child/*`,
   `rabbit_sup:start_restartable_child/*`
 * `deps/rabbit/src/rabbit_boot_steps.erl`:
   `rabbit_boot_steps:run_boot_steps/1`,
   `rabbit_boot_steps:find_steps/1`,
   `rabbit_boot_steps:sort_boot_steps/1`,
   `rabbit_boot_steps:run_step/2`

Broker startup is split across prelaunch, core boot steps, and postlaunch.
`rabbit:boot/0` and `rabbit:start/0` call `start_it/1`, which starts
`rabbitmq_prelaunch` and then `rabbit` with different restart semantics.
`rabbit:start/2` completes the second prelaunch phase, starts `rabbit_sup`,
starts `rabbit_ff_controller` before plugin feature-flag refresh, registers the
application process as `rabbit`, loads enabled plugins, runs boot steps for
`[rabbit | Plugins]`, marks `core_started`, and spawns the postlaunch phase.
Postlaunch starts plugin applications one-by-one, loads definitions, starts
client listeners via `rabbit_networking:boot/0`, logs broker readiness, and sets
boot state to `ready`.

`rabbit_sup` is intentionally an empty `one_for_all` root supervisor at init
time. Most durable broker children are attached dynamically by boot-step MFAs
using `rabbit_sup:start_child/*`, `start_supervisor_child/*`, or
`start_restartable_child/*`. Do not infer the core supervision tree only from
`rabbit_sup:init/1`; inspect boot-step declarations and their MFAs.

Boot steps are declared with `-rabbit_boot_step(...)` attributes in
`deps/rabbit/src/rabbit.erl` and other core/plugin modules. The executor gathers
`rabbit_boot_step` module attributes, builds a dependency graph from `requires`
and `enables`, topologically sorts it, checks exported MFAs, and applies each
`mfa` entry. Use text search for `-rabbit_boot_step` declarations because these
attributes are not normal call edges in the graph.

Known graph pitfall: `trace_path` is useful for normal function calls such as
`rabbit:start/2 -> run_prelaunch_second_phase/0`, but boot-step MFAs and some
remote Erlang calls can be invisible or incomplete as call edges. When a
startup behavior depends on boot-step ordering or `rabbit_sup` dynamic child
attachment, verify against `rabbit.erl` boot-step attributes and
`rabbit_boot_steps.erl`.

## Interfaces

 * CLI tools: `deps/rabbitmq_cli`; use `cli.md` for command discovery,
   validation, formatting, printer selection, and node RPC routing
 * Management HTTP API and UI: `deps/rabbitmq_management`; use
   `management.md` for listener/dispatcher/resource/auth/stats/UI ownership
 * Management UI assets: `deps/rabbitmq_management/priv/www`

### Management Overview

Use `deps/rabbitmq_management` for management listener setup, Cowboy route
construction, Webmachine HTTP API resources, management stats/cache reads, and
bundled UI/API-reference assets.

Stable trace starts are:

 * `deps/rabbitmq_management/src/rabbit_mgmt_app.erl`:
   `rabbit_mgmt_app:start_configured_listeners/2`,
   `rabbit_mgmt_app:start_listener/3`,
   `rabbit_mgmt_app:register_context/3`
 * `deps/rabbitmq_management/src/rabbit_mgmt_dispatcher.erl`:
   `rabbit_mgmt_dispatcher:build_dispatcher/1`,
   `rabbit_mgmt_dispatcher:build_routes/1`,
   `rabbit_mgmt_dispatcher:dispatcher/0`
 * `deps/rabbitmq_management/src/rabbit_mgmt_util.erl`:
   `rabbit_mgmt_util:direct_request/6`,
   `rabbit_mgmt_util:with_channel/4`

HTTP API resource modules are usually named `rabbit_mgmt_wm_*`. Look for
`to_json/2` for reads, `accept_content/2` for PUT/POST bodies, and
`delete_resource/2` for DELETE operations. The bundled UI routes live mainly in
`priv/www/js/dispatcher.js`, while request helper functions that prepend `api`
to UI paths live in `priv/www/js/main.js`.

Management API mutations split into two common paths. AMQP-method style
operations such as queue declaration and binding creation use
`rabbit_mgmt_util:direct_request/6`, which source verification shows RPCs to
`rabbit_channel:handle_method/6`. Publish and queue-get endpoints use
`rabbit_mgmt_util:with_channel/4` to open a direct AMQP connection/channel from
the HTTP handler.

Known graph pitfall: extension routes are collected dynamically from modules
with the `rabbit_mgmt_extension` behaviour, and `direct_request/6` reaches
`rabbit_channel:handle_method/6` through `rabbit_misc:rpc_call/4`. Use source
reads when route completeness or the broker-side target matters.

## Protocols and Plugins

 * Streams: `deps/rabbitmq_stream`, `deps/rabbitmq_stream_common`
 * MQTT and STOMP TCP adapters: `deps/rabbitmq_mqtt`,
   `deps/rabbitmq_stomp`
 * MQTT and STOMP WebSocket transport adapters: `deps/rabbitmq_web_mqtt`,
   `deps/rabbitmq_web_stomp`, with HTTP/Cowboy foundations in
   `deps/rabbitmq_web_dispatch`
 * Authentication and authorization: start with
   `deps/rabbit/src/rabbit_access_control.erl`, then route backend-specific
   behavior to `deps/rabbit/src/rabbit_auth_backend_internal.erl`,
   `deps/rabbit/src/rabbit_authn_backend.erl`,
   `deps/rabbit/src/rabbit_authz_backend.erl`,
   `deps/rabbitmq_auth_backend_*`, and
   `deps/rabbitmq_auth_mechanism_ssl`
 * Federation: `deps/rabbitmq_federation*`, `deps/rabbitmq_exchange_federation`,
   `deps/rabbitmq_queue_federation`
 * Shovel: `deps/rabbitmq_shovel*`

Protocol adapter flow routing:

 * Use `flows/protocol-adapters.md` for MQTT, STOMP, stream, and WebSocket
   ownership boundaries before jumping into adapter source
 * Use `flows/federation-shovel-streams.md` for federation link/upstream,
   shovel worker/protocol-module handoff, and stream queue/storage interaction
   boundaries
 * Use `flows/authentication-authorization.md` when the question is about core
   auth backend semantics after an adapter has called `rabbit_access_control`
 * Use `flows/queue-exchange-routing-delivery.md` when the question is about
   core AMQP exchange routing or queue-type delivery after MQTT/STOMP publish
   has handed off to broker routing

Federation/shovel/stream interaction routing:

 * Federation link behavior starts in `rabbit_federation_exchange_link` or
   `rabbit_federation_queue_link`; shared connection, forwarding, and status
   helpers live in `rabbitmq_federation_common`
 * Shovel worker lifecycle starts in `rabbit_shovel_worker`; source/destination
   callbacks dispatch through `rabbit_shovel_behaviour` to registered
   `shovel_protocol` modules such as AMQP 0-9-1, AMQP 1.0, and local shovel
 * Stream protocol command handling belongs to `deps/rabbitmq_stream`, but
   stream queue declaration, publish, consume, replica, and storage interaction
   cross into broker core at `rabbit_stream_manager` and
   `deps/rabbit/src/rabbit_stream_queue.erl`

## Support Areas

 * Runtime wrappers and scripts: `scripts`
 * Packaging: `packaging`
 * Browser and UI tests: `selenium`
 * Release history: `release-notes`
 * CI: `.github/workflows`

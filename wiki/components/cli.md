# CLI

## Purpose

Use `deps/rabbitmq_cli` for RabbitMQ command-line tools implemented in Elixir:
command discovery, argument parsing, command execution, output formatting,
printers, validators, Erlang distribution setup, and node RPC wrappers. Most
broker state changes still execute inside the target RabbitMQ node through
remote calls into `deps/rabbit` or `deps/rabbit_common`.

This page is routing knowledge, not a command inventory. Start from the generic
execution pipeline, then inspect the specific command module only when command
semantics, arguments, target broker module, or output shape matter.

## Entry Points

 * `deps/rabbitmq_cli/lib/rabbitmqctl.ex`: CLI executable entry point and shared execution pipeline
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/parser.ex`: global parsing, command lookup, aliases, command-specific switches and formatter switches
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/command_modules.ex`: command discovery from core modules and enabled plugin modules
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/command_behaviour.ex`: command callback contract and optional callback defaults
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/config.ex`: option, environment, formatter, and printer resolution
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/distribution.ex`: transient CLI Erlang node startup and cookie handling
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/validators.ex`: reusable node and Rabbit application state validators
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/requires_rabbit_app_running.ex`: macro for commands that require the `rabbit` app to be running
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/core/output.ex`: formatter and printer dispatch for single values and streams
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/default_output.ex`: common normalization from command return values to `:ok`, `{:ok, value}`, `{:stream, enum}`, or `{:error, ...}`
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/formatters/*.ex`: built-in output formatters such as table, JSON, CSV, Erlang, string, and report
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/printers/*.ex`: built-in printers such as standard IO, raw standard IO, and file output
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/ctl/rpc_stream.ex`: streaming RPC helper used by list-style commands
 * `deps/rabbitmq_cli/lib/rabbitmq/cli/*/commands/*_command.ex`: individual command modules

## Graph Query Strategy

Confirm the project with `list_projects`, select
`Users-wuchuni-Desktop-system_design-rabbitmq-server`, and verify
`index_status` is `ready` before graph work.

Useful graph starts:

 * `search_graph(query="command router formatter validator rpc node", file_pattern="deps/rabbitmq_cli/.*", limit=50)` to seed discovery across command hooks and helpers
 * `search_graph(name_pattern=".*(run|exec|execute|dispatch|main|main1|load|invoke).*", qn_pattern=".*deps.rabbitmq_cli.*", limit=80)` to find `RabbitMQCtl` and command-loading functions
 * `search_graph(query="formatter output format table json csv printer", file_pattern="deps/rabbitmq_cli/.*", limit=50)` to route output-format questions
 * `search_graph(query="validator validate validation argument node rpc call", file_pattern="deps/rabbitmq_cli/.*", limit=50)` to route argument and execution-environment validation
 * `search_graph(name_pattern=".*(rpc|Rpc|badrpc|remote|call).*", qn_pattern=".*deps.rabbitmq_cli.*", limit=80)` to find `RpcStream`, remote shell, and command-specific RPC helpers
 * `trace_path(function_name="<rabbitmqctl.exec_command qualified name>", direction="outbound", depth=2)` to inspect parse and dispatch calls
 * `trace_path(function_name="<rabbitmqctl.do_exec_parsed_command qualified name>", direction="outbound", depth=2)` to inspect validation, distribution, run, and output calls
 * `trace_path(function_name="<RabbitMQ.CLI.Ctl.RpcStream qualified name>", direction="outbound", depth=2)` to inspect streaming RPC handoff
 * `query_graph` for high fan-out CLI functions with a path predicate scoped to `deps/rabbitmq_cli` when looking for shared helpers worth reading before command modules

Graph caveats from ingest:

 * Broad BM25 searches can return similarly named broker/test symbols outside `deps/rabbitmq_cli`; check `file_path` on every result
 * Some Elixir function nodes were reported with zero line counts and recursive flags that did not reflect source behavior; use graph for routing and source for behavior
 * The MCP transport closed during trace and Cypher calls in this ingest; when that happens after initial graph discovery, record the interruption and continue with narrowly scoped source verification

## Behavioral Notes

`RabbitMQCtl.main/1` starts Elixir and the `rabbitmqctl` application, then
`main1/1` delegates to `exec_command/2`. `exec_command/2` handles empty input,
`--help`, `--version`, and `--auto-complete` rewrites before parsing the command
line with `RabbitMQ.CLI.Core.Parser`.

Command lookup is optimized for core commands first. `Parser.look_up_command/2`
loads the core command map, then falls back to the full command map including
enabled plugin modules, aliases, and autocomplete suggestions. Command modules
are discovered in `CommandModules` by scanning application modules whose names
match `RabbitMQ.CLI.*.Commands`, are loadable, implement
`RabbitMQ.CLI.CommandBehaviour`, and are in the active script scope.

Script scope comes from the script name or `--script-name`. The naming
convention maps command namespaces to scopes, while `scopes/0` can make a
command available from multiple tools. For example `list_queues` is under the
ctl namespace but declares both `:ctl` and `:diagnostics`.

The normal command pipeline is:

 * Parse global and command-specific options
 * Merge global defaults such as node, timeout, and longnames
 * Call command `merge_defaults/2`
 * Start Erlang distribution unless the command overrides `distribution/1`
 * Normalize the target node option
 * Call command `validate/2`
 * Call optional `validate_execution_environment/2`
 * Call command `run/2`
 * Call command `output/2`, usually from `RabbitMQ.CLI.DefaultOutput`
 * Resolve formatter and printer
 * Format and print the result

Validation is deliberately split. `validate/2` checks CLI arguments and
command-specific options; `validate_execution_environment/2` checks state such
as node reachability, whether the Rabbit app is running, or local files. The
`RequiresRabbitAppRunning` macro wires commands to
`Validators.rabbit_is_running/2`, which calls `rabbit_is_running?` through
`:rabbit_misc.rpc_call(node, :rabbit, :is_running, [], timeout)`.

Most node-affecting commands call into the broker through
`:rabbit_misc.rpc_call`. Examples verified during ingest include
`add_user` calling `:rabbit_auth_backend_internal`, `stop_app` calling
`:rabbit.stop`, `eval` calling `:erl_eval.exprs`, and `set_disk_free_limit`
calling `:rabbit_disk_monitor.set_disk_free_limit`. Some helpers use direct
`:rpc.call`, such as cluster node discovery through
`Helpers.with_nodes_in_cluster/3`.

List-style commands often stream output instead of collecting all results.
`ListQueuesCommand` builds broker MFAs such as `:rabbit_amqqueue.emit_info_*`,
then calls `RpcStream.receive_list_items_with_fun/6`. `RpcStream` spawns
broker-side emitters through `:rabbit_control_misc.spawn_emitter_caller`, waits
for `{ref, ...}` messages and `:DOWN` messages, maps timeout and emitter
errors, then applies `InfoKeys.info_for_keys` before output formatting.

Output selection has three layers. Command `output/2` normalizes the raw
`run/2` return value, `Config.get_formatter/2` selects either a loadable
`RabbitMQ.CLI.Formatters.<Name>` from `--formatter` or the command/default
formatter, and `Config.get_printer/2` selects a printer similarly. `Output`
then formats `{:ok, value}` with `format_output/2`, formats `{:stream, enum}`
with `format_stream/2`, and prints via the selected printer.

Machine-readable formatters suppress command banners. JSON and CSV implement
`machine_readable?/0`; `RabbitMQCtl.maybe_print_banner/3` checks
`FormatterBehaviour.machine_readable?/1` before calling command `banner/2`.
Table output is the common human formatter for list commands and adds headers
unless disabled or silent output is requested.

## Cross-Component Links

 * Broker core RPC targets live mainly in `deps/rabbit`; inspect the target MFA named in the command `run/2`
 * Shared RPC and environment helpers live in `deps/rabbit_common`, especially `rabbit_misc`, `rabbit_control_misc`, and `rabbit_env`
 * Authn/authz commands commonly call `rabbit_auth_backend_internal` and should be routed with `components/authn-authz.md` when backend semantics matter
 * Queue, exchange, binding, and channel list commands call broker modules such as `rabbit_amqqueue`, `rabbit_exchange`, `rabbit_binding`, or channel/process info helpers
 * Plugin-provided commands depend on enabled plugin discovery and `Application.spec(app, :modules)`; route plugin command behavior to that plugin after command discovery
 * Build and wrapper questions usually belong to `queries/build-test-release.md`, top-level `scripts/`, and `deps/rabbit/scripts/`, not the Elixir command modules

## Investigation Pitfalls

 * Do not infer command availability from files alone; command modules must be loadable, implement `RabbitMQ.CLI.CommandBehaviour`, and match the active scope
 * Do not skip plugin discovery when a command is missing from core; enabled plugins can contribute command modules
 * Do not treat `validate/2` as the whole preflight path; environment checks are optional callbacks and many commands add them through macros
 * Do not assume `--offline` has one global meaning; some commands define command-specific switches with different semantics, such as `list_queues --offline`
 * Do not assume every command requires the Rabbit app to be running; lifecycle commands and offline/plugin commands may intentionally avoid `RequiresRabbitAppRunning`
 * Do not assume all command output is eagerly materialized; stream output can surface errors while printing
 * Do not use formatter names as module names without `FormatterBehaviour.module_name/1`; CLI formatter selection lowercases and camelizes the user-provided name
 * Do not rely only on graph trace for remote broker targets; RPC module/function atoms in command bodies and `RpcStream` MFA lists need source verification

## Related Wiki Pages

 * `index.md`
 * `components/index.md`
 * `queries/build-test-release.md`
 * `components/broker-core.md`
 * `components/authn-authz.md`
 * `flows/queue-exchange-routing-delivery.md`

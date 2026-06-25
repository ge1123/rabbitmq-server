# Query Note: Connection and Channel Lifecycle

## Question

How do AMQP 0-9-1 network connections create, dispatch to, and clean up channel
processes?

## Recommended Graph Tools

 * `search_graph`
 * `trace_path`
 * `get_code_snippet`
 * `query_graph`
 * `get_architecture`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")
search_graph(name_pattern=".*rabbit_reader.*", limit=40)
search_graph(name_pattern=".*rabbit_channel.*", limit=40)
search_graph(query="connection lifecycle reader start channel open", limit=40)
search_graph(query="AMQP method dispatch handle method channel", limit=40)
trace_path(function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_reader.start_connection", direction="both", depth=1)
trace_path(function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_reader.create_channel", direction="both", depth=1)
trace_path(function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_channel.handle_method", direction="both", depth=1)
get_code_snippet(qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_reader.start_connection")
get_code_snippet(qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_reader.create_channel")
get_code_snippet(qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit_channel.init")
```

## Failed or Noisy Queries

```text
search_graph(qn_pattern=".*deps\\.rabbit\\.src\\.rabbit_reader\\..*", limit=120)
```

This returns useful symbols but is noisy because it includes variables and many
helper functions. Prefer natural-language searches plus exact trace starts once
`rabbit_reader` and `rabbit_channel` are identified.

```text
query_graph(...) over file_path = 'deps/rabbit/src/rabbit_reader.erl'
```

This returned no rows in this index even though `search_graph` found the same
symbols. Use `search_graph` for symbol discovery and reserve Cypher for broader
patterns after confirming property shapes.

## Source Verification Points

 * `deps/rabbit/src/rabbit_connection_sup.erl`: connection supervisor and
   helper supervisor child specs
 * `deps/rabbit/src/rabbit_connection_helper_sup.erl`: dynamic
   `channel_sup_sup` and queue collector child specs
 * `deps/rabbit/src/rabbit_reader.erl`: `start_connection/5`,
   `start_091_connection/2`, `handle_method0/2`, `process_frame/3`,
   `create_channel/2`, `terminate_channels/1`
 * `deps/rabbit/src/rabbit_channel_sup_sup.erl`: per-connection channel
   supervisor-supervisor child spec
 * `deps/rabbit/src/rabbit_channel_sup.erl`: per-channel writer, limiter, and
   channel worker child specs
 * `deps/rabbit/src/rabbit_channel.erl`: `init/1`, `handle_cast/2`,
   `handle_method/2`, `handle_method/3`

## Text Search Notes

Use source text search for pattern-matched Erlang clauses and generated record
names:

```text
rg -n "channel_sup_sup|start_channel_sup_sup|connection.open|connection.tune_ok|process_frame|handle_method0\\(#'connection.open'|handle_method0\\(#'connection.tune_ok'" deps/rabbit/src/rabbit_reader.erl
rg -n "^handle_method\\(" deps/rabbit/src/rabbit_channel.erl
```

## Notes

Supervisor topology is partly encoded in child-spec MFA tuples, so graph call
edges alone can understate lifecycle relationships. Treat source child specs as
the final authority for connection/channel process ownership.

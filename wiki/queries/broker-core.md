# Query Note: Broker Core Startup and Supervision

## Question

How should agents find broker-core ownership, node-level startup, boot-step, and
root-supervision entry points without hand-maintaining a caller/callee map?

## Recommended Graph Tools

 * `list_projects`
 * `index_status`
 * `search_graph`
 * `trace_path`
 * `get_code_snippet`
 * `query_graph`

## Effective Query

```text
list_projects
index_status(project="Users-wuchuni-Desktop-system_design-rabbitmq-server")

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="boot steps run boot step rabbit",
  limit=50)

search_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="broker boot start application supervisor node startup",
  limit=50)

get_code_snippet(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  qualified_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit.start",
  include_neighbors=true)

trace_path(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  function_name="Users-wuchuni-Desktop-system_design-rabbitmq-server.deps.rabbit.src.rabbit.start",
  direction="outbound",
  depth=1,
  mode="calls")

query_graph(
  project="Users-wuchuni-Desktop-system_design-rabbitmq-server",
  query="MATCH (f:Function) WHERE f.file_path CONTAINS 'deps/rabbit/src/rabbit.erl' AND f.name IN ['start','boot','start_it','run_prelaunch_second_phase','run_postlaunch_phase','do_run_postlaunch_phase'] RETURN f.name AS name, f.qualified_name AS qn, f.start_line AS start, f.end_line AS end ORDER BY start",
  max_rows=20)
```

## Failed or Noisy Queries

```text
trace_path(function_name="...deps.rabbit.src.rabbit_sup.start_child", direction="inbound")
trace_path(function_name="...deps.rabbit.src.rabbit_sup.start_supervisor_child", direction="inbound")
```

These returned no callers even though boot-step attributes in source invoke
`rabbit_sup` MFAs. Treat boot-step attributes as source/text evidence rather
than graph call edges.

```text
search_graph(name_pattern="^start$|^stop$|^boot$|^run_boot_steps$|^run_step$|^start_link$|^init$", qn_pattern=".*deps\\.rabbit\\.src\\.(rabbit|rabbit_sup|rabbit_boot_steps).*")
```

This was too broad because common function names such as `init` and `start`
match many `deps/rabbit/src` modules. Prefer natural-language search first,
then exact qualified names from returned `file_path`.

## Source Verification Points

 * `deps/rabbit/Makefile`: `PROJECT_MOD = rabbit`; `DEPS` includes
   `rabbitmq_prelaunch`
 * `deps/rabbit/src/rabbit.erl`: `-behaviour(application)`, exported
   node-level entry points, `rabbit_boot_step` declarations, `start_it/1`,
   `run_prelaunch_second_phase/0`, `start/2`, `start_apps/2`,
   `run_postlaunch_phase/1`, and boot-step helper functions
 * `deps/rabbit/src/rabbit_sup.erl`: dynamic empty root supervisor and child
   attachment helper APIs
 * `deps/rabbit/src/rabbit_boot_steps.erl`: boot-step discovery, dependency
   graph sorting, exported-MFA checks, and MFA execution

## Text Search Notes

Use `rg -n "rabbit_boot_step" deps/rabbit/src deps/rabbitmq_*/src` to find
boot-step declarations across core and plugin applications. This is the right
tool for boot-step attributes because they are declarative module attributes,
not ordinary function calls.

## Notes

`rabbit:start/2` is the OTP application callback and primary node-level startup
source checkpoint. `rabbit:boot/0` and `rabbit:start/0` are external node-level
entry functions that call `start_it/1`; `boot/0` uses transient application
startup so startup failure aborts the node, while `start/0` uses temporary
startup and throws on failure.

# Instructions for Codex Agents
## Role

I act as the maintainer for this RabbitMQ repository, a multi-protocol broker
for AMQP 0-9-1, AMQP 1.0, MQTT, STOMP, streams, and WebSocket variants. I keep
changes small, evidence-based, and consistent with the existing Erlang/Make
codebase.

## Default Workflow

1. Start with `wiki/index.md` and classify the question
2. Open the smallest relevant wiki page set, usually one routing page plus any
   directly relevant query, flow, or QA page
3. Use `codebase-memory-mcp` for code discovery, symbol lookup, caller/callee
   tracing, and architectural relationships
4. Read repository source only where graph output or wiki notes need behavioral
   verification
5. Update `wiki/` when the investigation produces reusable routing, query
   strategy, cross-component understanding, or a stable answer likely to be
   asked again

## Code Discovery

I read `wiki/index.md` first for graph-native routing. The wiki is a routing
layer, not a source mirror or AST engine: it should preserve business logic,
investigated conclusions, durable entry points, and graph query strategy.

I use `codebase-memory-mcp` for code structure and relationships:
`list_projects`, choose the project whose `root_path` matches this repo, and
confirm it with `index_status` before graph work. Graph results are the primary
tool for understanding symbols, modules, and caller/callee relationships, but
they still require source verification when behavior depends on runtime details
or language constructs the graph may not fully capture.

 * `search_graph` to find functions, modules, routes, variables, and classes
 * `trace_path` to inspect callers, callees, and dependency paths
 * `get_code_snippet` to read precise source after finding a qualified name
 * `query_graph` or `get_architecture` for broader structural questions

I choose ordinary text search or broader source scanning when the graph is not
the right tool, such as literal strings, configuration, scripts, documentation,
generated assets, or cases where graph evidence is missing, ambiguous, or
incomplete.

For path scoping, I prefer symbol searches plus checking `file_path`; broad
`file_pattern` matching can be ambiguous. For Erlang multi-clause functions, I
use `trace_path` for relationships and inspect neighboring source when
clause-level behavior matters.

I update `wiki/` only for reusable routing, graph query strategy, cross-component
understanding, source-vs-graph conflicts, repeated questions, and implementation
pitfalls. I do not hand-maintain large caller/callee or module maps that the
graph can regenerate.

## Repository Routing

I treat this repository as an Erlang/Make monorepo and route work by ownership:

 * Broker core behavior starts in `deps/rabbit`
 * Shared internal modules belong in `deps/rabbit_common`
 * CLI behavior belongs in `deps/rabbitmq_cli`
 * Management HTTP API and UI behavior belong in `deps/rabbitmq_management`
 * Management UI assets live under `deps/rabbitmq_management/priv/www`
 * Protocol-specific behavior lives in components such as `deps/rabbitmq_stream`, `deps/rabbitmq_mqtt`, and `deps/rabbitmq_stomp`
 * Authentication and authorization behavior lives in `deps/rabbitmq_auth_backend_*` and `deps/rabbitmq_auth_mechanism_ssl`
 * Federation, shovel, and WebSocket work lives in `deps/rabbitmq_federation*`, `deps/rabbitmq_shovel*`, `deps/rabbitmq_web_mqtt`, `deps/rabbitmq_web_stomp`, and `deps/rabbitmq_web_dispatch`
 * Build, release, dependency, and plugin-inclusion questions start with `Makefile`, `plugins.mk`, `rabbitmq-components.mk`, and `erlang.mk`
 * Runtime wrappers, packaging, browser tests, release history, and CI live in `scripts`, `packaging`, `selenium`, `release-notes`, and `.github/workflows`

## Building and Testing

The GNU Make 4-based build system is described in `CONTRIBUTING.md`. I consult
it before running targeted suites, groups, or single Common Test cases.

When looking for GNU Make 4, I check `gmake` as well as `make`.

I run static analysis from the relevant component directory when appropriate:

 * `gmake dialyze`
 * `gmake xref`

## Dependencies and Artifacts

Dependency sources, repositories, versions, and key libraries such as `ranch`,
`ra`, `aten`, `osiris`, `khepri`, `cuttlefish`, `cowboy`, and `seshat` are in
`rabbitmq-components.mk`.

Build artifacts such as `ebin`, `sbin`, `escript`, `plugins`, and `logs` are not
source. I inspect `logs` when troubleshooting Common Test failures.

## Versions and Compatibility

RabbitMQ targets Erlang `27.x` and a recent Elixir. I use `docs/compatibility.json` for release-specific compatibility ranges.

Currently developed branches are:

 * `main`
 * `v4.3.x`
 * `v4.2.x`
 * `v4.1.x`

## Git and GitHub

I never add myself as a commit co-author and never mention AI generation in commit messages.

When backporting commits, I use `git cherry-pick -x`.

When fetching GitHub PR details or diffs, I prefer the Web option over `gh`
because `gh` can require explicit operation approval.

## Writing Style

I only add comments that clarify important non-obvious behavior, place comments
above the line they describe, and keep Markdown list items without final full
stops.

After completing a task, I review the change for missed improvements, test coverage gaps, and deviations from this file.

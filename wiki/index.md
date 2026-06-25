# RabbitMQ Wiki

This wiki is a durable knowledge and navigation layer for agents working in
`rabbitmq-server`. Use it for business logic, prior investigation conclusions,
stable entry points, and routing into graph tools. It is not a source mirror and
should not duplicate relationship lists that `codebase-memory-mcp` can
regenerate.

## Lookup Order

1. Classify the question and open the smallest relevant wiki page for business
   context, prior conclusions, or routing
2. Use `list_projects` and choose the project whose root matches this checkout
3. Run `index_status` and confirm the graph is ready
4. Use `search_graph`, `trace_path`, `get_code_snippet`, `query_graph`, or
   `get_architecture` to find symbols and relationships
5. Read source when behavior needs proof or graph/wiki evidence is incomplete
6. Write reusable findings back to the wiki when they will reduce future search

## Routing

 * Ownership and component entry points: `components/index.md`
 * Reusable graph query strategies: `queries/index.md`
 * Cross-component conceptual flows: `flows/index.md`
 * Reusable investigated answers: `qa/index.md`
 * Coverage and provenance: `source.md`
 * Wiki change history: `log.md`
 * Deep ingest coordination queue: `ingest-queue.md`

## When to Update This Wiki

Update wiki pages for repeated questions, working graph query recipes,
cross-module understanding, source/graph conflicts, implementation pitfalls, and
stable routing decisions. Do not add exhaustive caller/callee lists, generated
module inventories, or broad source summaries that can drift from the graph.

## Default Graph-Native Pattern

Start with a narrow symbol search. Prefer checking returned `file_path` and
qualified names over broad `file_pattern` searches. For Erlang multi-clause
functions, use `trace_path` for relationships, then inspect nearby source with
`get_code_snippet` or direct source reads when clause-specific behavior matters.

Ordinary text search and broader scans are judgment tools, not a mandatory final
step. Use them when graph tools are not the right fit, such as literals,
configuration keys, scripts, documentation, generated assets, or suspected graph
gaps.

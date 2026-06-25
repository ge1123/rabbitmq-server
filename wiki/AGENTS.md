# RabbitMQ Wiki Agent Guide

This directory is a persistent project knowledge base for agents investigating
`rabbitmq-server`. Treat it as a business-logic and investigation layer, not as
a source mirror and not as a replacement for code graph tools.

## Default Workflow

1. Start from `index.md`
2. Classify the question and open the smallest relevant wiki page set
3. Use wiki pages for business context, prior conclusions, durable entry points,
   and known graph query strategies
4. Use `codebase-memory-mcp` for code structure, symbol lookup, caller/callee
   tracing, and module relationships
5. Verify behavior against repository source when graph or wiki evidence is not
   enough
6. Update the wiki when the investigation produces reusable routing, query
   strategy, cross-component understanding, source-vs-graph conflict notes,
   repeated Q&A, or stable implementation pitfalls

## Page Roles

 * `index.md`: global wiki entry point and lookup order
 * `components/index.md`: ownership and component routing
 * `queries/index.md`: reusable graph query recipes
 * `flows/index.md`: cross-component conceptual flow entry points
 * `qa/index.md`: reusable investigated answers
 * `source.md`: provenance and coverage notes
 * `log.md`: wiki change history
 * `templates/`: starting points for new query, flow, and QA pages

## Evidence Rules

Do not turn guesses into wiki facts. Distinguish verified facts, source-backed
behavior, graph evidence, and inferences.

Prefer `codebase-memory-mcp` for program relationships. Use ordinary text search
or broader source scanning only when it is the better fit for the evidence being
looked up, such as literals, configuration, scripts, documentation, generated
assets, or graph gaps.

When source and graph evidence conflict, prefer behavior verified in repository
source, record the conflict, and update the relevant wiki page if the finding is
likely to help future investigations.

## Maintenance Rules

Keep wiki entries concise and durable. Do not paste exhaustive caller/callee
lists, generated module inventories, or broad source summaries that can drift
from the graph.

When adding a reusable answer, copy `templates/qa.md`. When adding a graph query
strategy, copy `templates/query.md`. When adding a cross-component flow, copy
`templates/flow.md`.

Update directory indexes when adding new pages, and record meaningful wiki
changes in `log.md`. Do not log simple reading or searching.

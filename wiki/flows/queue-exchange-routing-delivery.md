# Flow: Queue, Exchange, Routing, and Delivery

## Scope

AMQP 0-9-1 queue declaration, exchange declaration, binding, publish routing,
and delivery from `rabbit_channel` into queue-type implementations.

## Components

 * `deps/rabbit/src/rabbit_channel.erl`
 * `deps/rabbit/src/rabbit_amqqueue.erl`
 * `deps/rabbit/src/rabbit_exchange.erl`
 * `deps/rabbit/src/rabbit_binding.erl`
 * `deps/rabbit/src/rabbit_queue_type.erl`
 * `deps/rabbit/src/rabbit_classic_queue.erl`

## Core Symbols

 * `rabbit_channel:handle_method/3`
 * `rabbit_channel:binding_action_with_checks/10`
 * `rabbit_channel:binding_action/4`
 * `rabbit_channel:deliver_to_queues/3`
 * `rabbit_amqqueue:declare/7`
 * `rabbit_exchange:declare/7`
 * `rabbit_exchange:route/3`
 * `rabbit_exchange:route1/4`
 * `rabbit_binding:add/3`
 * `rabbit_queue_type:declare/2`
 * `rabbit_queue_type:deliver/4`
 * `rabbit_queue_type:deliver0/4`
 * `rabbit_classic_queue:deliver/3`

## Trace Starts

 * `rabbit_channel:handle_method` with `direction="outbound"` to inspect the
   AMQP method families and find declaration, binding, publish, consume, get,
   ack, and queue-action helpers
 * `rabbit_channel:binding_action_with_checks` with `direction="both"` to
   inspect queue and exchange binding authorization checks before metadata
   mutation
 * `rabbit_channel:deliver_to_queues` with `direction="both"` to inspect
   channel-side post-routing delivery, mandatory returns, confirms, queue
   actions, and stats
 * `rabbit_exchange:route` with `direction="both"` to inspect exchange routing,
   decorators, alternate exchanges, and exchange-to-exchange traversal
 * `rabbit_queue_type:deliver` with `direction="both"` to inspect dispatch from
   routed queue targets to queue-type modules
 * `rabbit_amqqueue:declare` and `rabbit_exchange:declare` with
   `direction="both"` to inspect declaration validation and type dispatch

## Flow

`rabbit_channel:handle_method/3` is the AMQP 0-9-1 entry for queue and exchange
operations once `rabbit_reader` has assembled and cast a channel method. The
top-level clauses for `exchange.declare`, `exchange.bind`, `queue.declare`, and
`queue.bind` extract channel context such as vhost, connection pid,
authorization context, queue collector pid, and user, then delegate to helper
`handle_method/6` clauses before returning the AMQP `*_ok` reply unless the
method is `nowait`.

Queue declaration has separate active, passive, and direct-reply-to paths.
Active `queue.declare` strips CR/LF from names, augments arguments, chooses an
exclusive owner when requested, supports generated server-side names, checks
configure permission on the queue, handles dead-letter exchange permissions,
and calls `rabbit_amqqueue:declare/6`. `rabbit_amqqueue:declare/7` validates
arguments, resolves the queue type from arguments and vhost default type,
constructs the `amqqueue` record, checks queue-type availability and deprecated
argument combinations, then dispatches to `rabbit_queue_type:declare/2`.
`rabbit_queue_type:declare/2` applies policy/decorators, checks queue limits,
and calls the selected queue type module such as `rabbit_classic_queue`.

Exchange declaration similarly splits active and passive paths. Active
`exchange.declare` validates the type, rejects declaration of the default
exchange, checks configure permission, checks alternate-exchange permissions for
new exchanges, calls `rabbit_exchange:declare/7` when the exchange is missing,
and asserts equivalence for the resulting exchange. `rabbit_exchange:declare/7`
checks exchange limits, applies policy/decorators, validates through the
exchange type module, creates or gets the exchange from metadata storage, runs
exchange create callbacks for new exchanges, and emits `exchange_created`.

Queue and exchange binding share `binding_action_with_checks/10`. It normalizes
names, rejects volatile queues as binding targets, checks write permission on
the destination, rejects attempts to bind the default exchange, checks read
permission on the source exchange, performs topic read checks when the source
exists, builds a `#binding{}` record, and calls `binding_action/4`.
`binding_action/4` invokes `rabbit_binding:add/3` or `rabbit_binding:remove/3`
with an inner callback that checks exclusive queue access for queue
destinations. `rabbit_binding:add/3` sorts binding arguments, delegates
metadata creation to `rabbit_db_binding:create/2`, validates binding semantics
through `rabbit_exchange:validate_binding/2`, and emits `binding_created`.

`basic.publish` performs authorization and message preparation before routing:
the channel checks message size, write permission on the exchange, internal
exchange status, expiration and user-id headers, topic write permission, and
message interceptors. It calls `rabbit_exchange:route/3` with
`return_binding_keys => true`, resolves routed queue names through
`rabbit_db_queue:get_targets/1`, traces the publish, then either immediately
calls `deliver_to_queues/3` or stores the delivery in the transaction state.

Routing is exchange-type driven. The default exchange maps each routing key
directly to a queue resource of the same name. Other exchanges call
`rabbit_exchange:route1/4`, which selects the exchange type route function,
adds decorator destinations, adds the configured alternate exchange when the
type returns no destinations, and processes queue and exchange destinations.
Exchange-to-exchange destinations are followed without revisiting exchanges
already seen; queue destinations are accumulated, optionally with binding keys.

`rabbit_channel:deliver_to_queues/3` adds extra blind-copy queue targets, calls
`rabbit_queue_type:deliver/4`, records routed-message counters, processes
mandatory returns before confirms, records publish confirms or rejects, handles
queue actions, and updates exchange/queue stats. `rabbit_queue_type:deliver/4`
wraps `deliver0/4`. `deliver0/4` groups routed targets by queue type and binding
keys, adds binding-key annotations to the message for each group, calls each
queue type module's `deliver/3`, and stores updated per-queue state for
stateful delivery. For classic queues, `rabbit_classic_queue:deliver/3`
prepares the message for store, constructs a `rabbit_basic:delivery`, applies
credit-flow sends when needed, and casts `{deliver, Delivery, false}` to queue
processes through `delegate:invoke_no_result/2`.

## Source Evidence

 * `deps/rabbit/src/rabbit_channel.erl`: verified `basic.publish`,
   declaration top-level clauses, helper `handle_method/6` declaration and
   binding clauses, `binding_action_with_checks/10`, `binding_action/4`, and
   `deliver_to_queues/3`
 * `deps/rabbit/src/rabbit_amqqueue.erl`: verified queue declare argument
   validation, queue-type resolution, queue record construction, queue-type
   feature-flag check, and `rabbit_queue_type:declare/2` dispatch
 * `deps/rabbit/src/rabbit_exchange.erl`: verified exchange declare metadata
   creation path, default-exchange routing, exchange-type routing, decorator
   routing, alternate exchange routing, exchange-to-exchange traversal, and
   binding validation dispatch
 * `deps/rabbit/src/rabbit_binding.erl`: verified binding add/remove use sorted
   args, metadata operations, exchange-type binding validation, inner exclusive
   access checks, and binding-created notification
 * `deps/rabbit/src/rabbit_queue_type.erl`: verified queue-type declare
   dispatch, delivery error wrapper, grouping by queue type and binding keys,
   binding-key message annotation, queue-type `deliver/3` dispatch, and state
   updates
 * `deps/rabbit/src/rabbit_classic_queue.erl`: verified classic queue delivery
   prepares stored messages, creates `rabbit_basic:delivery`, applies
   credit-flow sends, and casts delivery messages to queue pids

## Graph Notes

`trace_path` from `rabbit_channel:handle_method` is useful for finding the
declaration, binding, publish, and delivery helper families, but it should not
be copied as a method inventory. Use source reads around the specific
`#'queue.*'`, `#'exchange.*'`, and `#'basic.publish'` clauses because
multi-clause Erlang dispatch controls behavior.

Known graph pitfall: in this index, `trace_path` from
`rabbit_channel:deliver_to_queues` reported local post-delivery helpers but did
not report the source-visible call to `rabbit_queue_type:deliver/4`. Verify
dynamic queue-type dispatch in `rabbit_channel.erl` and `rabbit_queue_type.erl`
before drawing delivery conclusions from graph edges alone.

## Open Questions

 * AMQP 1.0, MQTT, STOMP, stream protocol adapters, dead lettering, consumer
   delivery, and `basic.get` are intentionally outside this AMQP 0-9-1
   publish-routing-delivery path
 * Queue-type-specific declaration and delivery internals beyond the classic
   queue source checkpoint should be investigated per queue type when needed

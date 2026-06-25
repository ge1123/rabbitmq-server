# Flow: AMQP Method Handling

## Scope

AMQP 0-9-1 frame decoding and method dispatch from the network reader to
channel processes.

## Components

 * `deps/rabbit/src/rabbit_reader.erl`
 * `deps/rabbit/src/rabbit_channel.erl`
 * `deps/rabbit_common/src/rabbit_channel_common.erl`
 * `deps/rabbit/src/rabbit_channel_interceptor.erl`
 * `deps/rabbit_common/src/rabbit_framing_amqp_0_9_1.erl`

## Core Symbols

 * `rabbit_reader:handle_frame/4`
 * `rabbit_reader:handle_method0/2`
 * `rabbit_reader:process_frame/3`
 * `rabbit_command_assembler:analyze_frame/2`
 * `rabbit_command_assembler:process/2`
 * `rabbit_channel_common:do/2`
 * `rabbit_channel_common:do_flow/3`
 * `rabbit_channel:handle_cast/2`
 * `rabbit_channel:handle_method/2`
 * `rabbit_channel:handle_method/3`
 * `rabbit_channel_interceptor:intercept_in/3`
 * `rabbit_framing_amqp_0_9_1:decode_method_fields/2`

## Trace Starts

 * `rabbit_reader:handle_frame` with `direction="outbound"` to inspect
   channel-0 versus channel method routing
 * `rabbit_reader:process_frame` with `direction="both"` to inspect channel
   creation and command assembler handoff
 * `rabbit_channel_common:do` and `rabbit_channel_common:do_flow` with
   `direction="both"` to inspect reader-to-channel cast dispatch
 * `rabbit_channel:handle_cast` with `direction="outbound"` to inspect method
   execution entry
 * `rabbit_channel:handle_method` with `direction="both"` to inspect AMQP method
   clauses and downstream broker operations

## Flow

The reader receives AMQP frames and routes them by channel number and connection
state. Channel 0 frames are connection-level methods.
`rabbit_reader:handle_frame/4` uses `rabbit_command_assembler:analyze_frame/2`,
decodes method fields through
`rabbit_framing_amqp_0_9_1:decode_method_fields/2`, and dispatches to
`rabbit_reader:handle_method0/2` clauses for `connection.start_ok`,
`connection.secure_ok`, `connection.tune_ok`, `connection.open`, connection
close, and other channel-0 protocol operations.

Non-zero channel frames on a running connection go through
`rabbit_reader:process_frame/3`. The reader creates the channel process on
demand if this is the first frame for a channel number, then feeds the frame
into `rabbit_command_assembler:process/2`. The assembler can return only an
updated assembler state, a complete method, or a method plus content.

Complete channel methods are sent to the channel process with
`rabbit_channel_common:do/2`. Methods with content, such as publish, use
`rabbit_channel_common:do_flow/3`; after this path the reader runs
`control_throttle/1` so connection blocking/unblocking follows channel and
resource pressure.

`rabbit_channel:handle_cast/2` handles `{method, Method, Content, Flow}` casts.
For `flow`, it acknowledges reader credit with `credit_flow:ack/1`, then applies
`expand_shortcuts/2` and `rabbit_channel_interceptor:intercept_in/3` before
calling `rabbit_channel:handle_method/2`. Replies are written with `send/2`;
`noreply` continues the channel loop; `stop` terminates normally. AMQP errors
are annotated with the method name and routed through `handle_exception/2`.

`rabbit_channel:handle_method/2` unwraps `{Method, Content}` and delegates to
multi-clause `rabbit_channel:handle_method/3`. The first clauses enforce
channel state: the initial method must be `channel.open`, repeated
`channel.open` is a channel error, closing channels ignore most further methods,
and `channel.close` coordinates with the reader before the channel replies.

AMQP method behavior is mostly encoded as pattern-matched clauses in
`rabbit_channel:handle_method/3`. Examples include `basic.publish`,
`basic.ack`, `basic.get`, `basic.consume`, `basic.qos`, exchange and queue
declare/delete/bind/unbind/purge, transactions, publisher confirms, and
`channel.flow`. Queue and exchange declaration-style clauses often delegate to
helper `handle_method/6` clauses after extracting connection pid,
authorization context, queue collector pid, vhost, and user from channel state.

## Source Evidence

 * `deps/rabbit/src/rabbit_reader.erl`: verified channel-0 frame handling,
   method field decoding, non-zero channel `process_frame/3`, command assembler
   outcomes, and `do`/`do_flow` handoff
 * `deps/rabbit/src/rabbit_channel.erl`: verified method cast handling,
   interceptor entry, `channel.open` state transition, `channel.close`
   reader handshake, state guards, and method-clause dispatch families
 * `deps/rabbit_common/src/rabbit_channel_common.erl`: verified `do/2`,
   `do/3`, and `do_flow/3` cast shape, including `do_flow/3` credit-flow
   tracking
 * `deps/rabbit/src/rabbit_channel_sup.erl`: verified network channels have a
   writer child used by channel replies

## Graph Notes

`search_graph(query="AMQP method dispatch handle method channel")` finds
`rabbit_channel:handle_method` and the generated AMQP framing helpers.
`trace_path` from `rabbit_channel:handle_method` gives useful downstream
operation families but should not be copied as a method inventory. The exact
AMQP dispatch surface is best verified with source text search for
`^handle_method(` because Erlang multi-clause functions are represented at
function level in the graph.

## Open Questions

 * This flow documents AMQP 0-9-1 broker-side dispatch only; client library
   `amqp_client` method handling and AMQP 1.0 are separate flows

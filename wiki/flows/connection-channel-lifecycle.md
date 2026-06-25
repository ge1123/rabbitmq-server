# Flow: Connection and Channel Lifecycle

## Scope

AMQP 0-9-1 network client connection startup, channel process creation,
channel cleanup, and connection close behavior in broker core.

## Components

 * `deps/rabbit/src/rabbit_connection_sup.erl`
 * `deps/rabbit/src/rabbit_connection_helper_sup.erl`
 * `deps/rabbit/src/rabbit_reader.erl`
 * `deps/rabbit/src/rabbit_channel_sup_sup.erl`
 * `deps/rabbit/src/rabbit_channel_sup.erl`
 * `deps/rabbit/src/rabbit_channel.erl`

## Core Symbols

 * `rabbit_connection_sup:start_link/3`
 * `rabbit_reader:start_connection/5`
 * `rabbit_reader:start_091_connection/2`
 * `rabbit_reader:handle_method0/2`
 * `rabbit_reader:process_frame/3`
 * `rabbit_reader:create_channel/2`
 * `rabbit_connection_helper_sup:start_channel_sup_sup/1`
 * `rabbit_channel_sup_sup:start_channel/2`
 * `rabbit_channel_sup:start_link/1`
 * `rabbit_channel:init/1`

## Trace Starts

 * `rabbit_connection_sup:start_link` with `direction="outbound"` to inspect
   connection supervisor setup
 * `rabbit_reader:start_connection` with `direction="both"` to inspect the
   reader entry and connection shutdown cleanup
 * `rabbit_reader:start_091_connection` with `direction="both"` to inspect
   protocol selection after version negotiation
 * `rabbit_reader:create_channel` with `direction="both"` to inspect channel
   creation from received frames
 * `rabbit_channel:init` with `direction="outbound"` to inspect channel process
   state initialization

## Flow

`rabbit_connection_sup:start_link/3` is the Ranch protocol entry for a network
AMQP client connection. It starts a per-connection supervisor, creates AMQP
0-9-1 and AMQP 1.0 helper supervisors, then starts the `rabbit_reader` worker
with both helper supervisor pids.

`rabbit_reader:start_connection/5` owns the network connection process. It
initializes connection metadata, handshake timeout, socket endpoint data,
throttle state, metrics/event state, and enters the receive loop through
`run({rabbit_reader, recvloop, ...})`. Its `after` block fast-closes the socket,
unregisters the connection, records `connection_closed`, and emits the
`connection_closed` event.

After protocol version negotiation chooses AMQP 0-9-1,
`rabbit_reader:start_091_connection/2` removes the AMQP 1.0 helper supervisor,
registers the connection, sends `connection.start` on channel 0, switches the
connection state to `starting`, and continues reading AMQP frames.

The AMQP 0-9-1 channel supervisor tree is attached only after
`connection.open`. In the `connection.open` clause of
`rabbit_reader:handle_method0/2`, the reader checks connection limits, vhost
access, and vhost liveness, sends `connection.open_ok`, starts
`rabbit_channel_sup_sup` through
`rabbit_connection_helper_sup:start_channel_sup_sup/1`, records the
`channel_sup_sup_pid`, switches the connection state to `running`, and emits
connection-created metrics/events.

For non-zero channel frames on a running connection,
`rabbit_reader:process_frame/3` looks up `{channel, Channel}` in the reader
process dictionary. If absent, `rabbit_reader:create_channel/2` checks
negotiated `channel_max`, user channel limits, and node channel limits before
calling `rabbit_channel_sup_sup:start_channel/2`.

`rabbit_channel_sup_sup` is a per-connection `simple_one_for_one` supervisor
whose child is a per-channel `rabbit_channel_sup`. For network channels,
`rabbit_channel_sup:start_link/1` starts a per-channel supervisor, finds its
`rabbit_limiter` and `rabbit_writer` children, then starts the
`rabbit_channel` worker. The reader monitors the channel process and stores both
`{ch_pid, ChPid}` and `{channel, Channel}` entries in its process dictionary.

`rabbit_channel:init/1` initializes the channel state: process metadata,
process group membership, flow mode, default prefetch, limiter,
permission-cache settings, consumer timeout, interceptor context,
metrics/events, operation timeout, and tick timer. The channel starts in
`starting` state and transitions to `running` when it handles the first
`channel.open`.

Channel cleanup is coordinated between reader and channel. A client
`channel.close` makes the channel notify queues and send `{channel_closing,
self()}` to the reader before replying with `channel.close_ok`; this prevents a
new `channel.open` using the same channel number from racing with stale reader
state. The reader removes channel process-dictionary entries in
`channel_cleanup/2`.

Connection close terminates channels before final socket close.
`rabbit_reader:terminate_channels/1` sends `rabbit_channel:shutdown/1` to all
known channels and waits for monitored channel exits up to
`CHANNEL_TERMINATION_TIMEOUT * ChannelCount`. `close_connection/1` deletes
exclusive queues through the queue collector when present and schedules forced
connection termination after the negotiated timeout, capped by
`CLOSING_TIMEOUT`.

## Source Evidence

 * `deps/rabbit/src/rabbit_connection_sup.erl`: verified per-connection
   supervisor startup, helper supervisors, reader child, and helper-supervisor
   removal
 * `deps/rabbit/src/rabbit_connection_helper_sup.erl`: verified dynamic
   `channel_sup_sup` and queue collector child attachment
 * `deps/rabbit/src/rabbit_reader.erl`: verified connection initialization,
   AMQP 0-9-1 start, `connection.tune_ok`, `connection.open`, channel creation,
   frame processing, channel cleanup, and connection close paths
 * `deps/rabbit/src/rabbit_channel_sup_sup.erl`: verified per-connection
   `simple_one_for_one` channel-supervisor-supervisor
 * `deps/rabbit/src/rabbit_channel_sup.erl`: verified network channel children:
   writer, limiter, and channel worker
 * `deps/rabbit/src/rabbit_channel.erl`: verified channel initialization and
   `channel.close` reader handshake

## Graph Notes

`trace_path` shows normal function edges such as
`rabbit_reader:init -> rabbit_reader:start_connection`,
`rabbit_reader:process_frame -> rabbit_reader:create_channel`, and
`rabbit_channel:handle_cast -> rabbit_channel:handle_method`. Some supervisor
child starts are encoded as MFA tuples in child specs and are not always visible
as normal graph call edges. Verify supervisor topology in source when lifecycle
questions depend on child specs.

## Open Questions

 * AMQP 1.0 connection lifecycle is intentionally out of scope for this flow
 * Direct in-node AMQP channels use the `direct` branch of
   `rabbit_channel_sup:start_link/1`; this page focuses on network AMQP 0-9-1

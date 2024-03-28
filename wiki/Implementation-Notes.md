> **Non-normative** This section is not intended to imply that particular
> Lightning Network node software must be used to implement the LSPS
> specifications.
> LSP and client implementers are free to use any implementation of the
> Lightning Network protocol, whether or not they are mentioned here.
> **Rationale** This section exists since a typical "first attempt" at
> designing an LSP would be to spin up an out-of-the-box default
> instance of some popular Lightning Network node software, then
> writing a separate daemon that implements an HTTP(S) server, which
> then calls out to the RPC of the Lightning Network node software to
> make it open channels and perform other operations.
> Specifying the use of the Lightning Network peer-to-peer BOLT8 transport,
> however, requires using specialized hooks or APIs exposed by these
> implementations, instead of the "standard" RPC calls.
> This section acts as a quick guide for how to implement LSPS0 on a few of
> the popular Lightning Network node software, to ease the implementation
> of the LSPS specifications when using common open-source node software.

## CLN

Core Lightning plugins can set the `option_supports_lsps` feature bit in their
response to the `getmanifest` command.
An LSP plugin should set it in both the `node` and `init` fields of the
`featurebits` manifest field.
`featurebits` are hex-encoded strings, and feature bits are encoded as
big-endian variable-length bit fields; the `option_supports_lsps` feature bit
729 would be encoded as:

    "0200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"

An LSPS LSP or LSPS client plugin can hook into the chained hook `custommsg`
to identify `lsps0_message_id` messages.
The hook provides the node ID of the peer via the `peer_id` field.
The hook then provides the entire message (message ID and payload) as a
single hex-encoded string in its `payload` field; the BOLT8 message ID is the
first 4 hex characters.
The `lsps0_message_id` BOLT8 message ID 37913 would be 0x9419, and so the
first 4 characters would be `9419` in the `payload` field.
The rest of the `payload` would then be the hex-encoding of the UTF8-encoding
of the JSON-RPC 2.0 request, notification, or response from the peer.

An LSPS LSP or LSPS client plugin can then send messages to the peer via
the `sendcustommsg` command.
This accepts a `node_id` parameter, which is the node ID of the peer to
send the message to, and a `msg` parameter, which would be formatted
similarly to the `payload` of a `custommsg` hook, including the `9419`
prefix, which is the BOLT8 message ID 37913.

## LDK

LDK as of 0.0.114 requires that you create a delegating implementation
of the `RoutingMessageHandler` trait in order to change the `init` and
`node_announcement` feature bits for LSPS LSP implementations.
You should implement a type that contains an instance of the
LDK-provided `P2PGossipSync` object (or if you want something for
more general use, itself contains an instance of another object
with the `RoutingMessageHandler`), and whose implementation
of `RoutingMessageHandler` simply forwards most of the function
calls to the contained instance.

The only non-delegating functions would be `provided_node_features`
and `provided_init_features`.
The delegating function would call the corresponding function on
the contained instance, then serialize the returned `Features<T>`
object via `Writeable::encode`.
The serialized vector of bytes encodes the 16-bit big-endian length,
followed by the feature bits vector.
The feature bits vector needs to be extended to at least 92 bytes,
packing `0x00` bytes at the *front* of the vector if its length is
increased, and the byte at `length - 92` should be ORed with `0x2`.
Then if the length was changed, the 16-bit big-endian length needs
to be updated.
The 16-bit big-endian length is concatenated with the feature bits
vector, then fed into a `Read` object, then a new `InitFeatures`
or `NodeFeatures` is `Readable::read` from the modified feature
bits serialization vector.

An LSPS LSP or client must implement a type that implements the
`CustomMessageHandler` trait, which extends the `CustomMessageReader`
trait.
Any `lsps0_message_id` messages would be parsed in the
`CustomMessageReader::read` function, which is given the message ID
directly, and the message payload as a `Read`-trait buffer.
The application would need to define a specialized type to hold
its own parsed form of custom messages, possibly making the
`lsps0_message_id` message an `enum` variant.

The specialized type needs to implement `wire::Type`, specifying the
message ID via the `wire::Type::type_id` function.
This should check which variant of the custom message type it is and
return the appropriate message ID.
The same type is used for both incoming messages and outgoing
messages.

Actual handling of incoming requests or responses would be done in the
`CustomMessageHandler::handle_custom_message`, which is handed the
custom message type constructed by `CustomMessageReader::read`, as
well as the node ID of the peer that sent the message.

Outgoing messages will need to be buffered in the type that implements
`CustomMessageHandler`.
Periodically, LDK will call `CustomMessageHandler::get_and_clear_pending_msg`,
which should atomically get the contents of the buffer, empty it, and
return the contents.

## LND

As of 0.16.1, LND supports the [LND `UpdateNodeAnnouncement` RPC][],
which allows changing the `node_announcement` feature bits.
However, custom feature bits are not yet supported for `init`.

For feature bit announcement, use the `feature_updates` parameter.
This is an array of `action` and `feature_bit`.
Use `ADD`/`0` for `action`s to set feature bits.

An LSPS LSP or client can use the [LND `SubscribeCustomMessages` RPC][].
This is a parameterless subscription and will return a continuous stream
of objects with `peer`, `type`, and `data` keys.
`peer` is the node ID of the sender, `type` is the numeric code (which
should be compared against `lsps0_message_id` 37913), and `data` is the
the UTF-8 encoding of the JSON-RPC 2.0 request or response from the peer.
The exact encoding of `peer` and `data` will depend on what you use
for the RPC: they are `bytes` vectors in gRPC and base64-encoded strings
in HTTP.

An LSPS LSP or client can use the [LND `SendCustomMessage` RPC][].
The parameters of this RPC are identical to one object outputted by
the `SubscribeCustomMessage` RPC above.

[LND `UpdateNodeAnnouncement` RPC]: https://lightning.engineering/api-docs/api/lnd/peers/update-node-announcement
[LND `SubscribeCustomMessages` RPC]: https://lightning.engineering/api-docs/api/lnd/lightning/subscribe-custom-messages
[LND `SendCustomMessage` RPC]: https://lightning.engineering/api-docs/api/lnd/lightning/send-custom-message/index.html
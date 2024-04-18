# LSPS0 Transport Layer

| Name    | `transport`        |
| ------- | ------------------ |
| Version | 1                  |
| Status  | Stable             |

## Motivation

The transport layer describes how a client is able to request
services from the LSP.

### Actors

The 'LSP' is the API provider, and acts as the server.
The 'client' is the API consumer.

## Common Schemas

As the transport layer protocol described in the following uses JSON, we often
need to agree upon how particular types of data are encoded into JSON. This is
described in the separate [Common Schemas](common-schemas.md) document.

## Protocol

### Lightning Peer-To-Peer Transport

The Lightning Network [BOLT8][] specification describes the transport
layer for Lightning Network peer-to-peer messages.

[BOLT8]: https://github.com/lightning/bolts/blob/f7dcc32694b8cd4f3a1768b904f58cb177168f29/08-transport.md

Lightning Network BOLT8 describes messages as having two components:

* A 16-bit message ID (encoded in big-endian).
* A variable-length message payload (the remainder of the message).

Access to the endpoints defined in other LSPS specifications is provided
via messages using the [BOLT8][] protocol.

> **Rationale** All clients and LSPs are expected to be Lightning
> Network participants, and need to somehow talk using the BOLT8
> protocol.
> Thus, this does not introduce an additional dependency for the
> client or any LSP, the way an HTTP(S) or gRPC protocol would pull
> in additional dependencies.
>
> During development of this specification, onion messages were
> proposed.
> As a client needs to connect to the LSP node and manage channels
> with that node anyway, and LSP nodes want to be easily contactable
> from IPv4, IPv6, DNS, or TorV3 contact points, the ability of
> onion messages to send to a remote Lightning Network node (that
> might not be directly contactable) was deemed unnecessary for
> most client-LSP communication needs.

All LSPS messages MUST use the [BOLT8][] message ID 37913
(`lsps0_message_id`).

> **Rationale** We indicate a single message ID as this reduces
> the "footprint" of all LSPS specifications to only that single
> message ID, increasing the probability that other protocols
> using the Lightning BOLT8 peer-to-peer transport will be
> compatible with the LSPS specifications.
> The BOLT8 message ID 37913 is odd in order to comply with the ["it's
> OK to be odd"][] rule, and is in the 32768 to 65535 range reserved
> for extensions of the BOLT protocol.
> Some implementations, such as Core Lightning, only expose
> odd-numbered messages to custom message handlers, while others,
> such as LDK, only expose 32768 and above.
> The message ID was otherwise selected randomly and has no
> special meaning.

["it's OK to be odd"]: https://github.com/lightning/bolts/blob/master/00-introduction.md#its-ok-to-be-odd

#### Message Payload Format

The BOLT8 message payload contains the UTF-8 encoding of a
complete JSON object.

The JSON object embedded in the message payload is defined
by the [JSON-RPC 2.0][] protocol.

[JSON-RPC 2.0]: https://www.jsonrpc.org/specification

If a client or LSP receives a BOLT8 message with message ID
37913, it MUST perform the checks below.
If any of these checks fail, then the incoming message has
failed with a "bad message format" error.

* The payload MUST parse as a single complete UTF-8 encoded
  JSON object (i.e. a JSON key-value store or dictionary),
  with optional leading or trailing whitespaces (space, tab,
  line feed, carriage return).
  * For example, "` { } `" would pass this check, but
    "`{`" and "` [ ] `" would not.
* The payload MUST NOT parse as more than a single complete
  JSON object.
  * For example, "` { } { `" and "` { } { }`" would not pass
    this check.
* The payload MUST NOT contain any 0 bytes.
* If the client, the received payload MUST parse to a JSON
  object that is a [JSON-RPC 2.0][] response or notification
  object.
* If the LSP, the received payload MUST parse to a JSON
  object that is a [JSON-RPC 2.0][] request.

In case of a "bad message format" error, the client:

* MUST ignore the message.
* SHOULD log this as an unusual event.
* SHOULD NOT send further BOLT8 messages with ID 37913 to the
  peer on this connection session.
  * SHOULD re-attempt this on reconnection.

In case of a "bad message format" error, the LSP:

* MUST respond with a [JSON-RPC 2.0][] error response with
  the reason "parse error" (`code` = `-32700`, `id` = `null`).
* MUST ignore the message.
* SHOULD log this as an unusual event.

> **Rationale** Requiring a single complete JSON object
> simplifies handling of messages, so that a single message
> maps to a single request or response.
> C uses the NUL character as a string terminator and thus
> embedded 0 bytes may cause problems in implementations
> that pass the JSON string to C code.
> Conversely, we do not require the payload to be terminated
> by a 0 byte / NUL character as it is unnecessary in many
> modern non-C-based languages; C code can copy the buffer
> and append the NUL character if necessary, as the payload
> size is known.
> UTF-8 spends 1 byte per JSON representation character for
> characters in the ASCII range, and we expect most data sent
> over this protocol to fit in the ASCII range.
>
> An LSP sending a bad message is a serious bug that affects
> all clients of the LSP, and presumably the LSP operator has
> to restart the node in order to fix the bug.
> Thus, clients should not re-attempt sending any requests to
> the LSP until the client connects to it again, as the
> reconnection may signal that the LSP was restarted, which
> may signal that the LSP has had this serious bug fixed.

The LSP acts as a JSON-RPC 2.0 server, while the client acts
as a JSON-RPC 2.0 client.

> **Rationale** JSON is a simple format that is easy to
> extend and is extensively supported.
> The Lightning Network peer-to-peer transport protocol in
> BOLT8 is not inherently a request-response protocol like
> HTTP(S) or gRPC, and JSON-RPC 2.0 describes a
> JSON-based protocol that builds a request-response
> protocol from a simple tunnel; BOLT8 message ID 37913 acts as
> that tunnel.
> JSON-RPC 2.0 is simple to implement, and this specification
> describes an even simpler subset of JSON-RPC 2.0.
> Although BOLT8 messages are limited to payloads of 65533
> bytes, it is expected that both requests and responses
> would be far below that limit, and thus the ability to
> "cut" a large object across multiple messages, the
> use of a compression algorithm, and the use of a binary
> format instead of JSON, are all considered unnecessary.
> In particular, we expect most reasonable requests and
> responses to be less than 1000 bytes, and any compression
> would not significantly reduce the number of MTUs
> (generally about 1400 bytes per MTU) that lower network
> layers would need to transport.

> Moreover, the lack of compression greatly simplifies
> implementation, testing, interoperability, and dependencies.
> Compression is potentially vulnerable to [zip bomb][]s, a
> short piece of compressed data that expands to several
> gigabytes or terabytes of uncompressed data.
> We would need to impose *some* limit on the uncompressed
> text, and that limit might as well be the 65533-byte
> limit of BOLT8 message payloads.

[zip bomb]: https://en.wikipedia.org/wiki/Zip_bomb

The client:

* MUST send single complete JSON-RPC 2.0 request objects,
  UTF-8-encoded, as payloads for BOLT8 message ID 37913.
* MUST NOT send JSON-RPC 2.0 notification objects
  (i.e. every object it sends must have an `id` field).
* MUST NOT use by-position parameter structures, and MUST
  use by-name parameter structures.
* MUST NOT batch multiple JSON-RPC 2.0 request objects in
  an array as described in the ["Batch"](https://www.jsonrpc.org/specification#batch)
  section of the JSON-RPC 2.0 specification.

> **Rationale** By disallowing by-position parameter
> structures, other LSPS specifications need only to define
> parameter names and not some canonical order of parameters
> for by-position use.
> Having to handle only by-name parameter structures also
> simplifies the LSP code, as it does not have to check
> whether the `params` value is an array or a dictionary,
> and to separately map array elements to `params` keys.
> Batched requests require batched responses according to
> JSON-RPC 2.0, and it may be simpler for LSPs to handle
> unrelated LSPS methods separately without requiring
> re-batching of the responses; it gives a simple "one
> message is one request / response" rule.

The LSP:

* MUST send either of the below as payloads for BOLT8 message
  ID 37913:
  * a single complete JSON-RPC 2.0 response object, UTF-8-encoded.
  * a single complete JSON-RPC 2.0 notification object (i.e. an
    object without `id` but with `method`, `params`, and `jsonrpc`
    fields).
    * MUST use by-name parameter structures for notifications.
* MUST respect the [JSON-RPC 2.0 standard error codes][].
* SHOULD NOT send BOLT8 message ID 37913 unless the peer had
  already sent a BOLT8 message ID 37913, possibly in a past
  connection session.
* MAY send responses in an order different from the order in
  which the client sent the requests.

[JSON-RPC 2.0 standard error codes]: https://www.jsonrpc.org/specification#error_object

> **Rationale** The peer sending BOLT8 message ID 37913 is an
> indicator that it understands the LSPS0 protocol and wishes
> to act as a client.
> If the peer does not send it, then it is not an LSPS client
> and the LSP has no reason to send BOLT8 message ID 37913.
> Notifications allow the LSP to signal events to the client,
> provided the client has already previously signalled a
> willingness to receive such events by calling some
> LSPS-defined method to enable such events.
> For example, an LSPS might specify a method that enables
> the client to be signalled by an LSPS-specified notification,
> whenever the client could have received a payment but lacks
> the inbound liquidity for it.

#### LSPS Method And Notification Specifications

Other LSPS specifications:

* MUST indicate `method` names for each client-callable API
  endpoint and each LSP-initiated notification, as well as
  the key names of the `params` for each `method`, and the
  meanings of each parameter.
  * `method` names MUST be in snake_case, i.e. words are lower
    case and separated by `_` characters.
  * MUST prefix `method` names with `lsps` followed by the LSPS
    number followed by `.`, e.g.  `lsps999.do_this` for a client
    request or `lsps999.that_happened` for an LSP notification.
  * MUST also describe the possible error `code`s for each API
    endpoint, together with any `data` (which MUST be an object
    (dictionary)) for that error code, if there should be a
    `data` field.
    * MAY elect to not define a `data` object for an error
      `code`, in which case clients MUST NOT expect a `data`
      field.
  * MUST define `result` values for its API endpoints that
    are objects (dictionaries), and MUST define keys of the
    response.

> **Rationale** A prefix ensures that method names do not
> conflict across LSPS specifications, and creates a convention
> that allows non-standard extensions to define their own prefix.
> A `result` dictionary allows for later revisions of an
> LSPS specification to seamlessly add new keys in the response.

An LSP MAY return additional keys in the `response` values
that are not defined in the relevant LSPS specification.
Clients conversely MUST ignore unrecognized keys.

> **Rationale** This allows later revisions of an LSPS specification
> to seamlessly add new keys to the response while maintaining
> backwards compatibility with older clients that do not know the
> later revision with additional keys.
> Later revisions can make parameters backwards compatible by only
> adding optional new parameters, which when absent causes the
> API endpoint to behave identically to older revisions.

Other LSPS specifications MUST be designed to be resilient
against responses and notifications being lost on the way
from LSP to client.
The LSP may believe it has delivered the message, but the
IP packet containing the message may not have reached the
client before the client suffers some unexpected crash, or
the client may have been parsing and doing early processing
of the message before being able to persist the data related
to the response or notification.
Other LSPS specifications MUST:

* require that the LSP time-bound any information that the LSP
  has to keep before the LSP is compensated.
* require or support that the client store as much state as
  possible, and include some kind of signature or MAC that the
  LSP can use to recognize that it issued that state, without
  the LSP having to remember that state.
* describe level-triggered and not edge-triggered
  notifications (i.e.  notifications should mean "client, you
  still have some X you have to handle" and not "client, a
  new X was added / an X was removed / an X was changed").
* require that LSPs MUST send notifications whenever a change
  occurs in the items the client has to handle (send on edge),
  but also send notifications on connecting with the client
  if the level-trigger is still true on reconnection (i.e.
  also check the level on reconnection and re-send it).
* describe any queues (i.e. for items the client has to
  handle, such as HTLCs that cannot be delivered to the client
  via the normal BOLT messages yet, due to lack of a viable
  channel) as having separate "peek at first item" and "remove
  first item then peek at next item" APIs, while ensuring that
  each item in a queue has a unique identifier that the client
  can use to check if the item has already been read but the
  "remove first item" call for that item was not able to be
  delivered previously.

> **Rationale** Clients may crash due to operator mistakes
> or unrelated reasons, and a client may be an attacker and
> not a legitimate paying client.
> Thus, a client may initiate some flow or process that
> requires multiple communication rounds with the LSP, and
> then abort partway through due to a crash or a deliberate
> attempt to waste LSP resources.
> The network between the client and the LSP may also be
> unreliable, so notifications may be lost, so the state
> of whatever is being notified should be re-sent on
> reconnection.
> In particular, notifications are not ever explicitly
> acknowledged by the client on the transport layer
> level.
> Queues are particularly good for level-triggered
> notification, with the "level" being "is the queue
> not empty?".
> Processing of one item in this queue can take time on
> the client side, during which the client may crash or the
> connection interrupted, thus the first item should be
> retained on the LSP side, as the client may not have been
> able to persist it after it received the response from
> the "peek at first item" call.
> The client needs to be able to detect if it already
> completed processing of the queue top item, and combining
> the "remove first item then peek at next item" into a
> single call reduces the round trips needed to handle each
> completely-processed item while still retaining the item
> currently being processed on the LSP side in case of a
> client crash during processing.

#### Error Handling

In general, the client, the LSP, and any LSPS building on top of LSPS0 MUST respect
[JSON-RPC error codes][]. 
This document extends the error codes by describing edge cases
combining [JSON-RPC 2.0][] with [BOLT8][]. Any error code like `-32603 Internal error` is still valid
even though not mentioned explicitly.

[JSON-RPC error codes]: https://www.jsonrpc.org/specification#error_object

---

JSON-RPC 2.0 `error`s include a `message` field which is a
human-readable error message.

Clients MUST carefully filter error messages of possibly
problematic characters, such as NUL characters, `<`, newlines,
or ASCII control characters, which may cause problems when
displaying in a dynamic context such as an HTML webpage, a
TTY, or C string processing functions.

Clients SHOULD NOT directly expose error `message`s to users
unless users have enabled some kind of "advanced" mode or
"developer" mode.
Clients MAY write them in a log that is accessible to "advanced"
or "developer" mode users.

Clients SHOULD generate their own messages based on the
error `code`, for display to the user, possibly integrating
information in the optional `data` object if the error `code`
defines it.
If the client does not recognize the error `code`, or is
expecting a field in the `data` object that is not present, it
SHOULD indicate this as an "unrecognized" error to the user.

> **Rationale** LSPs might write incorrect or misleading
> human-readable error messages, and users might report such
> error messages as bugs to the client developers, since the
> user-visible source of the error message would be the client;
> it is thus better if the client writes its own error
> messages that it can change based on user feedback.
> Later revisions of some LSPS spec may introduce new error codes,
> or the specification may be incomplete and actual development
> shows that some unspecified error could possibly occur, in
> which case the human-readable `error` field could contain a
> description of this error, which developers of clients can
> then use to help guide the evolution of the specification, or
> to comply with later revisions of the LSPS spec.

##### `-32602` Invalid Parameters Error

An LSP that sends back an invalid parameter error with
`code = -32602` MUST include a `data` object in the `error`
response.

This `data` object MUST contain at least the field `unrecognized`,
a JSON array of strings, representing an unordered set of
parameter names.
These are the parameter names that are not recognized by the LSP as
valid for the RPC method.

For example, suppose the client sent this request:

```JSON
{
  "jsonrpc": "2.0",
  "method": "example.method_name",
  "params": {
    "future_feature1_param": "value1",
    "future_feature2_param": "value2"
  },
  "id": "42"
}
```

Suppose the LSP recognizes `example.method_name` as a valid
method, and recognizes `future_feature2_param` as a valid
parameter of that method, but does not recognize
`future_feature1_param`.
In that case, the LSP would respond with:

```JSON
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "unrecognized": ["future_feature1_param"]
    }
  },
  "id": "42"
}
```

> **Rationale** Suppose there exists some other LSPS.
> Suppose some future revision(s) of this LSPS includes (in
> whatever order) two additional parameters,
> `future_feature1_param` and `future_feature2_param`.
> If a client sends a request that includes
> `future_feature1_param` *and* `future_feature2_param`, and the
> LSP does not support one or both of them, the LSP would return
> an "Invalid params" -32602 error.
> However, without the `unrecognized` field in the `data` object,
> the client cannot know if the LSP does not support
> `future_feature1_param`, `future_feature2_param`, or both.

##### Custom Errors

[JSON-RPC 2.0][] protocol defines the range of `-31999 to +32767` (inclusive) to be application defined errors.
Each LSPS is provided and MUST use an error range of max 100 error codes. 
The range for each LSPS is calculated as follow: `LSPS-number * 100 to LSPS-number * 100 + 99` (inclusive).

For example:
- LSPS0: `00000 to 00099`
- LSPS1: `00100 to 00199`
- LSPS2: `00200 to 00299`

And so on until `+32699`. The range of `-31999 to -1` (inclusive) is undefined 
and MAY be used by applications outside of the LSPSpec. Such applications MAY request 
the spec group to register an error code range to avoid collision.

As per [JSON-RPC 2.0][], the range between `-32000 to -32099` is 
"reserved for implementation-defined server-errors". These error codes MAY be used by LSPs 
too. Clients MUST treat an error in this range similar to a `-32603 Internal error` if it does not know otherwise.


#### Disconnection Handling

Networks are unreliable, and network participants are also unreliable.

Clients MUST use a high-entropy `id` string for JSON-RPC 2.0 requests,
such as a UUID, or a hex encoding of a random binary blob of at least
80 bits, with randomness acquired from a cryptographically secure
source.
Clients MUST NOT use just a simple incrementing counter for the `id`.

If a client receives a JSON-RPC 2.0 response with an `id` it does
not remember sending a request for, it MUST ignore that response.

> **Rationale** Clients, LSPs, and the network between them
> can individually be unreliable, leading to clients that forget
> `id`s they issued previously, or clients or LSPs thinking that
> the other side may have restarted when it is the network
> between them that failed.
> Thus, random `id`s picked from a large space are the safest
> when the client might lose any counter state (by crashing if
> using an in-memory counter, or loss of persistent storage on
> hardware without storage redundancy), and are resilient
> against reconnections when both sides remained running.

Clients MAY include additional information in their `id` for
internal tracking, as long as the total `id` has sufficient
entropy for universal uniqueness.

Clients SHOULD internally impose a reasonable timeout, on
the scale of minutes, for receiving a response for a request,
and SHOULD treat a timeout event as a temporary server failure,
and forget the `id` of the timed-out request.

> **Rationale** The LSP might crash between the time the client
> makes the request to the time it could complete processing of
> the response, and thus lost track of the request.

If the LSP is unable to deliver a response to the client due
to a disconnection, it SHOULD treat this no differently from
successfully delivering the response but the client then does
not "follow up" on any action after that.

> **Rationale** Even if the LSP delivers the response to the
> client, the client could then crash while processing the
> response, which means that the LSP cannot be sure that any
> response is received by the client anyway.

### Lightning Feature Bit

The [BOLT7][] specification describes the `node_announcement`
gossip message, which includes a `features` bit field.
The [BOLT1][] specification describes the `init` message,
which also includes a `features` bit field.

[BOLT7]: https://github.com/lightning/bolts/blob/f7dcc32694b8cd4f3a1768b904f58cb177168f29/07-routing-gossip.md
[BOLT1]: https://github.com/lightning/bolts/blob/f7dcc32694b8cd4f3a1768b904f58cb177168f29/01-messaging.md

LSPs MAY set the `features` bit numbered 729
(`option_supports_lsps`) in both the `init` message on connection
establishment, and in their own advertised `node_announcement`.
Clients MUST NOT set `features` bit numbered 729 in either
context.

> **Rationale** Because all communication is initiated 
> through clients sending BOLT-8 messages
> only servers need to advertise themselves.
> Servers can choose to advertise themselves using feature
> bit 729. Clients can discover LSP's by downloading
> gossip and inspecting the channel-graph.
> The bit 729 was chosen randomly and has no special meaning.

LSPs MAY set the `features` bit 729 `option_supports_lsps` if it
supports at least LSPS0, and MAY set the `features` bit even if it
does not support some of the LSPS specifications.

> **Rationale** This specification also describes a `lsps0.list_protocols`
> API which the LSP uses to report exactly which LSPS specifications
> it supports.

### LSPS Specification Support Query

The client can determine if an LSP supports a particular LSPS
specification other than LSPS0 via the `method` named
`lsps0.list_protocols`, which accepts no parameters `{}`.

`lsps0.list_protocols` has no errors defined.

The response datum is an object like the below:

```JSON
{
  "protocols": [1, 3]
}
```

`protocols` is an array of numbers, indicating the LSPS specification
number for the LSPS specification the LSP supports.
LSPs do not advertise LSPS0 support and 0 MUST NOT appear in the
`protocols` array.

> **Rationale** LSPS0 support is advertised via `features` bit 729
> already, so specifying `0` here is redundant.

> **Non-normative** The example below would not be necessary for other
> LSPS specifications, but gives an idea of how the JSON-RPC 2.0
> protocol would look like, when using this API endpoint.

As a concrete example, a client might send the JSON object below
inside a BOLT8 message ID 37913 in order to query what LSPS protocols
the LSP supports:

```JSON
{
  "method": "lsps0.list_protocols",
  "jsonrpc": "2.0",
  "id": "example#3cad6a54d302edba4c9ade2f7ffac098",
  "params": {}
}
```

The LSP could then respond with a BOLT8 message ID 37913 with the following
payload, indicating it supports LSPS1 and LSPS3 (in addition to LSPS0):

```JSON
{
  "jsonrpc": "2.0",
  "id": "example#3cad6a54d302edba4c9ade2f7ffac098",
  "result": {
    "protocols": [1, 3],
    "example-undefined-key-that-clients-should-ignore": true
  }
}
```

### LSPS Extension And Versioning

Individual LSPS SHOULD NOT include their own explicit versioning
scheme.

An individual LSPS MAY be revised to a later revision.
Later revisions of an individual LSPS MUST NOT break compatibility with
earlier revisions.

If there is a need for a change that breaks compatibility with an
existing revision of an LSPS, then a new LSPS with a different
 LSPS number MUST be used.

Later revisions of an LSPS MAY, if a new need arises:

* Add new optional `params` fields for a client-callable API `method`.
  * If the optional parameter is not specified, the `method` MUST
    act the same as in previous revision of the LSPS, before the
    parameter existed.
* Add new required or optional `result` fields for a client-callable
  API `method`.
  * Provided that the client is free to ignore the added fields and
    that acceptance of a new `result` field is signalled by using a
    new optional `params` field of some client-callable API `method`,
    or by calling a new client-callable API `method`, or by some other
    method, and that a lack of signal of acceptance of the new field
    results in behaving the same as in older revisions.
    (**Rationale** Older clients WILL ignore unknown fields and thus
    would not be aware of them)
  * Allowed recursively for any field that itself contains a
    dictionary (i.e. a `result` field may contain a dictionary, and a
    new revision of an LSPS may define a new field for that dictionary)
    to any depth of dictionary fields, provided the previous provision
    is followed.
* Add new error `code`s for a client-callable API `method`.
* Add completely new client-callable API `method`s.
* Add completely new LSP-initiated notification `method`s, provided
  they are enabled only by a new optional field in some
  client-callable API `method` or by a completely new client-callable
  API `method`.

Later revisions of an LSPS MUST NOT make changes beyond the above.

A client:

* MUST ignore unrecognized `result` fields from the result of a
  client-callable API `method`.
  * MUST recursively ignore unrecognized fields from any
    dictionaries in a recognized `result` field, to any depth.
* MUST treat as a generic error any unrecognized error `code` from
  an `error` result for a client-callable API `method`.
  * SHOULD log this as unusual.
* MUST ignore unrecognized LSP-initiated notification `method`s.
  * SHOULD log this as unusual.

An LSP MUST:

* Respond with a -32602 "Invalid params" error, described in a
  previous section with `unrecognized` field in the `data`
  dictionary, if it receives a client-callable API `method` it
  recognized, but with an unrecognized parameter.
* Respond with a -32601 "Method not found" error, if it receives a
  request to call a client-callable API `method` that it does not
  recognize.

If the client supports an older revision of an LSPS, it simply does
not provide any new optional parameters to existing client-callable
API `method`s, or any new client-callable API `method`s.
Then the interface would act the same as in the older revision.

If a client wants to use a newer revision of an LSPS, it can look
for some new `result` field, and if that does not exist, it knows
the LSP supports an older revision of the LSPS.
The client can provide any new optional parameters it wants to
use, and if the LSP responds with a -32602 "Invalid params" error,
that error includes an `unrecognized` field, described above, that
contains the parameters that the LSP does not supoort.
The client can use a new client-callable API `method`, and if the
LSP responds with a -32601 "Method not found" error, knows that the
LSP does not support that `method`.

Vendor-specific extensions to an LSPS can also obey the above
rules, and would remain compatible with a non-vendor LSP and a
non-vendor client.

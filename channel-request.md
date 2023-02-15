# LSP channel request

## Version: 0.0.2

## Access to the API

This interface is accessed via HTTP, using either of the below protocols:

* HTTPS with a DNS-resolved hostname.
* HTTP with a TorV3 onion.

LSP services MUST support all widely-deployed HTTP versions.
As of this writing, HTTP/1.1 and HTTP/2 are widely-deployed.

An LSP channel request API access point MUST be relative to a given URI.
For example, if an LSP API base URI is given as:

    https://example.org/~zmnscpxj

Then to access the `lsp/channel` API, the URI to be accessed will be:

    https://example.org/~zmnscpxj/lsp/channel

If the LSP API is given as:

    http://t7pqxhwpcjjm75vlnq3fbqxthsfr2ceb6oqlr322xgzlmp5ciyjlh3ad.onion

Then to access the `lsp/channel` API, the URI to be accessed will be:

    http://t7pqxhwpcjjm75vlnq3fbqxthsfr2ceb6oqlr322xgzlmp5ciyjlh3ad.onion/lsp/channel

The base URI MAY specify a port number; if unspecified, the standard
for HTTPS or HTTP is assumed.

The base URI MUST NOT be HTTPS if the domain is TorV3 onion.
The base URI MUST NOT be HTTP if the domain is not a TorV3 onion

For HTTP `POST` accesses:

* Both request parameters and response parameters are in a JSON object
  in UTF-8 encoding (MIME type `application/json; charset=utf-8`) as
  per [RFC 7158](https://www.rfc-editor.org/rfc/rfc7158).
  * The JSON object will then be the request body or response body,
    respectively.
* The client MUST ignore the HTTP response code returned by the server,
  unless it is a redirection code.
* The server MUST disallow caching of the response.
* The server MUST ignore any HTTP cookies and MUST NOT require any
  authentication.

For HTTP `GET` accesses:

* Request parameters are given in the `GET` query parameters.
* The response parameters are in a JSON object in UTF-8 encoding, as
  above.
* The server MUST disallow caching of the response.
* The server MUST ignore any HTTP cookies and MUST NOT require any
  authentication.

For example, when passing `0123456789abcdef` as the `id` parameter to the API
`GET` access point `lsp/channel` on server `https://example.org/~zmnscpxj`,
the client may send:

    GET /~zmnscpxj/lsp/channel?id=0123456789abcdef HTTP/1.1
    Hostname: example.org

And the server may respond:

    HTTP/1.1 200 OK
    Content-Type: application/json
    Cache-Control: no-cache
    Content-Length: 64

    { "order_id": "0123456789abcdef", "state": "UNKNOWN_OR_UNPAID" }

All other HTTP methods are left unspecified.

### Rationale

HTTP is a well-defined interface that is commonly implemented and
well-supported in the industry, even outside of Lightning or Bitcoin.

HTTP/2 and HTTP/1.1 are widespread enough that their implementation is
expected to be easily available.

Caching is disabled as users are expected to query the interface for the
latest status of their channel request.

Other HTTP methods are left unspecified in case of future extensions of
this protocol.

## API Definitions

### HTTP `POST` to `lsp/channel`

#### POST
##### Summary

Request an inbound channel.

##### Description

Request an inbound channel with a specific size and duration.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ------ |
| node_connection_info | body | pubkey or pubkey@host:port | Yes | string |
| remote_balance | body | Inbound liquidity amount in satoshis | Yes | integer |
| local_balance | body | Outbound liqudity amount in satoshis | No | integer |
| on_chain_fee_rate | body | On-chain fee rate of the channel opening transaction in satoshis per vbyte | No | number |
| channel_expiry | body | Channel expiration in weeks | No | integer |
| options | body | What options the client wants | No | array of strings |

For example:

    GET /~zmnscpxj/lsp/channel HTTP/1.1
    Hostname: example.org
    Content-Length: 125
 
    { "node_connection_info": "03456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123"
    , "remote_balance": 1000000
    }

`remote_balance` is the inbound liquidity being purchased by the client.
This MUST be positive non-0.

`local_balance` is how much the server will `push_msat` into the channel.
The client will pay this amount to the server as part of the fee opening.
This MUST be positive or 0.
If unspecified, it is treated as 0.

`on_chain_fee_rate` is the minimum the client will consider as the
channel opening fee rate.
If unspecified, the server may set any onchain fee rate.

###### `options` Specification

The `options` field encodes a set of options, indicated as strings.
If unspecified, it is considered as the empty set.

As a set, duplicate options are considered as a single option.

The following options are allowed:

* `"require-0-conf-open"` - Tells the server that the client will
  respond with `minimum_depth` of 0 in the `accept_channel` message,
  and that the server MUST emit `funding_locked` / `channel_ready`
  as soon as the opening transaction is broadcast.

##### Response

In case of success:

| Name | Description | Required | Schema |
| ---- | ----------- | -------- | ------ |
| error | Whether this is an error | No | `false` |
| order_total | The total fee plus the `local_balance` requested | Yes | number |
| fee_total | The total fee the lsp will charge to open this channel | Yes | number |
| fee_per_payment | For intercepted payment of fee, fee taken from each payment until fee is paid | No | number |
| temporary_scid | The scid user puts in the route hint of invoice to identify order | No | string |
| lsp_connection_info | pubkey@host:port | Yes | string |
| ln_invoice | A lightning bolt11 invoice to pay the fee for this channel open | No | string |
| btc_address | An on-chain bitcoin address to pay the fee for this channel open | No | string |
| btc_address_confirms | The number of confirmations before btc_address is considered as funded | No | integer |
| order_id | An lsp generated order id used to look-up the status of this request | Yes | string |

`error` may be unspecified, in which case, the client assumes that the
request suceeded.
If it is specified, it should be `false` to indicate that no error
occurred and the server accepted the request.

`order_total` MUST equal the sum of `fee_total` and the `local_balance`
from the request.

`lsp_connection_info` must explicitly be a full node ID with a `host:port`,
to allow the client to connect to the LSP.

The server MAY provide one of several nodes that it controls as the LSP
the client MUST connect to, in order to be serviced, i.e. the same base
URI may return one of several nodes on the network, if one single entity
controls multiple nodes.

`ln_invoice` is a BOLT11 invoice.
The `ln_invoice` MUST give the same amount as in `order_total`.

The client:

* MUST validate that the amount required by `ln_invoice` is the same
  as the amount given in `order_total`.
* MUST validate that `order_total` equals the sum of `local_balance`
  and `fee_total`.
* SHOULD check whether `fee_total` is reasonable.

`btc_address` MAY be a bech32 version 0 ("SegWit") or bech32m version 1
("Taproot") address.
The server SHOULD NOT provide any other address type.
The client MAY support other address types.

`btc_address_confirms` is the number of confirmations that the server
will accept before considering the order as paid.
If 0, this indicates that the server accepts 0-confirmation payments to
this address.
If unspecified, it defaults to 3.
It MUST be positive or 0.

The server:

* MUST provide at least one of `ln_invoice` and `btc_address`.
  * MAY provide *both* `ln_invoice` and `btc_address`.

###### `order_id` Specifications

The `order_id` MUST:

* not be treated as a number of any kind, i.e. `"01"` must be distinct from `"1"`.
* Be composed of the following ASCII characters ONLY:
  * `0` - `9`
  * `a` - `z`
  * `A` - `Z`
  * `+`
  * `/`
  * `-` minus
  * `_` underscore
  * `=`
* not be longer than 128 characters.

Servers:

* MUST NOT use a simple incrementing numeric counter, such as an `AUTOINCREMENT`
  database row ID.
  * SHOULD use an encrypt-then-MAC or AEAD construction to wrap such a numeric
    counter in an unforgeable manner, then encode in hex or any Base64 variant
    with compatible character set.
* MAY use a large nonce of at least 80 bits entropy as order ID.
  * MAY use the payment hash of the issued invoice.

Clients:

* MUST encode `+` as `%2B`, `/` as `%2F`, `-` as `%2D`, `_` as `%5F`, and
  `=` as `%3D`, when encoded in the `GET` interface `id` parameter.
* MUST treat the `order_id` as case-sensitive, i.e. `"A"` is distinct from
  `"a"`.

###### `order_id` Rationale

A simple incrementing numeric counter would allow third parties to query the
`GET` interface with arbitrary order IDs, which would allow them to snoop on
other clients of the LSP.
This restriction prevents LSPs from inadvertently leaking the privacy of their
clients;
LSPs can still publish or otherwise leak this information by other means, but
LSPs that honour the privacy of clients will at least not leak it by naive
implementation of this specification.

The set of allowed characters allows for large byte sequences from
cryptographic-quality entropy to be encoded in hexadecimal or some variant
of Base62 or Base64.

128 characters can encode 64 bytes when using hex encoding.
For reference, an AEAD encoding can have 24 bytes for nonce and 32 bytes
for tag, and a simple numeric counter can fit in 8 bytes, for a total of
64 bytes.
When using Base64 encoding, 128 characters can encode 96 bytes.

A large nonce allows an LSP to provide a stateless invoice.
For instance, the order ID might be the payment hash of the invoice, and
the order details may be encoded in the `m` field of the BOLT11 invoice,
with the order never being stored on the server until it is actually
paid.

`+` and `=` have special meaning in the `GET` parameters, and must be
suitably escaped; characters other than the letters and numbers should
also be similarly escaped for consistency.

##### Channel Opening Specifications

The client:

* MUST pay `order_total` satoshis to the `ln_invoice` or `btc_address`
  given.
* MUST connect to the returned `lsp_connection_info`.

If the server returned a `btc_address`, it MUST monitor the state of
the address automatically without further queries or requests from the
client.

The server:

* MUST open the channel to the client, when:
  * The `order_total`, or more, has been paid to the given `ln_invoice`
    or `btc_address`.
  * The client is connected to the node indicated in
    `lsp_connection_info`.
* MAY make multiple open requests: as long as one of them commits, the
  order is considered fulfilled.
* if the `"require-0-conf-open"` was specified, MUST emit `funding_locked` /
  `channel_ready` as soon as the opening transaction is broadcast.

On channel opening, the server MUST:

* Provide a total balance that is greater than or equal to the sum of
  `remote_balance` and `local_balance`.
* Provide a `push_msat` parameter that is greater than or equal to the
  `local_balance`.
* Set the channel opening transaction fee rate as greater
  than or equal to the given `on_chain_fee_rate`, if specified.

###### Rationale

The server is allowed to overprovision the channel it opens with the
client.

For instance, the server may need to "bin" balances.
In that case, it may round up the requested `remote_balance` and
`local_balance` of the client.

The server may wish to open channels in batches, i.e. opening
multiple channels at once.
In that case, it would use the largest requested `on_chain_fee_rate`
for the batched transaction.

The server may implement RBF for channel opening.
In that case, the client will see multiple parallel channel open
requests, each one being a different version of the channel
opening transaction.

##### Error

In case of error, the server response schema is:

| Name | Description | Required | Schema |
| ---- | ----------- | -------- | ------ |
| error | Whether this is an error | Yes | `true` |
| type | What exactly the error is | Yes | string |
| detail | Error details | Yes | dependent on `type` |

In case of an error, the server returns an error object, with a
field `error` set to `true`, indicating that an error occurred
in processing the request.

The error `type` defines what the `detail` is.
The valid `error` types are:

* `"unsupported-options"` - one or more of the `options` specified
  is not supported by the server.
  The `detail` is an array of strings, containing the options that
  were not supported by the server.
* `"local_balance-out-of-bounds"` - the given `local_balance` is
  too small or too large for the server.
  The `detail` is a two-length array of integers, `[0]` being the
  low boundi, inclusive, `[1]` being the upper bound, inclusive.
* `"remote_balance-out-of-bounds"` - the given `remote_balance`
  is too small or too large for the server.
  The `detail` is a two-length array of integers, `[0]` being the
  low boundi, inclusive, `[1]` being the upper bound, inclusive.
* `"total_balance-out-of-bounds"` - the sum of the given
  `local_balance` and `remote_balance` is too small or too large
  for the server.
  The `detail` is a two-length array of integers, `[0]` being the
  low boundi, inclusive, `[1]` being the upper bound, inclusive.
* `"on_chain_fee_rate-out-of-bounds"` - the given `on_chain_fee_rate`
  is too small or too large for the server.
  The `detail` is a two-length array of integers, `[0]` being the
  low boundi, inclusive, `[1]` being the upper bound, inclusive.
* `"channel_expiry-out-of-bounds"` - the given `channel_expiry`
  is too small or too large for the server.
  The `detail` is a two-length array of integers, `[0]` being the
  low boundi, inclusive, `[1]` being the upper bound, inclusive.

The client:

* MUST report to the user that the error is unrecognized if it is
  not one of the `type`s it recognizes, and MUST NOT show any of
  the `type` or `detail` fields to the user. 

###### Rationale

There is no human-readable error message; the expectation is
that the client will synthesize its own human-readable error
message from the error `type` and `detail` to report the
error to the user.

Human-readable error messages may:

* Embed encodings that are problematic in some display contexts,
  such as unsafe HTML or invalid unicode or unicode that changes
  the printing context (e.g. left-to-right marker).
* Give misleading information to users (i.e. wetware hacking)
  that may lead naive users to behave in ways that could lose
  their funds or privacy.

Thus, there is no provision for human-readable error messages.

For the same reason, the client must not show the `type` or
`detail` fields if it does not recognize the `type`, as this
can be used to sneak in a human-readable error message, with
the same issues as above.

### /lsp/channel

#### GET
##### Summary

Get information about a channel order

##### Description

Get information about a channel order

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| id | query | order_id provided in response to channel request or scid | Yes | string |

##### Response

| Name | Description | Schema |
| ---- | ----------- | ------ |
| order_id | The order id | string |
| created_at | Number of seconds since epoch when this order was created | number |
| local_balance | Local balance in sats requested by client | number |
| remote_balance | Remote balance in sats requested by client | number |
| channel_expiry | Channel expiry in weeks requested by client | number |
| channel_expiry_ts | Number of seconds since epoch when the channel can be closed | number |
| order_expiry_ts | Number of seconds since epoch when this order can still be paid | number |
| order_total | The total fee plus the local_balance requested | number |
| fee_total | The total fee the lsp will charge to open this channel | number |
| fee_per_payment | For intercepted payment of fee, fee taken from each payment until fee is paid | number |
| temporary_scid | The scid user puts in the route hint of invoice to identify order | string |
| scid | The scid of the channel if already established | string |
| lsp_connection_info | pubkey or pubkey@host:port of the lsp node | string |
| ln_invoice | A lightning bolt11 invoice to pay the fee for this channel open | string |
| btc_address | An on-chain bitcoin address to pay the fee for this channel open | string | 
| lnurl_channel | A way to request the open via lnurl after the order is paid | string |
| amount_paid | Amount paid by client so far in sats | number |
| node_connection_info | The node_connection_info for the node to open the channel to | string |
| channel_open_tx | The txid of the channel funding tx once it is broadcast | string |
| state | The state of the order | string |
| onchain_payments | A list of payments received to btc_address on-chain | object[] |

At minimum, the response MUST contain `state` field, and depending on state, other
fields may or may not exist.

###### Order `state` enum

| State          	| Description                                                                           	|
|----------------	|---------------------------------------------------------------------------------------	|
| "UNKNOWN_OR_UNPAID"     | The order does not exist, or the user has a valid invoice for it but has not paid yet, or order has expired.|
| "UNDER_FUNDED"      | Funding received but under paid.                                                       	|
| "PENDING"        	| Order is paid but the channel has not been opened yet.                                	|
| "OPENING"       	| The opening transaction has been broadcasted. 0conf might skip directly to OPENED.     	|
| "OPENED"         	| Channel is open and has the necessary block confirmations.                            	|

Order states are encoded in JSON as strings in all capitals.

###### `state` Rationale

The ``UNKNOWN_OR_UNPAID` state allows for a server to create a valid channel order
without requiring the server to store this data.
For instance, the server may issue a valid LN invoice with an `m` field
containing server-validatable information about the order.
This allows the server to avoid a DoS vector, where an attacker uses a botnet to
make multiple requests to the `POST` API for new channels without ever paying the
server, and thus overburdening the server permanent storage with useless entries.
For this reason, a non-existent order ID, an unpaid order, and an expired order
are all merged into a single state.

`UNDER_FUNDED` may occur if the server has responded with `btc_address`, where
the client has complete control over how much funds will be paid into the onchain
address.
Lightning invoices should be rejected by the server if insufficient funds are
offerred.

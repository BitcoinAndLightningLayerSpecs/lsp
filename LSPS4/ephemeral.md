# LSPS4 Continuous JIT Channel --- Ephemeral SCIDs

## Motivation

BOLT11 invoices are, as of the start of the LSPS project, already
widely deployed and supported by many Lightning Network wallets
and applications.

BOLT11 invoices need to indicate:

* Some destination payee node ID.
* Optionally, some path, in cleartext, from a published node to
  the destination.

While a node that has no published channels can replace its node
ID with a one-time-use node ID, it would then need to indicate a
path from a published node to itself.
The path includes an SCID, which, if it is a stable, unchanging
SCID, can be used to correlate different invoices as actually
going to the same destination, even if an unpublished node uses
different node IDs fpr each invoice.

In the case of an LSP-client, the LSP would need to first agree
that *some* SCID maps to a particular client.
An LSP would also need to support reserving multiple SCIDs, each
one being single-use.

### Privacy Model

This extension cannot work around the "[Axiom of Terminus][]", i.e.
an unpublished channel leaks the fact that a Lightning Network node
after an unpublished channel and a published node, is the final
destination of the payment.

[Axiom of Terminus]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-October/002241.html

LSPS4 Continuous JIT Channels require 0-conf channels, and 0-conf
channels need to be unpublished channels, thus channels between an
LSP and a client using LSPS4 are always unpublished.

Because of the [Axiom of Terminus][], an LSP will always be able
to log every payment to the client, regardless of the use or
non-use of this extension.

Thus, this extension only provides the client privacy against
anyone it issues BOLT11 invoices to (i.e. payers).
It does not provide privacy against the LSP, and the LSP can
always leak payment details of its clients.

## Flow

The flow of using ephemeral SCIDs is largely the same as that of
the normal LSPS4 flow, with the following differences:

* On version negotiation, the client checks if the LSP supports
  `ephemeral_scids`.
* Whenever the client wants to issue an invoice, it calls the LSP
  with `lsps4.reserve_scid`, adding an extension parameter
  `ephemeral` set to `true`.
  * The client then generates a fresh ephemeral node ID, and
    issues the invoice using the ephemeral node ID and the ephemeral
    SCID.
* When the LSP notifies a new PPP `lsps4.you_have_new_ppp`, from a
  payment part that indicates an ephemeral SCID as the next hop,
  the PPP `details` includes a `via_scid` field, which is the
  ephemeral SCID indicated in the LSP-inbound payment part.
* When the LSP forwards a payment part, the forwarding message
  includes a `via_scid` TLV extension, which is the ephemeral SCID
  indicated in the LSP-inbound payment part.
  * The LSP adds this whether the payment part was forwarded using
    `lsps4.forward_ppps` or `lsps4.open_with_ppps`.

The sections below are numbered according to the base LSPS4 section
that it modifies.

### 0. API Version Negotiation

A client MUST check for the presence of this extension, by checking
for the existence of a key `ephemeral_scids` in the result of
`lsps4.select_version`.
The LSP supports this extension if the key exists and the value is
the JSON Boolean `true`.

```JSON
{
  "api_version": 1,
  "ephemeral_scids": true,
  "some_other_extension_the_client_does_not_recognize": true
}
```

### 1. Request SCID

Whenever a client wants to issue a privacy-preserving invoice, the
client MUST call `lsps4.reserve_scid` with an additional parameter,
`ephemeral`, set to the JSON Boolean `true`:

```JSON
{
  "expiry_time": "2023-04-20T10:08:41.222Z",
  "min_final_cltv_expiry_delta": 18,
  "ephemeral": true
}
```

`ephemeral` is a JSON Boolean value, which must be `true` if the
parmater is present.
IF the `ephemeral` parameter is absent, then the client is not
using this extension and the LSP behaves the same as in the base
protocol.

When the `ephemeral` flag is indicated, the LSP MUST issue an
epheemeral SCID.

The ephemeral SCID differs from the normal LSPS4 SCID in the
following ways:

* The `expiry_time` is always strictly followed; after the
  `expiry_time`, the LSP MUST forget the ephemeral SCID even if
  the client and the LSP have a channel in "normal operation".
* Once the client has claimed any payment part sent using the
  ephemeral SCID, the LSP SHOULD forget the ephemeral SCID.
* The client MAY reserve multiple ephemeral SCIDs.
  Every time the `lsps4.reserve_scid` API is called with the
  `ephemeral` flag, a new SCID is reserved for the client.

The result schema is the same as without the `ephemeral` flag:

```JSON
{
  "lsps4_scid": "8359321x85942x65443",
  "expiry_time": "2023-04-19T11:02:11.841Z",
  "min_final_cltv_expiry_delta": 24,
  "ln_base_fee_msat": "1000",
  "ln_prop_fee": 1000,
  "cltv_expiry_delta": 144
}
```

`lsps4_scid` is the ephemeral SCID reserved [<LSPS0.scid>][].
The LSP MUST ensure that the SCID is unique at the LSP, that is, no
other peer (client or non-client) of the LSP has the same SCID allocated
to it, whether or not the SCID could refer to an actual possible channel
outpoint.
The returned ephemeral SCID MUST be different from the normal LSPS4
SCID that the LSP would return for a non-`ephemeral` call to
`lsps4.reserve_scid`.

`expiry_time` is the expiry time for the reserved SCID
[<LSPS0.datetime>][].
For an `ephemeral` SCID, it is the same as indicated in the parameter.

The ephemeral SCID fees and CLTV expiry delta cannot be changed for
the lifetime of the ephemeral SCID.

After getting an ephemerl SCID, the client then creates a fresh ephemeral
node ID, and uses the ephemeral SCID and the ephemeral node ID in the
BOLT11 invoice it issues.

The client MAY generate the ephemeral node ID by derivation from the
private key of its normal node ID, and the ephemeral SCID returned by
this call.
**Non-normative** For example, it can get the HMAC-SHA256 of some
serialization of the SCID, using the private key of its node ID as the
HMAC secret key, and use the resulting HMAC as the private key for the
ephemeral node ID.

> **Rationale** Using the ephemeral SCID to generate the ephemeral node
> ID allows the client to issue private stateless invoices, i.e. invoices
> where the information of the payment is stored in the `payment_secret`
> and/or the `metadata` fields of the invoice.
> Both of those fields are in the forwarding onion received by the
> payment destination, and in order to decryption the onion, the payment
> destination needs to know the node ID.
> In the case of a private invoice, the ephemeral node ID needs to be
> known by the destination node (client in this context).
> By deriving from its normal node ID private key, and the ephemeral
> SCID, the client can re-derive the encryption key needed to read the
> onion, thus allowing for stateless private invoices.
>
> An LSP can mess with this by issuing the same SCID for multiple
> requests for an ephemeral SCID.
> However, the client already has to trust LSP for privacy as noted
> above.

### 3. Pending Payment Part Notification

When the LSP receives a payment part that indicates the next
hop with an ephemeral SCID reserved to a specific client, the
PPP of the LSP MUST include the `via_scid` field in the `details`
object of the PPP.

When the LSP sends the `lsps4.you_have_new_ppp` notification,
the `details` object has the required key:

* `via_scid` - the ephemeral SCID that was indicated as the next
  hop for this payment part [<LSPS0.scid>][].
  * For `"htlc"`-type PPPs, this is the next hop indicated in
    the type 6 (`short_channel_id`) TLV format of the
    [BOLT 4 payload format][] that is decoded at the LSP.

[BOLT 4 payload format]: https://github.com/lightning/bolts/blob/803a532c49be2f152c7f2dbaa0ec7d4c23a6013d/04-onion-routing.md#payload-format

```JSON
{
  "ppp_id": "id#0123456789abcdef",
  "details": {
    "type": "htlc",
    "amount_msat": "21000",
    "via_scid": "8359321x85942x65443",
    "hash": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "cltv": 599999
    "onion": "abcdefghijklmnopqrstuvwxyz+/"
  }
}
```

If the client derived the ephemeral node ID using the ephemeral
SCID, then the client can use the `via_scid` field to re-derive
the same ephemeral node ID, which it can then use to decrypt the
incoming onion.

### 4. Attempting Normal Forwarding

When the LSP forwards an HTLC-type PPP to the client, because of a
`lsps4.forward_ppps` call, if that HTLC-type PPP indicated an
ephemeral SCID, then the LSP MUST include a `via_scid` TLV in the
`update_add_htlc` message to the client.

In BOLT terms, the `update_add_htlc` message includes the following
TLV:

* types:
  1.  type: 65535 (`via_scid`)
  2.  data:
      * [`short_channel_id` : `short_channel_id`]

The `short_channel_id` is the binary format for SCIDs, an 8-byte
format defined in [BOLT 7 Definition of `short_channel_id`][]:

* The SCID `BBBxTTTxOOO` is split into block index `BBB`,
  transaction index `TTT`, and output index `OOO`.
* The first three bytes are the big-endian block index.
* The next three bytes are the big-endian transaction index.
* The last two bytes are the little-endian output index.

[BOLT 7 Definition of `short_channel_id`]: https://github.com/lightning/bolts/blob/803a532c49be2f152c7f2dbaa0ec7d4c23a6013d/07-routing-gossip.md#definition-of-short_channel_id

> **Rationale** The `via_scid` TLV is odd, following the BOLT
> "it's OK to be odd" rule.
> Clients that do not derive the ephemeral node ID from the ephemeral
> SCID can ignore the existence of this TLV.

### 7. Opening Channel And Forwarding Selected Pending Payment Parts

After opening the channel to the client, the LSP forwards any payment
parts in the PPP Opening Set to the client.

For HTLC-type PPPs where the next hop indicated an ephemeral SCID,
the `update_add_htlc` message includes `via_scid` (65535), described
above.
This is in addition to any `extra_fee` (65536) TLV that the message
may have.

[<LSPS0.scid>]: ../LSPS0/common-schemas.md#link-lsps0scid
[<LSPS0.datetime>]: ../LSPS0/common-schemas.md#link-lsps0datetime

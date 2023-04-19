# LSPS0 Common Schemas

## Motivation

This document describes how particular Lightning and LSPS-specific types
are encoded into a JSON format, so that other LSPS specifications need
only refer to this document without having to repeat this over and over
again.

Here are a few facts about JSON:

* JSON numbers are technically supposed to be [IEEE 754][] Double
  Precision floating-point numbers.
  These numbers have 53-bit mantissas; if a number would require
  more than 53 bits of significant binary digits to represent, then
  IEEE Double Precision numbers will lose accuracy.
  * Even so, as a text representation, there may be numbers that can
    be accurately represented in [IEEE 754][] but which will require a
    very long decimal string representation, which common JSON printers
    are likely to truncate.
  * Some short decimal string representations may not be accurately
    represented as [IEEE 754][] Double Precision numbers, too.
* Many JSON parsers will separate floating-point and integer numbers,
  based on whether a `.` character exists in a number.
  Most of these will use signed 32-bit integers for parsed integral
  numbers.

[IEEE 754]: https://ieeexplore.ieee.org/document/30711

## Common Schemas

When a schema specifies a JSON string format, characters that can be
embedded into a JSON string without escapes MUST be encoded directly.
For example, even though the JSON strings `"A"` and `"\u0041"` are
equivalent encodings of the same string, only `"A"` would be allowed
under this specification.

> **Rationale** Using alternative ways of expressing the same
> characters will both increase the size of the string *and* make
> the string less readable to humans, to no advantage.

### Monetary Amounts

###### Link: LSPS0.sat
###### Link: LSPS0.msat

Monetary amounts MUST be expressed in either millisatoshi or satoshi
units.

Other LSPS specifications MUST add a suffix to object field keys whose
value is a monetary amount:

* `_msat` for monetary amounts in millisatoshi units.
* `_sat` for monetary amounts in satoshi units.

> **Rationale** In some contexts, such as on-chain amounts like
> channel capacities, it is impossible to use sub-satoshi amounts,
> so using millisatoshi units for those is pointless.
> On the other hand, in many contexts Lightning uses millisatoshi
> amounts.
> An explicit suffix in field names helps ensure that developers
> do not confuse the two units.
>
> Using larger units may require writing numbers out as a decimal
> string representation with `.`, which might lose accuracy during
> conversion from text to an IEEE 754 number or vice-versa.

Monetary amounts MUST be encoded as JSON strings containing the
decimal text representation of the number of millisatoshis or
satoshis.
LSPS implementations SHOULD internally use an unsigned 64-bit
number to represent amounts.

> **Rationale** The maximum number of millisatoshis on the Bitcoin
> blockchain would require 63 significant bits.
> A JSON integral number might be parsed into only a 32-bit
> representation, or an actual IEEE 754 floating-point number
> with only 53 bits of mantissa.

For example, the Bitcoin dust limit of 546 satoshis would be
encoded as `"546000"` for an `_msat`-suffixed field, or
`"546"` for a `_sat`-suffixed field.

### On-chain Feerates

###### Link: LSPS0.onchain_fee_rate

On-chain feerates MUST be expressed in units of millisatoshi per
weight unit, or equivalently, satoshi per 1000 weight units
(sats/kWU).

The minimum feerate when using satoshi per 1000 weight units is
253sat/kWU, or approximately 1.0sat/vbyte.

On-chain feerates MUST be encoded as JSON integral numbers.

For example, the minimum feerate would be encoded as `253`.

### Proportional / Parts-per-million

###### Link: LSPS0.ppm

Proportional numbers (i.e. anything that humans might typically
express as a perentage) MUST be expressed in units of
parts-per-million.

Parts-per-million units MUST be encoded as JSON integral numbers.

For example, 0.25% would be encoded as `2500`.

> **Rationale** This is its own type so that fractions can be
> expressed using this type, instead of as a floating-point
> type which might lose accuracy when serialized into text.
> This is effectively a fixed-point number format.
> Using parts-per-million gives granularity smaller than a
> percentage does.
> Lightning Network BOLT specifications already use the
> parts-per-million unit for proportional channel feerates.
>
> We expect that proportional amounts would be smaller than
> 100% or 1.0, which would be encoded as the JSON integral
> number 1000000, which is small enough to easily fit into
> IEEE 754 numbers or 32-bit signed integers with no loss
> of significant bits.

### Short Channel Identifiers (SCID)

###### Link: LSPS0.scid

SCIDs MUST be encoded as a JSON string containing the
"human-readable" format of `BBBxTTTxOOO`, as defined
in [BOLT7 Definition of `short_channel_id`][].

[BOLT7 Definition of `short_channel_id`]: https://github.com/lightning/bolts/blob/29c14c6e12cbdf33f6b724094c81658a614d2e02/07-routing-gossip.md#definition-of-short_channel_id

`BBB` is the top 24 bits in decimal text.
`TTT` is the middle 24 bits in decimal text.
`OOO` is the lowest 16 bits in decimal text.
`x` are literal lowercase `x` characters.

For example, an SCID which would be hex-dumped as the binary
blob `083a8400034d0001` when encoded in a typical BOLT binary
encoding, would be written as the JSON string `"539268x845x1"`.

> **Rationale** This format is a recognizable and distinctive
> format for SCIDs, and helps separate the SCID type from other
> types.

### SECP256K1 Points / Public Keys / Lightning Network Node IDs

###### Link: LSPS0.pubkey

Lightning Network node IDs are SECP256K1 ECC public keys, which
are points on the SECP256K1 elliptic curve, as noted in
[BOLT8](https://github.com/lightning/bolts/blob/50b7391a6ef5310021c2a6378334e65e04e46876/08-transport.md?plain=1#L5-L6).

SECP256K1 elliptic curve points MUST be encoded as a JSON string
of the hexadecimal dump of the compressed DER encoding of the
point.

The compressed DER encoding is a 33-byte representation, and the
JSON string hexadecimal dump would therefore be encoded as 66
characters (hexadecimal digits).
The first byte is either the byte `0x02` for an even Y coordinate,
or `0x03` for an odd Y coordinate, while the rest of the bytes is
the big-endian 32-byte integer representation of the X coordinate.

In contexts where the case is significant (for example, if it
will be committed to by some hash or signature) then the
representation MUST be in lowercase.
Otherwise, readers MUST allow both lowercase and uppercase
hexadecimal digits.

Readers SHOULD validate that the point is indeed on the SECP256K1
curve.

For example, the SECP256K1 generator point `G` would be written
as the JSON string
`"0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"`.

> **Rationale** Bitcoin and Lightning have standardized on this
> format, and traditionally used lowercase for hexadecimal digits.

### Lightning Network Connection Strings

###### Link: LSPS0.connection_string

Lightning Network connection strings describe a Lightning Network
Node ID and one way to connect to that node.

Connection strings MUST be encoded as a JSON string, with three
parts:

1.  The Lightning Network Node ID, encoded as the hexadecimal
    dump of the compressed DER encoding, followed by the `@`
    sign.
2.  The address, which follows the `@` sign and lasts to the
    last `:` in the string.
    Address formats allowed are those defined in the BOLT
    specifications:
    * [IPv4 addresses](https://datatracker.ietf.org/doc/html/rfc4001).
    * [IPv6 addresses](https://datatracker.ietf.org/doc/html/rfc5952).
    * [A Torv3 hidden service](https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt).
    * A DNS-resolvable name.
3.  A port number, which is started by the last `:`
    in the string, and is a decimal text representation of
    the number.

For example, a node with ID `0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`
on an IPv6 `localhost` listening on the default
port 9735 might be expressed as the JSON string:
`"0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798@::1:9735"`

A reader SHOULD extract the node ID as the part of the string until
the first `@` character, and the port number as the part of the
string from the last `:` character to the end of the string,
and the address as between the first `@` to the last `:`, then
individually parse and validate each part.

### On-chain Addresses

###### Link: LSPS0.onchain_address

An on-chain address MUST be a SegWit address, from version 0 to
any future version.

An on-chain address MUST be encoded as a string containing the
SegWit address:

* For SegWit v0 addresses, encoded using [bech32][].
* For SegWit v1 (Taproot) or later, encoded using [bech32m][].

[bech32]: https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
[bech32m]: https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki

Readers MUST support, and writers SHOULD only emit:

* SegWit v0 addresses with 20 byte commitment (P2WPKH) or
  32 byte commitment (P2WSH).
* SegWit v1 addresses with 32 byte commitment (Taproot).

Readers MAY support other SegWit versions.

### Lightning Network Node Signatures

###### Link: LSPS0.ln_signature

Signatures generated by a particular Lightning Network node, with a
particular known node ID, MUST be generated and represented using
the [LND `signmessage` #specinatweet][], and encoded as a JSON string
containing the zbase32 encoding:

> `zbase32(SigRec(SHA256(SHA256("Lightning Signed Message:" + msg))))`.
> `zbase32` from https://philzimmermann.com/docs/human-oriented-base-32-encoding.txt
> and `SigRec` has first byte 31 + recovery id, followed by 64 byte sig.

[LND `signmessage` #specinatweet]: https://web.archive.org/web/20191010011846/https://twitter.com/rusty_twit/status/1182102005914800128

> **Rationale** This non-BOLT specification is widely immplemented
> and is commonly used in many bespoke protocols, often used to
> prove that a particular user owns a particular node, or as an
> informal way to reduce spam by requiring that a participant
> prove they have a published Lightning Network node.

CLN, LND, and Eclair all have a `signmessage` command that allows
generation of this signature, and verification can be implemented
using (**Non-normative**) e.g. [ln-verifymessagejs][].

[ln-verifymessagejs]: https://github.com/SeverinAlexB/ln-verifymessagejs

Other LSPS specifications MUST specify that messages to be signed
include the string `LSPS` followed by the LSPS specification
number, as well as a human-readable ASCII text warning not to sign
the message manually.
For example, a hypothetical LSPS999 might specify:

 ```Markdown
 The signature MUST sign a message with the template:

     "LSPS999: DO NOT SIGN THIS MESSAGE MANUALLY: I will make an `OP_RETURN` output with ${hash}."
 ```

> **Rationale** `signmessage` is widely used in
> many bespoke protocols where a verifier will ask a human operator
> to prove they control a node by signing a specified message
> from the verifier.
> The verifier could then provide a template from an LSPS
> specification, hoping to get a signature that could be used
> in an LSPS protocol to prove something other than what the
> human operator expected.

### Datetimes

###### Link: LSPS0.datetime

Particular points of time in the modern era (a "datetime") MUST be
encoded as a JSON string containing the [ISO 8601][] format
`"YYYY-MM-DDThh:mm:ss.uuuZ"`.
These are always in the UTC timezone.

[ISO 8601]: https://www.iso.org/iso-8601-date-and-time-format.html

### Binary Blobs (Raw Transactions, PSBTs, Lightning onions, etc.)

###### Link: LSPS0.binary_blob

Binary blobs MUST be encoded as a JSON string containing the
Base 64 encoding of the binary blob, as described in [RFC 4648
Section 4][].

[RFC 4648 Section 4]: https://datatracker.ietf.org/doc/html/rfc4648#section-4

Padding characters `=` MUST be used.

> **Rationale** All characters in the [RFC 4648 Section 4][]
> Base 64 encoding can be inside a JSON string without any escapes,
> leading to an encoding that is only 33.33% longer than the
> equivalent straight binary encoding, while having reasonably
> simple encoding and decoding implementations.
>
> Although padding is unnecessary, as the length of a JSON string
> can be determined from the `"` delimiters,
> some base64 parsers completely reject the input if you do
> not include the padding `=` characters, despite their
> uselessness, as this [tweet](https://twitter.com/fiatjaf/status/1558525040374718465)
> laments.

### Transaction IDs

###### Link: LSPS0.txid

Transaction ID without witness data as defined in [BIP0141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#transaction-id). 

It MUST be converted to little-endian to enable searching and MUST be encoded as a 64 character HEX string.

> **Rationale** We use the little-endian format to let users easily copy-paste the txid to a block explorer.

Example:

```json
{
  "txid": "F27C97F46ED7281A3EFA7287410082EBA0CD1424D72703A217E435EA840957B0"
}
```



### Output Index

###### Link: LSPS0.output_index

`output_index` is the 0 based index for transaction output UTXOs. It is a maximum of [16 bits (2 bytes)](https://github.com/lightning/bolts/blob/aad959a297ff66946effb165518143be15777dd6/07-routing-gossip.md#definition-of-short_channel_id). It MUST be represented as a JSON integer (number).

Example:

```json
{
  "vout": 0
}
```


### Outpoints

###### Link: LSPS0.outpoint

An outpoint consist of a `LSPS0.txid` and a `LSPS0.vout`. It is defined in [BOLT0](https://github.com/lightning/bolts/blob/master/00-introduction.md#outpoint). It MUST be encoded as a JSON string in the `txid:vout` format.

> **Rationale** The `txid:vout` format is conveniently used in block explorers and can just be copy pasted to search for the affected UTXO.

Example:

```json
{
  "outpoint": "F27C97F46ED7281A3EFA7287410082EBA0CD1424D72703A217E435EA840957B0:0"
}
```


## References

All data types defined here MUST be referenced via the provided anchor link.

### Example

The field `min_initial_client_balance_sat` is of data type `LSPS0.sat`. You MUST reference it with an anchor link:

- `min_initial_client_balance_sat` [<LSPS0.sat>][] This field is used to describe the minimal initial channel balance on the client side.

[<LSPS0.sat>]: ./common-schemas.md#link-lsps0sat

### Convenient Schema
To minimize the characters typed, use the following markdown schema.

```markdown
- `min_initial_client_balance_sat` [<LSPS0.sat>][] This field is used to describe the minimum initial channel balance on the client side.
- `max_initial_client_balance_sat` [<LSPS0.sat>][] This field is used to describe the maximum initial channel balance on the client side.
- `onchain_address` [<LSPS0.onchain_address>][] On-chain address to pay your fee to.
- `lsp_connection_string` [<LSPS0.connection_string>][] Connection string of the LSP.

[<LSPS0.sat>]: ./common-schemas.md#link-lsps0sat
[<LSPS0.onchain_address>]: ./common-schemas.md#link-lsps0onchain_address
[<LSPS0.connection_string>]: ./common-schemas.md#link-lsps0connection_string
```
`#link-lsps0sat` is the anchor link defined in this doc. It is defined as a h6 header (6 times #).

`./common-schemas.md` is the relative link to this document and needs to be set individually.






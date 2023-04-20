# LSPS4 Continuous JIT Channel

| Name    | `continuous_jit_channels`  |
| ------- | -------------------------- |
| Version | 1                          |
| Status  | For Implementation         |

## Motivation

It is logical to users that before they can pay using some
currency, they must first receive that currency.

However, by default, Lightning Network users cannot receive
unless somebody first creates a channel to them.
And whoever creates that channel needs to pay mining fees,
as well as possibly get revenue for this service.

Generally, this means that a channel must first be bought,
often using Bitcoin, before a user can actually receive
Bitcoins over Lightning.
This means the user must first possess onchain Bitcoins
before they can receive Lightning Bitcoins; they cannot
simply "jump into" Lightning and receive their first
ever Bitcoins on Lightning.

JIT Channels allow the channel opening fee to be paid from
the first payment to the user.
This allows users to "jump into" Lightning easily.

Continuous JIT Channels are a logical extension of the JIT
Channels concept.
Whenever the user has insufficient inbound liquidity and
receives a payment, the user can use part of the payment
to pay for the channel open.
The LSP holds onto such payments until both the user and
the LSP can agree on the channel opening fee, then opens
the channel and deducts from incoming payments.

### Trust Models

As of this version, there are two trust models:

* The LSP trusts the client to actually claim their payment once the channel is
  opened and the payment is forwarded.
* The client trusts the LSP to actually open a real channel with a valid funding
  transaction and valid funding outpoint.

If the client does not claim the payment, and the funding transaction confirms,
then the LSP will have its funds locked in a channel that was not paid for.
Even if the LSP disables all other uses of the channel until the payment paying
for the channel is claimed, the client may then refuse all mutual close attempts
in retaliation (by disconnecting), and force the LSP to unilaterally close,
which locks the funds until the `to_self_delay` (indicated by the client, and
imposed on the LSP) has passed.
The LSP can protect against this (i.e. LSP does not trust the client) by simply
not broadcasting the funding transaction until after it gets the preimage.

Similarly, the LSP may provide a random number for the transaction ID of the
funding transaction during opening, then forwarding as "normal" into the
invalid channel and then receiving the preimage.
The LSP can then spuriously fail any attempt to send out the client-received
funds into the public network.
In turn, the client is unable to claim its received funds, even though the LSP
would be able to claim the funds from the incoming forward.
Even if the client wanted to unilaterally drop the channel on-chain, as the
funding transaction is not valid and will never appear on the blockchain,
it will be unable to claim the funds.
The client can protect against this (i.e. client does not trust the LSP) by
simply not providing the preimage until it sees the funding transaction has
been broadcast and is in its own local mempool.

If neither the client nor the LSP trust each other, they will deadlock: the
untrusting LSP withholds the funding transaction until it sees the payment
preimage, the untrusting client withholds the preimage until it sees the
funding transaction.

In an "LSP trusts client" model, the client MAY send the preimage
immediately, or MAY wait for the funding transaction to be broadcast and
validated as being correct before sending the preimage, or MAY wait for 1
or more confirmations before sending the preimage.

As the client is allowed to defer preimage sending until there is no
need to trust the LSP (i.e. the funding transaction is deeply confirmed
enough that reorganization is negligible), the LSP cannot reliably
double-spend the funding transaction.
In short, the client has the option whether or not to trust a 0-conf
funding transaction from the LSP.
If the client does not trust the LSP, the client SHOULD wait for at
least 1 confirmation and check that the funding transaction output is
the expected script and value.

In a "client trusts LSP" model, the client MUST transmit the preimage
as soon as the incoming HTLC has been irrevocably committed.
The LSP can cheat in this model by giving a random number for the funding
transaction ID, and the client cannot validate this, as the LSP is
allowed to not broadcast the funding transaction until the client transmits
the preimage.

The LSP SHOULD default to "LSP trusts client" model.

The LSP MAY signal a "client trusts LSP" model, and a field is specified
for the LSP to do this.
The LSP might do this for example if it detects that it is being attacked
and needs to protect its funds, even though most of the time it is in
"LSP trusts client" model.

A client that does not want to trust any LSP can simply refuse to use an
LSP that signals "client trusts LSP".

LSPs that implement this API SHOULD include mitigations to allow them to
determine if clients can be trusted, or to detect and protect against attacks
on the LSP.
How those mitigations are performed is beyond the scope of this specification.

### Actors

The 'LSP' is the API provider.
The 'client' is the API client.
The 'payer' is an entity that pays to the client.

## Flow

Overview:

* The client requests for an SCID it can use in an invoice via
  `lsps4.reserve_scid`.
  The client generates the invoice using the given SCID, and issues
  the invoice to the payer.
* Payer pays, forwarding towards the LSP with the given SCID as the
  next hop.
  When payment arrives at the LSP, it creates pending payment parts
  (PPPs, a fancy invented term for HODL HTLCs) for it.
* LSP notifies the client with `lsps4.you_have_new_ppp`, indicating that
  the client has a new pending payment part, so that the client can
  read them and process them.
* Client attempts to have the LSP directly forward the PPP(s) of a
  single payment via the `lsps4.forward_ppps` API call.
  If this succeeds and all the HODL HTLCs arrive at the client, the
  client can claim them using normal BOLT operations and this flow
  ends.
  However, if it fails due to lack of inbound liquidity towards the
  client, the client proceeds below.
* Client determines the current LSP fees for opening a new channel
  via `lsps4.get_opening_fee`.
* Client accepts the opening fee, and determines a set of PPPs that
  are sufficient to pay for it.
  Client provides this set to the LSP via `lsps4.open_with_ppps`.
* The LSP opens a channel with the client, and deducts the opening
  fee from the given set of PPPs.

> **Rationale** An alternative flow would be for the client to provide a
> payment hash for the LSP to recognize, instead of the LSP providing a
> SCID for the client to use in its invoice.
> However, this has drawbacks for some future modifications of the Lightning
> protocol:
>
> - Protocols where the payer generates the payment preimage and gives
>   the preimage to the payee inside the encrypted onion are incompatible
>   with having the client (the payee) provide the payment hash to the
>   LSP beforehand.
>   - Current plans for asynchronous receive use such a scheme.
> - PTLCs are supposed to improve privacy by having each hop have a
>   different blinding factor, but for the LSP to recognize the payment
>   point, it would have to know the blinding factor towards the client,
>   which reduces the privacy of the client in a multipath payment.
>   - PTLCs would allow a payer to buy a private key from a payee
>     (client in this context); normally the per-hop blinding factors
>     would hide the sold private key from forwarding nodes, but due to
>     the LSP having to know the exact payment point and the blinding
>     factor towards the client / payee, it would also learn the sold
>     private key.

### 0.  API Version Negotiation

The client tells the LSP about the versions of this specification
that the client supports, via the `lsps4.select_version` call,
which it must perform on every connection.
This call takes as parameters:

```JSON
{
  "client_versions": [1]
}
```

`client_versions` is an array of integral non-zero positive
JSON numbers, in any order.
This represents a set of API versions that the client supports.

The LSP MUST select the highest-numbered version in the
`client_versions` that the LSP also supports.
If the LSP does not support any of the versions, the
`lsps4.select_version` call will fail with the following error
code (error `code` is the number in parenthesis):

* `unsupported_version` (1) - The LSP and the client do not
  have a common API version that both of them support.

If the LSP is able to find a version in the `client_versions`
array that is supported by the LSP, then the LSP selects the
highest-numbered version it can find as the "current client
version", which it retains as long as the client is connected.
If the client disconnects, the LSP forgets the current client
version for that client.
On connection, the LSP sets the current client version to 0,
which is an invalid version, indicating that the rest of the
LSPS4 API cannot be accessed.

On success, the result of the `lsps4.select_version` call
is:

```JSON
{
  "api_version": 1
}
```

`api_version` is the version that the LSP selected among
`client_versions`.

On connection establishment, the client MUST call `lsps4.select_version`
to negotiate the API version with the LSP.

All APIs other than `lsps4.select_version` have the following error code
defined (error `code` is the number in parenthesis):

* `unsupported_version` (1) - The client and the LSP have not yet
  negotiated a common API version via `lsps4.select_version`.

As of this version of the specification, all the APIs below are
for API version 1.

### 1.  Request SCID

The client can request two types of SCIDs:

* Permanent.
  The client can have at most one permanent SCID.
* Ephemeral.
  The client can have any number of ephemeral SCIDs, up to some LSP-defined
  limit.

A permanent SCID is actually only permanent as long as the LSP and client
have a channel in "normal operation" state (after both peers have sent
`channel_ready`, and before either has sent `shutdown`, and no commitment
transaction has confirmed).
If the LSP thinks there are no more "normal operation" channels, the
permanent SCID has a defined "expiry time"; if the current time has
exceeded the expiry time then the LSP forgets the permanent SCID.

An ephemeral SCID also has an "expiry time", and is forgotten by the
LSP as soon as the expiry time has passed, regardless of the existence
or non-existence of a channel between the LSP and the client.
It is also forgotten if at least one forwarded payment using that SCID
has been successfully claimed (`update_fulfill_htlc`) by the client
(but not if the client fails the HTLC).

* Permanent: Forget if past expiry time AND no channel.
* Ephemeral: Forget if past expiry time OR payment succeeded.

> **Rationale** All SCIDs, including the "permanent" one, have an
> expiry time in order to time-bound any extra information the
> LSP has to store for clients that request SCIDs (which are shared
> among all clients of the same LSP) and then never actually use
> them.
> The permanent SCID needs an expiry time so that the client has
> an opportunity to have a channel opened (either by opening one
> itself, or otherwise purchasing a channel from the LSP via this
> LSPS or via [LSPS1][] or [LSPS2][]) during this expiry time
> period.

[LSPS1]: ../LSPS1/README.md
[LSPS2]: ../LSPS2/README.md

Ephemeral SCIDs are provided to give privacy to clients that are
willing to use ephemeral node IDs to prevent invoices issued by
the same client from being correlated.
For each invoice, the client requests an ephemeral SCID and generates
a fresh keypair as the node ID; separate invoices thus have a
different final node ID and final hop SCID.
On receiving a payment, the client iterates over its known invoice
keypairs until it is able to decode the onion, then can claim the
funds.

> **Rationale** If ephemeral SCIDs did not exist, then even if the
> client used ephemeral node IDs, its invoices could be correlated
> by looking at the LSP and the SCID from the LSP to the client
> in the route hints of the invoices.
> Similarly, if the client did not use ephemeral node IDs, its invoices
> could be correlated by looking at the payee node ID of the invoices.
> Thus, privacy requires that a client both implement ephemeral node
> IDs, *and* request ephemeral SCIDs from the LSP.

Permanent SCIDs are provided to give a stable path to clients.
For example, the upcoming BOLT12 offers will need a stable permanent
SCID to be embedded in a blinded route.
This still provides privacy as the client can generate multiple
different BOLT12 offers with different ciphertexts of the same
blinded route.

Permanent SCIDs also allow a client to generate an invoice without
being online and connected to the LSP; it can simply cache the
permanent SCID with the LSP (as it is permanent), and use it to
generate invoices (for example, which the payer might pay using
asynchronous receive that the client can then claim at a later
time).

> **Rationale** The permanent SCID represents all channels between
> the LSP and the client, independently of the lifetimes of
> individual channels; the permanent SCID can remain valid across
> closures and re-openings of channels between the LSP and the
> client.
> If there are no channels yet, then the permanent SCID represents
> the possibility of a future channel between the LSP and the client.

The client MAY use a permanent SCID and its stable node ID for BOLT11
invoices, though this leaks the privacy of the client.
The client, if it is concerned about privacy, SHOULD use an ephemeral
SCID and a public key from a fresh key pair for BOLT11 invoices.

The client requests for an SCID via the `lsps4.reserve_scid` API,
which accepts the following parameters:

```JSON
{
  "type": "permanent",
  "expiry_time": "2023-04-20T10:08:41.222Z",
  "min_final_cltv_expiry_delta": 18
}
```

All parameters are required.

`type` is a string, either `"permanent"` to request for a permanent SCID,
or `"ephemeral"` to request for an ephemeral SCID.

`expiry_time` is the time to request [<LSPS0.datetime>][].
Expiry times have granularity up to the millisecond.

The client MUST set an expiry time that is at most 7 days (7 times
24 hours) from the current time.
The LSP MUST check the requested expiry time is at most 7 days
from the current time.

`min_final_cltv_expiry_delta` is the final CLTV delta for the
recipient, i.e. the `c` field in BOLT11 invoices.
The LSP MUST persist this together with the generated SCID if
this call succeeds.

On success, the result looks like:

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

`lsps4_scid` is the SCID reserved [<LSPS0.scid>][].
The LSP SHOULD set the highest bit of the block index (the first 24-bit
number in the `BBBxTTTxOOO` format), and select the rest of the bits
of the block, transaction, and output indexes by random;
the LSP MAY use any other method of generating this SCID, and
MAY use an SCID that could be interpreted as an actual channel, provided
the LSP can uniquely identify the client.
The LSP MUST ensure that the SCID is unique at the LSP, that is, no
other peer (client or non-client) of the LSP has the same SCID allocated
to it, whether or not the SCID could refer to an actual possible channel
outpoint.

> **Rationale** Setting the highest bit of the block index, and
> selecting the rest by random, reduces the probability of the same
> randomly-selected SCID, giving 63 bits of entropy.
> The blockheight 8,388,608 is unlikely to be reached for about ~120
> years from the time of this writing (2023).
>
> Regardless, the only true requirement is that the SCID refers to
> only one possible peer, and the LSP is free to use any policy that
> fits.
> The LSP needs some database or lookup table for SCIDs in order to
> implement LSPS4 anyway, and could use a simple incrementing 64-bit
> counter that it reinterprets as an SCID with the block index as the
> highest 24 bits (and checking it against "real" SCIDs it has with
> peers); every actual block mined would then grant a space of
> 1,099,511,627,776 (= 2^40) possible SCIDs, with only a tiny few of
> them likely to become real SCIDs with peers of the LSP node, even
> if the counter is started from 0.

`expiry_time` is the expiry time for the SCID [<LSPS0.datetime>][].
If the client asked for an `"ephemeral"` SCID, then the `expiry_time`
MUST be the same as in the request parameters.

If the client asked for a `"permanent"` SCID, and the LSP currently
remembers a permanent SCID for the client, the LSP MUST return
the current permanent SCID and its current `expiry_time` and
`min_final_cltv_expiry_delta`, which may be different from those
in the request parameters.
In this case, the `expiry_time` can be in the past, for example if
there is currently a channel between the LSP and the client.

> **Rationale** There is a race condition where the client reserves
> a permanent SCID, but crashes before it can persist the returned
> permanent SCID.
> In that case, the LSP has already reserved the SCID but the client
> has forgotten it by accident.
> Thus, repeating that request should be idempotent until the LSP
> has also forgotten the permanent SCID.
>
> The LSP should not change the expiry time for the permanent SCID
> just because the request was repeated, and thus should return
> the original expiry time for the permanent SCID, as otherwise
> the client can keep repeating the request without ever creating
> a channel with the LSP, and thus force the LSP to maintain this
> stored data without being compensated.
>
> `expiry_time` is set by the client instead of being determined
> and set by the LSP as the `expiry_time` would be determined from
> the `expiry`/`x` field of the BOLT11 invoice the client intends
> to issue.

Otherwise, if a fresh `"permanent"` SCID was created, then the
`expiry_time` MUST be the same as the request parameter.

`ln_base_fee_msat` is the base fee that the client MUST indicate as the
base fee for the given SCID [<LSPS0.msat>][].
`ln_prop_fee` is the proportional fee the client MUST indicate as
the proportional / parts-per-million fee for the given SCID
[<LSPS0.ppm>][].

`cltv_expiry_delta` is the CLTV difference between the LSP-inbound
timelock and the to-client timelock.

For `"ephemeral"` SCIDs, the `ln_base_fee_msat`, `ln_prop_fee`, and
`cltv_expiry_delta` are fixed for the lifetime of the SCID.

For `"permanent"` SCIDs, the `ln_base_fee_msat`, `ln_prop_fee` and
`cltv_expiry_delta` MAY be changed by the LSP at any time, with a
grace period during which the LSP MUST accept both the previous
and current settings.
The client is notified by this change via the
`lsps4.i_changed_permanent_fee_rates` notification, to be
described later.

`lsps4.reserve_scid` has the following defined errors (error `code`
in parenthesis):

* `expiry_too_long` (2) - The `expiry_time` requested by the client
  exceeds 7 days from now.
  The `data` object contains a field `current_time` which is a
  [<LSPS0.datetime>][] indicating what the LSP believes is the
  current time.
* `too_many_scids` (3) - The client requested for an SCID but the
  LSP has hit some internal limit on the number of SCIDs.

#### Invoice Generation

The client MAY then generate a [BOLT11][] invoice.

[BOLT11]: https://github.com/lightning/bolts/blob/f7dcc32694b8cd4f3a1768b904f58cb177168f29/11-payment-encoding.md

If the client generates a BOLT11 invoice, the client MUST set
the following:

* The sum of `timestamp` and `x`/`expiry` fields is equal to
  or less than the `expiry_time` from `lsps4.reserve_scid` if
  using an ephemeral SCID, **or** if using the permanent SCID
  and has no channels with the LSP.
  * If using the permanent SCID while a channel exists with
    the LSP, then the `x` field can be any value selected by
    the client.
* `n` is the client node ID, or any public key whose private
  key the client knows.
* `r` contains a single route hint with a single hop, from the
  result of `lsps4.reserve_scid`:
  * `pubkey` is the LSP public node ID.
  * `short_channel_id` is the `lsps4_scid`.
  * `fee_base_msat` and `fee_proportional_millionths` are the
    `ln_base_fee_msat` and `ln_prop_fee`, respectively.
  * `cltv_expiry_delta` is `cltv_expiry_delta`.
* `c` contains a `min_final_cltv_expiry_delta` that is at least
  +2 higher than its normal setting.
  * The client MUST impose its normal setting on actual received
    payment.
    For example, suppose the client normally uses a
    `min_final_cltv_expiry_delta` of 144.
    For a JIT Channels invoice, it sets it to 146 in the invoice,
    but only still expects 144 when the payment reaches it
    even for JIT Channels invoices.
  * If the client intends to wait for confirmation before sending
    the preimage (when `client_trusts_lsp` is not specified or
    is `false`), it MUST increase the margin to 2 plus the number
    of confirmations it intends to wait.
    For example, if it intends to wait for 1 confirmation, it
    should add +3 to this setting.

All other fields are to be filled in by the client.

> **Rationale** Bitcoin blocks may be mined at any time.
> When a payer sends out a payment, it must "lock in" a specific
> blockheight for the timeout of the HTLC arriving to the client,
> but if a block is mined between when the payer sends out the
> payment and when the client receives it, then the timeout may
> now be below the normal `min_final_cltv_expiry_delta` of the
> client.
> BOLT requires that the payee fail this with
> `incorrect_or_unknown_payment_details` with a `height` indicating
> the new current blockheight, but the LSP cannot differentiate
> between this failure (which would cause the payer to retry with
> renewed timeouts), versus a deliberate attempt by the payer
> and client to waste LSP resources (which should cause the LSP to
> deny future service to this client).
> By adding a small +2 margin here, we greatly reduce the incidence
> of such an accident.

#### Forward Compatibility With Post-BOLT11 Payment Schemes

Ultimately, the client MUST provide the information below to
the payer:

* The client node ID (or any public key whose private key the
  client knows).
* An SCID issued from `lsps4.reserve_scid`.
  * How much the LSP wants to charge for forwards through that
    SCID.
* The LSP node ID.
* The LSP-provided `cltv_expiry_delta`.
* The expiry for the payment.
* Whether it can use MPP or not.

However, the client MAY provide this information to the payer
by means other than BOLT11 invoice, including in a form that
is not decodable by the payer, but which can be decoded by
another LN participant.

The only necessary requirement is for the LSP to see an incoming
HTLC onion that indicates the `lsps4_scid` as the next hop,
and as long as the client is able to cause the payer to do this,
it is is compatible with this protocol.

> **Non-normative** For example, the client node ID, the
> `lsps4_scid`, the LSP node ID, and the LSP-provided
> `cltv_expiry_delta` can be encoded as the last hop of
> a blinded path, for example in a BOLT12 offer or invoice.
> The payer would not have direct access to the information,
> but the introduction point of the blinded path would be able
> to start the decoding of the blinded path.

### 2. Payment

The payer now attempts to send money to the client.
The payer will now send out HTLC(s) going towards the LSP, with
the next hop being some LSPS4-reserved SCID.

#### Pending Payment Parts (PPPs)

When the LSP receives an HTLC whose onion indicates an SCID
reserved for LSPS4 as the next hop, the LSP MUST hold the
incoming HTLC and put its details into a structure called a
"pending payment part" or "PPP".
Once it has done so, the client will be notified and the client
can then parse the details of the HTLC.

The LSP MUST hold **all** HTLCs going to LSPS4 SCIDs, *even if*
the client associated with that LSPS4 SCID already has a channel
with sufficient capacity to forward those HTLCs.

> **Rationale** Suppose that the client receives a multipart
> payment, where most parts can fit in existing inbound capacity,
> but the last part cannot fit.
> Suppose that the LSP always forwards HTLCs if it can fit into
> existing capacity, and only puts HTLCs into PPPs if they cannot
> fit in existing capacity.
> In that case, then all but the last part will already be
> forwarded, without deductions for channel opening fees, to the
> client on the existing inbound capacity.
> If the last part is tiny, then it might not be able, by itself,
> to pay for a new channel open to get more capacity.
> If so, the entire payment fails to reach the client: the last
> part cannot be forwarded due to lack of capacity, and that last
> part is not big enough to buy additional capacity.
>
> Thus, it is better if all parts are first "buffered" by the
> LSP into PPPs without forwarding.
> The client can then use the entire payment to pay for new
> capacity, not just the last part that did not fit
> in existing capacity, or the client can use some reasonable
> split where some of the parts are passed through existing
> capacity while remaining parts are used to pay for new
> capacity.
>
> The LSP cannot know, on receiving just a single HTLC, whether
> this is one part of a larger multipart payment (and that the
> LSP will receive the remaining parts later), or that this is
> the entire payment.
> Thus, the LSP has to put all HTLCs it receives into PPPs that
> are held at the LSP first before the client then parses its
> details and is able to decide whether to deliver some or all
> of them into existing capacity, or to use them to pay for new
> inbound capacity.

A pending payment part is a generalized term for some payment,
or *part* of a *payment*, that is *pending* --- i.e. the LSP is
waiting for the client to decide what to do with that part before
being actually forwarded.

We use the name "pending payment part" instead of "pending HTLC"
since in the future, pending payment parts might be pending PTLCs,
or pending asynchronous receives.
The same general mechanism is still applicable --- the LSP must
still "hold on" to the HTLC/PTLC/asyncpay (i.e. not respond to
it, just wait for the client to decide what to do with it) until
the client can ask the LSP to use existing capacity if available,
or the client can arrange to pay for a new channel open,
or the client decides to refuse the payment in order to avoid
paying for new channels with this LSP (and presumably picks a
different LSP with lower channel opening fees).

Once an HTLC (or in the future, PTLC or asynchronous receive)
has been placed into a PPP stored at the LSP, the client can
then learn its details and decide on what to do with the PPP.

Each PPP has a type.
Currently, the only defined PPP type is HTLC-type PPP, also
known as `"type": "htlc"` PPPs.

LSPs MUST impose a reasonably short time frame to hold onto
PPPs, after which it should just fail the PPP regardles of
whether the client has gotten around to processing it or not.

* For HTLC-type PPPs, the hold time SHOULD be 90 to 120 seconds.

> **Rationele** The LSP needs to hold onto the PPP so that the
> client can process it, and for the messages involved to
> traverse the network towards the client.
>
> The hold time can be larger for other future PPP types; for
> example, it would be reasonable to hold onto an asynchronous
> receive request for 24 hours or more, as no actual resources
> are held on the rest of the network; thus we specify the
> required hold time based on type.
>
> Multipath payments already mandate a hold time of up to 60
> seconds at the payee;
> the additional 30 seconds in the 90-second minimum is to
> allow for client and LSP intercommunications.

If the LSP decides to fail an HTLC-type PPP due to hold
timeout, the LSP MUST fail it with a
`temporary_channel_failure` error.

An HTLC-type PPP is a hold on an incoming HTLC that is to be
forwarded to the client, as indicated by an LSPS4-reserved
SCID.
The onion for the outbound HTLC (the one that would go to
the client) can be read by the client, so that the client
can read any information in the onion, such as a
`keysend`-style preimage, or how large the total payment
would be for a multipart payment.

> **Rationale** As their name implies, the PPP system is
> designed to handle multipart payments.
> The LSP cannot decode the onion to determine how much
> the total payment would be, but the client can.
> The problem is that it is possible that only some parts
> of a multipart payment reached the LSP, and then the
> payer gives up on trying to get the remaining parts to
> the destination.
> If so, if the LSP then opens a channel to the client and
> forwards the incomplete set of parts, the client is forced
> to fail the payment (since it did not receive the
> entire payment) but now has a channel opened by the LSP,
> which the LSP should now close (additional onchain
> activity that requires mining fees) to prevent use of a
> channel that was not paid for.
>
> If the client knows beforehand what the payment size
> would be, it could simply outright inform the LSP of
> the payment size, but this would prevent MPP from
> working with variable-amount invoices ("0-amount
> invoices") where the client does not know the
> payment size beforehand.
>
> A side benefit of the PPP mechanism is that it can also
> be used to aggregate multiple small separate payments
> and use their total to pay for a channel open.
> Even if no single payment is large enough for a channel
> open, if enough payments arrive in a short enough time
> frame, the aggregate may be large enough to pay for a
> channel open.
>
> Finally, a side benefit of the PPP mechanism is that
> it also fixes the issue with prepayment probing.
> In prepayment probing, the payer takes the BOLT11
> invoice details given by the payee, but changes the
> payment has to a random number, then attempts to
> reach the payee, until it gets an error from the
> payee, indicating that it was able to successfully
> reach the payee.
> If the payee was a client of an LSP, and they already
> have a channel, then there is no problem, the client
> simply fails the payment.
> However, if the client arranged for a JIT channel,
> but the PPP mechanism did not exist, then the LSP
> has to first open a channel and then send the HTLC
> via the channel, and then the client would fail
> the HTLC due to not knowing the preimage to the
> randomly-generated hash, resulting in a channel
> open that is not paid for.
> *With* the PPP mechanism, the LSP allows the client
> to access the HTLC details, and the client can
> fail the HTLC using an error originating from the
> client, which signals to the payer that it was
> able to reach the payee, but the LSP does not
> open a channel yet, preventing a channel open
> that is not paid for.

### 3. Pending Payment Part Notification

A "pending payment part" or "PPP" is an abstraction.
As of this specification version, all PPPs are HTLCs, but in the
future, a PPP may be a pending asynchronous receive, or a PTLC.

Every PPP has a unique string identifier, the "PPP identifier".
The format of this identifier is defined by the LSP, but MUST be a
high-entropy identifier, with at least 80 bits of entropy.
The PPP identifier:

* MUST be generated with at least 80 bits of entropy.
* MAY have any format, provided:
  * MUST be encodable in an ASCII encoded JSON string without any
    string escapes needed.
  * MUST not be longer than 24 bytes (not including the `"`
    delimiters) when encoded as an ASCII-encoded JSON string.

Whenever the LSP receives an incoming payment that goes to a reserved
SCID, it:

* Generates a fresh PPP identifier for it.
* Starts a hold timer for that incoming payment, duration depending on
  its PPP type.
  * Does not respond to the inbound until after the hold timer expires,
    or the client has handled the payment fully.
* Adds the PPP to a called the "PPP Map", which maps the PPP identifier
  to the information the LSP needs to forward, handle, or fail the
  underlying payment, as well as the information that the client needs to
  process for that PPP.
  * If the hold timer expires before the client has decided to forward,
    fail, or use the PPP for opening a new channel, should also remove
    it from the PPP Map.
* Sends the `lsps4.you_have_new_ppp` notification to the client, containing
  the PPP identifier and the details the client needs to process the
  PPP.

The LSP SHOULD persist the PPP Map on-disk, so that if the LSP crashes and
then restarts quickly, the client can reconnect and re-download the PPP Map
from the LSP, improving payment experience.

However, the LSP MAY choose not to persist the PPP Map.
In this case, if the LSP crashes and restarts, it MUST fail all HTLCs that
were held in PPPs with `temporary_channel_failure`.

If the client has previously called `lsps4.reserve_scid` (even on a
previous connection), this also enables the LSP sending the notification
`lsps4.you_have_new_ppp`.

This notification contains the detail of one PPP that the LSP
wants to inform to the client.

The notification contains these parameters:

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

`ppp_id` is the PPP identifier, and is required.

`details` is an object, and is required.
`details` always has the keys:

* `amount_msat` - a monetary amount that is the pending outgoing amount
  of this PPP (i.e. the amount that the client would receive if
  it is able to claim the PPP) [<LSPS0.msat>][].
* `type` - a single-character string denoting the type of this
  PPP.

The PPP `type` may be one of the following, which defines the
additional keys the `details` object has:

* `"type": "htlc"` - HTLC-type PPP.
  Additional keys (all required):
  * `via_scid` - the short channel ID in the LSP-decoded forwarding
     onion, which matches a previous `lsps4_scid` returned from a
     previous `lsps4.reserve_scid` call and reserved for the client
     [<LSPS0.scid>][].
     This is basically the value of the type 6 (`short_channel_id`)
     field of the [BOLT 4 Payload Format][], written as an SCID
     string.
  * `hash` - the 32-byte payment hash in the hashlock branch of the
    HTLC, as a hex-encoded JSON string.
  * `cltv` - the timeout of the HTLC, a blockheight JSON integral
    number.
  * `onion` - the onion for the client, as a Base64 binary blob
    [<LSPS0.binary_blob>][].
    The binary blob format is the [BOLT 4 Packet Structure][].

[BOLT 4 Packet Structure]: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#packet-structure
[BOLT 4 Payload Format]: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#payload-format

> **Rationale** The `via_scid` field is provided to the client so
> that if the client wants to have privacy in its generated invoices,
> the client can do something like:
>
> * Ask for an ephemeral SCID in `lsps4.reserve_scid`.
> * Securely combine its node private key with the ephemeral SCID,
>   and then treat the combination as a new private key, and derive
>   the corresponding public key, called the ephemeral invoice node
>   key.
> * Use the ephemeral SCID and the ephemeral invoice node key in
>   the issued invoice.
> * When receiving this notification, the client can then re-derive
>   the same ephemeral invoice node private key, and can now decode
>   the incoming onion.
>
> This allows the client to issue stateless, private BOLT11 invoices.

On connection establishment, after the client has negotiated an
API version via `lsps4.select_version`, the LSP MUST send
`lsps4.you_have_new_ppp` notifications, one for each PPP still being
held in its PPP Map.

The client MUST ignore duplicated `lsps4.you_have_new_ppp`; if it
already knows the given PPP identifier, it can completely ignore
the notification.

> **Rationale** By having the LSP always replay the PPP Map on
> reconnection, the client can keep its copy of the PPP Map
> in-memory only; if the client crashes and restarts, the LSP
> simply resends the same PPP Map to the client.
> In addition, a disconnection may mean that the last few
> messages sent by the LSP might not have reached the client;
> thus, it is best and simplest to just send the entire PPP Map,
> as previous notifications might not have reached the client.
>
> A reconnection may occur if the client does not crash, but
> instead loses connectivity (e.g. it is a mobile app and the
> mobile unit entered a street tunnel and loses carrier signal).
> In that case the client may receive multiple copies of the
> same `lsps4.you_have_new_ppp`, and can ignore the additional one.

On receiving the `lsps4.you_have_new_ppp` notification, the client
SHOULD:

* Update its own copy of the PPP Map with the new PPP, together
  with any decoded information in the PPP, such as the onion of
  an HTLC-type PPP.
* Evaluate if its own copy of the PPP Map now has a complete
  set of PPPs for a multipath payment, and if so, should continue
  to attempt normal forwarding of the payment.

On receiving the `lsps4.you_have_new_ppp` notification, if it is a
new PPP that the client has not received before, then the client
MUST:

* Check if it cannot claim a PPP at all, such as if it is an
  HTLC-type PPP that terminates at the client, but the client does
  not know the preimage for the hash of the HTLC and cannot learn it
  from the onion.
  * If so, it MUST fail the PPP immediately with an
    `incorrect_or_unknown_payment_details` without adding it to
    its own PPP map.
    Failing is done via `lsps4.fail_ppp` detailed later in this
    document.

### 4. Attempting Normal Forwarding

For HTLC-type PPPs, the client decodes the onion using any of the
invoice node IDs it has used.
If the onion cannot be decoded by any of the invoice node IDs the client
knows about, the client MUST fail the PPP with an `invalid_onion_hmac`
error.
The client may fail a PPP via `lsps4.fail_ppp`.

The client can then determine if enough MPP payment parts have reached
the client as actual HTLCs or PTLCs, and as LSP-held PPPs, and
can then attempt to have the LSP forward all those parts to the client.
This attempt to normally forward is done via `lsps4.forward_ppps`.

The client can always **attempt** to forward --- if there is
insufficient liquidity, `lsps4.forward_ppps` will fail with a specific
error code and the client can proceed to steps 5 and 6.
This simplifies the client design as it can delegate checking for
liquidity to the LSP.

> **Rationale** Even if the client does pre-checking if it has
> sufficient inbound liquidity before attempting to forward
> normally, the LSP can always refuse to forward even if there is
> enough inbound liquidity.
> Presumably, if the LSP did indeed hold onto incoming HTLCs,
> it has a stake in getting the payment resolved, and will
> forward immediately if possible.

#### Forwarding Pending Payment Parts

The client can use the `lsps4.forward_ppps` API, which accepts
the parameters:

```JSON
{
  "ppp_ids": ["id#abcdefghijklmnop"]
}
```

`ppp_ids` is a required non-empty array of PPP string identifiers,
for the PPPs to be forwarded or otherwise "normally handled".

> **Rationale** The "normal handling" of an HTLC would be to
> just forward it to the client using any available channel
> capacity from the LSP to the client, but the "normal handling"
> of an asynchronous receive would be to respond to the
> sender or sender LSP with the notification that the client
> is online.

On receiving this request, the LSP MUST **atomically**
perform the following:

* Look up all `ppp_ids` in the PPP Map.
  * If any PPP is not found, fail with `nonexistent_ppp_id`
    described below.
* Do a spot check of whether the client has enough
  inbound liquidity to actually fulfill all the `ppp_ids`.
  * For HTLC-type PPPs, there must be enough inbound
    liquidity in one channel to send the HTLC amount,
    as well as to actually pay for the onchain fees of
    the HTLC for the current onchain feerate of the
    channel.
  * If this check fails, fail with
    `insufficient_liquidity` described below.
    Do *not* forward *any* of the PPPs if at least one
    cannot fit in the available capacity from the LSP
    to the client; either all are forwarded or none
    are.
* Suppress the hold timers of the `ppp_ids`.
  * Hold timeout handling code must be atomic to this
    call, if this API call is sent near the timeout;
    either the timeout handler is called before this
    entire atomic section (and the PPP is no longer
    in the PPP Map and the section fails in the first
    step above) or the timeout handler is called
    after this entire atomic section (and does not
    remove this PPP and release the incoming HTLC or
    underlying payment, just notes that it would have
    timed out).
* Remove the `ppp_ids` from the PPP Map and arranges
  them to be handled normally.
  * For HTLC-type PPPs, arrange to have the HTLC be
    forwarded to the client via any available
    inbound liquidity to the client.

The above atomic section ends before normal handling of
the PPPs is completed; it is enough for the LSP to
arrange to have the payment handled without actually
forwarding the HTLCs (or appropriate handling for its
type) yet, just arranging them to occur after the atomic
section completes or after te call returns.

The `lsps4.forward_ppps` call has the following errors
defined (error `code`s in parenthesis):

* `nonexistent_ppp_id` (2) - One or more of the listed `ppp_ids` is
  not in the PPP Map; the expected case is that the hold
  timer for the PPP has passed and the LSP has failed the
  underlying PPP.
  The error `data` object contains the required field
  `unknown_ppp_ids`, which is an array of PPP identifiers
  containing the PPPs which were not found (and were
  presumably timed out).
  * The client SHOULD remove the indicated PPPs from its
    own copy of the PPP Map, and should then re-evaluate
    if it can accept the PPPs (for instance, it might
    have received a `lsps4.you_have_new_ppp` while this
    call was being handled by the LSP) in the PPP Map.
* `insufficient_liquidity` (3) - The client has insufficient
  inbound liquidity to actually receive the entire set of
  `ppp_ids`, and none of the `ppp_ids` were forwarded.
  * The client SHOULD continue to the next step (5 and 6) of
    the flow.

If the `lsps4.forward_ppps` call succeeds, the LSP SHOULD
make a best-effort attempt to deliver all the specified
PPPs.

* For HTLC-type PPPs:
  * The LSP SHOULD send an `update_add_htlc` to the client,
    via any channel that has sufficient capacity to the
    client.
    * The LSP MUST include a `via_scid` (type 65535) TLV to
      this `update_add_htlc` message, regardless of whether a
      permanent or ephemeral SCID was used.
    * `via_scid` has type 65535 and length 8.
    * The value of `via_scid` is the binary serialization of
      the SCID that is encoded as the type 6
      (`short_channel_id`) in the [BOLT 4 Payload Format][]
      that is decoded by the LSP.
      In other words, it is the same as some `lsps4_scid` that
      the LSP reserved for this client.

> **Rationale** The `via_scid` TLV is odd to comply with the
> "it's OK to be odd" rule in BOLT 0.
> The SCID is odd since it is only of interest to clients that
> use an ephemeral SCID to derive an ephemeral invoice node
> key that the client can then use to decode the onion for
> its hop.
> The client can thus generate a stateless private BOLT11
> invoice; it can claim and decode the onion without storing
> any data about the invoice, and it can get privacy if it
> ensures it uses each ephemeral SCID for only one invoice.

As noted above, the LSP SHOULD make a best-effort attempt to
deliver all the specified PPPs.
However, race conditions can cause the actual inbound
liquidity to change from when it was checked in the atomic
section, to when the LSP is now sending `update_add_htlc`s
to the client:

* The channel initiator might send an `update_fees` which
  increases the onchain feerate, such that an HTLC that would
  have barely fit in the channel, no longer fits since the
  increased onchain feerate prices it out of the channel.
* Some automated system in the client or LSP (such as the
  one checking that the `update_fees` from the remote are
  reasonable) might close a channel that the LSP was planning
  to send an HTLC through.
* The client might be trying to mess with the LSP and
  broadcasts a currently-unrevoked commitment transaction,
  closing the channel before the LSP can deliver an HTLC it
  was planning to send via that channel.
* The LSP might allow specific channels to be indicated via
  their SCID, and an unrelated payment might go through a
  channel that the LSP was intending to send one of the
  `lsps4.forward_ppps` HTLCs through.
* One party might have sent `shutdown` for a channel that
  was intended to be used for forwarding.

In such a case the LSP, for each PPP it was unable to handle
normally due to a race condition that changes the liquidity
towards the client:

* MUST do one of the following:
  * (preferred) add the PPP back to the PPP Map under a
    new PPP identifier and resumes its hold timer.
    It re-sends `lsps4.you_have_new_ppp` under the new PPP
    identifier.
    If its hold timer has already expired at this point, the
    LSP MUST NOT do this, but MUST do the alternative below
    instead.
  * (unpreferred) NOT add the PPP back to the PPP Map and
    fail the payment with `temporary_channel_failure`.

> **Rationale** Most node implementations, once you have
> indicated that the HTLC is to be forwarded "normally", will
> take over processing, and if it then turns out that there
> is insufficient liquidity, will automatically fail the
> HTLC with no way to re-handle the HTLC.
> Thus, the "unpreferred" branch is allowed (it is the only
> behavior such node implementations can do), but is still
> marked as unpreferred as it increases payment failure (the
> payer might give up retrying).
> The hope is that node implementations will eventually be
> modified so that the preferred branch can be taken instead.

Regardless, normal BOLT-specified handling on the client
would still work despite this issue:

* If the payment is a single part:
  * If the preferred branch is taken by the LSP, the
    client gets notified and can retry the
    `lsps4.forward_ppps` call, hopefully now with the updated
    information on available liquidity and onchain fees
    allowing the call to correctly report
    `insufficient_liquidity`.
  * If the unpreferred branch is taken by the LSP, the
    payer may still retry, and the client would again be
    notified of the new PPP, with the same result as above.
* If the payment is multipart:
  * Any parts that were successfully forwarded will eventually
    time out with the remaining parts undelivered, and a
    BOLT-compliant client will `mpp_timeout` the delivered
    parts.
  * Any parts that were not successfully forwarded will be
    re-added to the client copy of the PPP Map (if the LSP
    takes the preferred branch) for re-evaluation by the
    client, or might eventually be retried by the payer
    (if the LSP takes the unpreferred branch).

#### Failing Pending Payment Parts

A pending payment part can be failed by the client using the
`lsps4.fail_ppp` API.

For example, suppose that the payer attempts a multipath
payment to the client via the LSP.
Some of the parts arrive at the LSP and are converted to PPPs.
However, other parts fail to reach the LSP, and eventually
the payer gives up.

The client, as a "good citizen", SHOULD fail the multipath
parts that reached the LSP using an `mpp_timeout` error, in
order to free up network resources.

`lsps4.fail_ppp` accepts the parameters:

```JSON
{
  "ppp_id": "id#abcdefghijklmnop",
  "details": {
    "type": "htlc",
    "error_onion": "AbCdEfGhIjKlMnOpQrStUvWxYz+/"
  }
}
```

`ppp_id` and `details` are required.

`ppp_id` is the PPP string identifier for the PPP to be failed.

`details` is an object whose keys are dependent on the type of
the PPP.
It always has the field `type`, a one-character JSON string which
indicates the type of the PPP.

Depending on the PPP `type`, the additional fields of `details`
are:

* `"htlc"` - HTLC-type PPP.
  Additional fields:
  * `error_onion` - an *optional* binary blob containing the error
    onion [<LSPS0.binary_blob>][].
    The binary blob format is described in
    [BOLT 4 Returning Errors][], with the client as the *erring
    node*.
    If not specified, the LSP will issue a
    `temporary_channel_failure` error at its hop, with the LSP
    as te *erring node*.

[BOLT 4 Returning Errors]: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#returning-errors

The LSP MUST look up the `ppp_id` in the PPP Map, and check that
the PPP type matches the `details.type` field.
The LSP MUST then check that the `details` field object
contains the fields necessary for that type.

If the above checks succeed, the LSP removes the PPP from the
PPP Map and fails the PPP based on its type.

* For HTLC-type PPPs:
  * If the client gave an `error_onion`, the LSP adds its own
    encryption layer to the client `error_onion` and
    propagates the error onion back to the payer via
    `update_fail_htlc`.
  * If the client did not give an `error_onion`, the LSP issues
    a `temporary_channel_failure` error at its hop, with the LSP
    as the *erring node*, and propagates its error back to the
    payer via `update_fail_htlc`.

On success, `lsps4.fail_ppp` returns no results `{}`.

In addition to `unsupported_version`, `lsps4.fail_ppp` may
error with the below error codes:

* `nonexistent_ppp_id` (2) - the PPP is not in the PPP Map
  (it never existed, or was removed due to timing out its
  hold time).
* `invalid_details` (3) - the `details.type` does not
  match the PPP type, or the `details` object does not
  contain the fields expected of its type.

### 5. Determining Opening Fee

The client can query for the cost of opening a new inbound
channel via the `lsps4.get_opening_fee` call, as well as any
limits the LSP imposes, and parameters for payment.

The client MUST request `lsps4.get_opening_fee` to determine the
`opening_fee` of the LSP and its related parameters.
It takes the parameters:

```JSON
{
  "token": "SECRETDISCOUNTCOUPON100"
}
```

`token` is an *optional*, arbitrary JSON string.
This parameter is intended for use between the client and the LSP; it
may be used to negotiate for a better fee rate than the LSP otherwise
offers (i.e. a "discount coupon"), or for the LSP to actually offer
this service to the client (i.e. an "API key") instead of failing to
provide any offers, or for any other purpose.

`lsps4.get_opening_fee` has the following errors defined in addition to
`unsupported_version` (error code numbers in parentheses):

* `unrecognized_or_stale_token` (2) - the client provided the `token`,
  and the LSP does not recognize it, or the token has expired.

Example `lsps4.get_opening_fee` result:

```JSON
{
    "opening_fee_params_menu": [
        {
            "min_fee_msat": "546000",
            "proportional": 1200,
            "client_htlc_minimum_msat": 100,
            "valid_until": "2023-02-23T08:47:30.511Z",
            "min_lifetime": 1008,
            "max_client_to_self_delay": 2016,
            "promise": "abcdefghijklmnopqrstuvwxyz"
        },
        {
            "min_fee_msat": "1092000",
            "proportional": 2400,
            "client_htlc_minimum_msat": 100,
            "valid_until": "2023-02-27T21:23:57.984Z",
            "min_lifetime": 1008,
            "max_client_to_self_delay": 2016,
            "promise": "abcdefghijklmnopqrstuvwxyz"
        }
    ],
    "min_payment_size_msat": "1000",
    "max_payment_size_msat": "1000000",
    "client_trusts_lsp": false
}
```

The `opening_fee_params_menu` is an array of `opening_fee_params`
objects.
The LSP MAY return an empty array; in which case, the client currently
cannot use JIT Channels with this LSP.

An `opening_fee_params` object describes how much the LSP will charge
for a channel open, and until when the described fee rates will be
considered valid.
An `opening_fee_params` object MUST have all of the following fields,
and MUST NOT have any additional fields:

* `min_fee_msat` is the minimum fee to be paid by the client to the
  LSP.
  It is a string containing the decimal encoding of a number of
  millisatoshis.
  [<LSPS0.msat>][]
* `proportional` is a parts-per-million number that describes how many
  millisatoshis to charge for every 1 million millisatoshis of payment
  size for the first payment.
  If the proportional fee is less than the `min_fee_msat`, then the
  `min_fee_msat` is paid instead of the proportional times payment size
  divided by 1 million.
  [<LSPS0.ppm>][]
* `client_htlc_minimum_msat` is the size of the smallest HTLC that can be sent
  from the LSP to the client.
  It is the same as the `htlc_minimum_msat` field in the
  [BOLT2 Channel Establishment][] `accept_channel` message sent by the
  client.
  [<LSPS0.msat>][]
* `valid_until` is a datetime (as an ISO8601 string) up to which this specific
  `opening_fee_params` is valid, and also serves as the timeout for the JIT
  Channel flow, if this particular object is selected.
  The LSP SHOULD provide a `valid_until` time that is at least 10 minutes from
  the current time.
  [<LSPS0.datetime>][]
* `min_lifetime` is a number of blocks that the LSP promises it will keep the
  channel alive without closing, after confirmation.
  The LSP MUST NOT close the channel if both conditions below are satisfied:
  * It was paid via a JIT channel payment to open this channel (i.e. the
    client released the preimage to the HTLC(s) that have had the value of
    the opening fee deducted), AND
  * The channel funding transaction confirmation depth is below the
    `min_lifetime`.
* `max_client_to_self_delay` is a maximum number of blocks that the client
  is allowed to set its `to_self_delay` parameter.
  The client-side `to_self_delay` is the delay imposed on the LSP to recover
  the funds in case the LSP has to perform a unilateral close on the channel
  and defines the security of the client against malign or hacked LSPs.
* `promise` is an arbitrary LSP-generated string that proves to the LSP that
  it has promised a specific `opening_fee_params` with the specific
  `min_fee_msat`, `proportional`, `client_htlc_minimum_msat`, `valid_until`,
  `min_lifetime`, and `max_client_to_self_delay`.

> **Rationale** A "minimum" fee is used instead of an additive base fee as
> this mimics similar financial services.
>
> A time limit (`valid_until`) on the offerred fee rates is
> needed as channel opening is an on-chain activity, and on-chain fee rates
> need to be reflected in the channel opening fee rates the LSP charges.
> The LSP cannot predict future on-chain fee rates, and thus must estimate
> how long its current on-chain fee rate estimates are valid for.
> The LSP may offer multiple parameter sets with varying time limits.
>
> Both `min_lifetime` and `max_client_to_self_delay` in total impose a
> maximum limit on how long a client can lock the funds of the LSP without
> giving the LSP any opportunity to earn from forwards.
> Both parameters are placed in the `opening_fee_params` object as they
> are part of the service that the LSP is promising.
> Although neither parameter is necessary to compute `opening_fee`, they
> do define what the client is paying for with that fee.

The LSP, when ordering the `opening_fee_params_menu` array, MUST order by
the following rules:

* The 0th item MAY have any parameters.
* Each succeeding item MUST, compared to the previous item, obey any one
  of the following:
  * Have a larger `min_fee_msat`, and equal `proportional`.
  * Have a larger `proportional`, and equal `min_fee_msat`.
  * Have a larger `min_fee_msat`, AND larger `proportional`.

> **Rationale** This simplifies how the client expresses the expected
> opening fee to the user, as it assures the client that the order of
> the fee rates is strictly increasing.

The LSP, when generating the `promise` field:

* SHOULD use a cryptographically-verifiable method of generating
  `promise`, such as a MAC of some deterministic serialization of the
  other `opening_fee_params` keys, then encoded in hexadecimal, base64,
  or other text encoding.
  The specific requirements are:
  * Verifiable - the LSP must be able to verify that it issued the
    `promise` before.
    It does not have to be third-party-verifiable (i.e. it is OK
    if only the LSP can do this verification using private information
    only the LSP knows).
  * Unforgeable - only the LSP can generate the `promise`.
    Knowledge of the `promise` for one set of parameters should not
    allow anyone else to generate the `promise` for a different set
    of parameters.
  * Committing - the `promise` commits to a specific set of parameters;
    if any of the parameters is changed, the `promise` validation would
    fail.
* MAY include additional information in the `promise`, provided that
  additional information does not affect how `opening_fee` would be
  calculated.
* MUST ensure the promise is no longer than 512 bytes (not including the
  `"` delimiters) when encoded as a UTF8-encoded JSON string.
* MUST avoid characters outside of the printable ASCII range.
* MUST avoid characters that would require escapes in JSON format.

> **Rationale** The method to generate `promise` is left vague in order
> to allow LSP implementors the flexibility to structure their code
> as necessary.
> The above specification suggests the use of any MAC, which would cover
> the above requirements, and requires a symmetric key known only by
> the LSP.
> Alternatively, an LSP might be componentized and use private-public
> key cryptography, with a signature as the `promise`, so that the
> component that generates `opening_fee_params` objects is the only
> one with access to a specific private key, while the rest of the LSP,
> including the `lsps2.buy` component, only knows the public key.
> Another alternative for a componentized LSP would be to have the
> `opening_fee_params` generator know the public key of the
> `opening_fee_params` validator; the generator creates an ephemeral
> key pair and uses the ephemeral private key with the validator public key
> in a Diffie-Hellman scheme to create a session key for a MAC, then
> concatenate the ephemeral public key with the MAC as the `promise`; the
> validator takes its private key and uses it with the ephemeral public key
> in a Diffie-Hellman scheme to regenerate the same session key and
> validates the MAC.

Clients, when reading the `promise` field:

* MUST abort this flow if `promise` exceeds 512 bytes (not including the
  `"` delimiters) when encoded as a UTF8-encoded JSON string.
* MUST NOT parse or interpret this string, and only provide it
  verbatim later.

> **Rationale** By using a promise that must be remembered by the
> client, the LSP does not have to remember what it offered via this
> API, reducing state storage, especially if the client then decides
> that the resulting `opening_fee` is too high and ultimately decides
> not to continue this flow.
> By using a cryptographically-verifiable method, the LSP can ensure
> that it will only honour `opening_fee_params` it actually offered.
> A size limit is imposed so that LSPs cannot burden clients with
> unreasonable storage requirements.

LSPs MUST NOT add any other fields to an `opening_fee_params` object.

Clients MUST fail and abort the flow if a `opening_fee_params`
object has unrecognized fields.

> **Rationale** If additional fields are deemed necessary for a
> future version of this specification, then the `version` should
> be increased, and a new schema defined for the later `version`.
>
> Clients that expect this `version` of LSPS4 will compute the
> `opening_fee` in the manner indicated in this specification, and
> any additional fields may imply an extension that may mislead
> the client into computing the wrong `opening_fee`.
>
> If the LSP wants to include other information in the
> `opening_fee_params`, and that information does not affect how
> the `opening_fee` is calculated, then the LSP can actually just
> embed this information in the `promise`; the LSP can also commit
> to that additional information in the MAC or signature or whatever
> cryptographic construct it uses to validate, and the length of
> the `promise` can be used to determine whether this information
> exists or not, and the LSP can cut the added information in this
> `promise` from the cryptographic commitment.

`min_payment_size_msat` and `max_payment_size_msat` are the limits of the
payment size, inclusive.
These are strings containing decimal numbers, in millisatoshis
[<LSPS0.msat>][].
The payment size is the amount that the payer will send to the
client, not including the forwarding fees of nodes along the way.

`client_trusts_lsp` is an *optional* Boolean.
If not specified, it defaults to `false`.
If specified and `true`, the client MUST trust the LSP to actually
create and confirm a valid channel funding transaction.

The client MAY abort the flow if the LSP specified `client_trusts_lsp`
as `true` in the result, and the client does not want to trust the LSP.
If it aborts the flow, the client SHOULD call `lsps4.fail_ppp` on each
PPP of this payment.

The client now takes one of the `opening_fee_params`, and the
`payment_size_msat` (the sum total of the PPPs the client will
use to buy inbound liquidity), to compute the `opening_fee`,
and determine if the resulting `opening_fee` is reasonable.

#### Computing The `opening_fee`

For a given `payment_size_msat` (in millisatoshis) and a selected
`opening_fee_params` object, both client and LSP MUST compute the
`opening_fee` (in millisatoshis) as follows:

    opening_fee = ((payment_size_msat * proportional) + 999999) / 10000000
    if opening_fee < min_fee_msat:
        opening_fee = min_fee_msat

* All numbers MUST be computed in unsigned 64-bit integers.
  * Clients and LSPs MAY use arbitarry-precision integers, such as
    Haskell `Integer` type or bignums in Lisplike languages, IF AND
    ONLY IF they check for overflow of unsigned 64-bit at each basic
    arithmetic operation.
* Integer division MUST round down; the `+ 9999999` causes this to
  round up.
* Arithmetic overflow MUST cause the computation to fail and abort
  this flow (the client SHOULD report the failure to the user, the
  LSP SHOULD error the client request in the next step).
  * Unsigned integer division cannot overflow.
  * Addition overflow can be detected by comparing the sum to an
    addend:
    if the sum is less than one addend, overflow has occurred and
    the flow must be aborted.
  * Multiplication overflow can be detected by checking that the
    first factor is nonzero, and dividing the product by the first
    factor does not result in the second factor.
  * The check MUST be done on EACH addition and multiplication
    operation, including those with constants.

The C routine `compute_opening_fee` below performs the computation,
and indicates an error (returns `-1`) and sets `errno` if the
computation fails, otherwise returns `0` on success.

```C
#include<errno.h>
#include<stdint.h>

static inline int
ov_add (uint64_t *sum, uint64_t a, uint64_t b) {
  *sum = a + b;
  if (*sum < a)
  {
    errno = EDOM;
    return -1;
  }
  return 0;
}
static inline int
ov_mul (uint64_t *prod, uint64_t a, uint64_t b) {
  *prod = a * b;
  if ((a != 0) && ((*prod / a) != b))
  {
    errno = EDOM;
    return -1;
  }
  return 0;
}

int
compute_opening_fee(uint64_t* opening_fee, uint64_t payment_size_msat,
		    uint64_t opening_fee_min_fee_msat,
		    uint64_t opening_fee_proportional) {
  uint64_t tmp;
  if (0 != ov_mul (&tmp, payment_size_msat, opening_fee_proportional))
    return -1;
  if (0 != ov_add (&tmp, tmp, 999999))
    return -1;
  /* C specifies division rounds towards 0.  */
  *opening_fee = tmp / 1000000;
  if (*opening_fee < opening_fee_min_fee_msat)
    *opening_fee = opening_fee_min_fee_msat;
  return 0;
}
```

Below is a similar implementation in Rust, with overflow errors
signalled as `None`:

```rust
fn compute_opening_fee(payment_size_msat: u64,
                       opening_fee_min_fee_msat: u64,
                       opening_fee_proportional: u64) -> Option<u64> {
  payment_size_msat
    .checked_mul(opening_fee_proportional)
    .and_then(|f| f.checked_add(999999))
    .and_then(|f| f.checked_div(1000000))
    .map(|f| std::cmp::max(f, opening_fee_min_fee_msat))
}
```

> **Rationale** The computation is described in excruciating detail
> in order to ensure that both LSP and client agree on the result
> of computation of the `opening_fee`.
> In C, signed integer overflow is undefined behavior, and even if
> neither LSP nor client is written in C, they may be written in a
> language whose implementation is an interpreter written in C or
> using C libraries, which may also trigger the same undefined
> behavior on signed integer overflow, thus unsigned integers are
> specified, which, in C, are specifically modulo 2 to the number
> of bits.
> Arithmetic overflow is problematic and may lead to unexpectedly
> low `opening_fee` charges for high `payment_size_msat` and high
> `opening_fee_params.proportional`, thus must be specifically
> detected and cause an abort in this flow rather than continue
> and cause problems for the LSP.
> Most modern microprocessors have dedicated instructions to detect
> overflow, and most modern compilers can recognize the above code
> and optimize those down to these dedicated instructions at high
> optimization settings; in particular, the division in the
> overflow-detecting multiplication routine is optimized away
> and replaced with a simple overflow-flag check.

### 6. Buying Inbound Liquidity

The client selects a set of PPPs it wants to have forwarded
to it.

The total amounts of the PPPs would now be the `payment_size`.
The client then computes the `opening_fee`.

A portion of the PPPs will be used to pay the `opening_fee`,
while the total of the PPPs (i.e. the `payment_size_msat`)
determines how large the `opening_fee` would be.
The larger the total amount of the PPPs selected, the higher
the `opening_fee` due to the proportional part.

The client MUST check if the `opening_fee` is strictly less
than `payment_size - number_of_ppps * client_htlc_minimum_msat`,
where `number_of_ppps` is the number of PPPs in the set and
`client_htlc_minimum_msat` is the corresponding setting in the
`opening_fee_params`.

The client MUST check that `client_htlc_minimum_msat` is strictly
less than or equal to the smallest PPP amount in the selected
PPP set.

Once the client has selected the set of PPPs it will use to
pay for the channel open, the client calls `lsps4.open_with_ppps`
call, which accepts the parameters:

```JSON
{
  "ppp_ids": ["id#abcdefghijklmnop"],
  "opening_fee_params": {
    "min_fee_msat": "546000",
    "proportional": 1000,
    "client_htlc_minimum_msat": 100,
    "valid_until": "2023-04-19T16:55:21.295Z",
    "min_lifetime": 1008,
    "max_client_to_self_delay": 1008,
    "promise": "abcdefghijklmnopqrstuvwxyz+/"
  }
}
```

All parameters are required.

`ppp_ids` is a set of PPP string identifiers that the client wants
to use to open the channel.
It MUST NOT be empty.

`opening_fee_params` is an object selected from the result of
the `lsps4.get_opening_fee` call, copied verbatim.

The entire `lsps4.open_with_ppps` call MUST be atomic, including
the checks below.

The LSP, atomically:

* MUST check that there is currently no pending LSPS4-bought
  channel currently being opened (i.e. has not completed up to
  `channel_ready` from both parties).
  * This is done by checking for the existence of a
    "PPP Opening Set", described later.
* MUST check that the given `opening_fee_params.promise` validates
  the given `opening_fee_params`, and that
  `opening_fee_params.valid_until` is a future datetime.
* MUST check that all the PPPs listed in `ppp_ids` are still in the
  PPP Map.
* MUST sum up the amounts of all PPPs listed and set `payment_size`
  to that sum.
* MUST compute the `opening_fee` based on the `payment_size` and
  the given `opening_fee_params`, and MUST check that the
  `opening_fee` is strictly less than
  `payment_size - number_of_ppps * client_htlc_minimum_msat`.

If all above checks pass, the call succeeds.
Before responding, the LSP, still in the same atomic section:

* Removes the PPPs listed in `ppp_ids` from the PPP Map and puts
  them into a PPP Opening Set.
  * The existence of the PPP Opening Set is what is checked to
    determine if a pending LSPS4-bought channel is being opened.

Once the above operations complete, the call succeeds and the
atomic section ends, and the LSP responds with an empty object
`{}` and proceeds to the next step.

If the checks fail, the atomic section ends and the LSP responds
with an error code:

* `nonexistent_ppp_id` (2) - One or more of the `ppp_ids` is no longer in
  the PPP Map.
  The error `data` object contains the required field
  `unknown_ppps`, which is an array of PPP identifiers
  containing the PPPs which were not found.
* `invalid_opening_fee_params` (3) - The `opening_fee_params.promise`
  failed validation, or the `opening_fee_params.valid_until` is
  already a past datetime.
* `operation_pending` (4) - An LSPS4-bought channel is not yet
  completed opening and sending the PPPs being used to pay for
  it.
  * The client SHOULD retry later once the channel has opened and
    the existing payments resolved.
* `insufficient_funds` (5) - The total `payment_size` is not
  enough to pay for the `opening_fee`.

> **Non-normative** The intent is that once asynchronous receives
> are possible, asyncpay-type PPPs will NOT be accepted in the
> `lsps4.open_with_ppps` call.
> Instead, those PPPs can only be passed to `lsps4.forward_ppps`
> call, which will cause the LSP to respond to the senders of those
> asynchronous receives.
> The senders would then deliver actual HTLCs/PTLCs that the LSP
> will convert again to PPPs.
>
> This will help with the multiple-small-separate-payments
> case; asyncpay-type PPPs are expected to be holdable by the
> LSP longer, and once enough asyncpay-type PPPs are available,
> the client can then request the senders to try sending the
> actual payments in the hope that the LSP can gather enough
> to pay for a new channel.

### 7. Opening Channel And Forwarding Selected Pending Payment Parts

The LSP requests a channel open to the client via standard
[BOLT2 Channel Establishment][] flow.

The LSP selects the channel size.
The LSP MUST ensure the channel is large enough to transmit the
`payment_size_msat - opening_fee`, plus any reserve needed on the LSP
side.

The LSP and client MUST negotiate these options and parameters in
their [BOLT2 Channel Establishment][] flow:

* `option_zeroconf` is set.
* `announce_channel` is `false`.
* The client-provided `to_self_delay` is less than or equal to
  the agreed-upon `max_client_to_self_delay`.
* The client-provided `htlc_minimum_msat` is equal to the
  `client_htlc_minimum_msat` specified in the selected
  `opening_fee_params`.

> **Rationale** `option_zeroconf` is set so that the payment can
> be immediately forwarded to the client and resolved as soon as
> the opening negotiation completes; if the client and LSP had to
> wait for confirmation, the incoming payment would have to be
> locked for an extended period, which is indistinguishable from
> a channel jamming attack.
> `option_zeroconf` cannot be set unless `announce_channel` is
> `false`.

Clients MAY reject the channel open attempt if any other
parameters or options are not desired by the client (for example,
it MAY reject non-zero channel reserve requirements imposed on
the client by the LSP).
Rejection is via the `error` Lightning BOLT message.

* If the client disconnects before the LSP can receive
  `fuding_signed` from the client, OR if the client sends an
  `error` before it sends `channel_ready` for the channel, the
  LSP atomically:
  * Removes the PPP Opening Set and returns the contained PPPs
    to the normal PPP Map, under new PPP IDs, but with the same
    hold timeouts.
    * After the atomic section, this will cause the LSP to issue
      new `lsps4.you_have_new_ppp` with the new PPP IDs, unless
      they have already timed out.
    * After the atomic section, the LSP can handle timed out
      PPPs as specified in a previous section.

> **Rationale** The client might have crashed at this point, and
> recovery might take an inordinate amount of time.
>
> Client disconnection before the LSP can receive `funding_signed`
> will prevent the LSP from broadcasting the funding transaction
> and the LSP may re-allocate the UTXOs spent in the funding
> transaction for a different payment.
> Thus, the correct behavior is to abort the payment, so that the
> LSP funds may be re-used in a new channel to the same client if
> the payer retries, or so that the LSP funds may be re-used for a
> different client if the payer gives up on paying the client.
>
> The PPPs are returned to the PPP Map to allow the client to
> re-handle them via other means, including issuing an error at
> the client.

The LSP MAY `error` the channel before it broadcasts the funding
transaction and sends `channel_ready`.
If it does so, it MUST attempt to re-initiate a new channel open
with the same client before continuing with this flow.
The LSP MAY repeat this channel open-`error`-repeat flow any number
of times.

The client MUST NOT treat an `error` of a channel open as a
failure of the LSP to deliver the requested service, as long as
the LSP is able to re-initiate the channel open in a timely manner.

> **Rationale** The above allows the LSP to batch multiple channel
> opens with the [BOLT2 Channel Establishment][] protocol.
> When coordinating a single batched funding transaction with
> multiple peers, one of the peers may fail, reject, or abort the
> channel opening flow, which requires that the LSP abort the
> channel opening flow for the remaining peers (as there is only a
> single batched transaction).
> This is done by `error`ing the channel before broadcasting the
> funding transaction, then re-starting the channel open flow
> with the failing peer removed from the batched channel funding
> transaction.
> Thus, other clients will see their channel opens `error`ed
> if one of the clients in the same batch aborts the flow, but as
> long as the LSP attempts a re-open before continuing with
> payment, this is not a funds loss event, and neither the LSP nor
> the client is obligated to pay onchain fees in this case, only
> spend CPU processing time and network bandwidth on the additional
> negotiations.

Depending on the `client_trusts_lsp` from the `lsps4.get_opening_fee`
result, after the client has sent `funding_signed` and the LSP is
willing to broadcast the funding transaction:

* If `client_trusts_lsp` is specified and `true`:
  * The LSP MAY wait for the client to send the preimages to the
    incoming HTLCs (or equivalent for other PPP types) via an
    `update_fulfill_htlc` before broadcasting the funding
    transaction.
  * The client MUST NOT wait for the LSP to broadcast the funding
    transaction before sending the preimage via a
    `update_fulfill_htlc`.
  * The client and LSP MUST immediately send `channel_ready`.
* If `client_trusts_lsp` is unspecified or `false`:
  * The client MAY wait for the funding transaction to appear in
    its mempool or the mempool of a trusted node, or confirmed in a
    block with any depth, AND confirm that the funding transaction
    output has the correct amount and `scriptPubKey`, before sending
   `channel_ready`.
  * The LSP MUST immediately broadcast the channel funding
    transaction and send `channel_ready`.
  * The client MAY immediately send `channel_ready` (i.e. the
    client trusts the LSP, even though the LSP does not require
    that trust).

> **Rationale** As mentioned in the section "Trust Models" above,
> in LSP trusts client model, the client is allowed to choose its
> risk appetite; it may accept 0 confirmations, 1, 3, 6, 100, or
> however many it believes would be safe with the particular LSP.

As soon as both client and LSP have sent `channel_ready` for the
new channel, the LSP then forwards PPPs in the PPP Opening Set to
the client.

The LSP MUST NOT forward other HTLCs to the new channel until
the PPP Opening Set is empty.

The LSP generates non-standard forwards, where the amount received
by the client is smaller than specified in the onion;
the client MUST accept the non-standard forward(s), provided they
sum to at least `payment_size_msat - opening_fee`.

The LSP MUST ensure that each forwarded part forwards at least
`client_htlc_minimum_msat` millisatoshis.
The LSP MUST ensure that all parts forwarded sum up to at least
`payment_size_msat - opening_fee`.

The LSP MUST include an `extra_fee` (type 65536) TLV to the
`update_add_htlc` message of forwarded parts that have had
fees deducted.
The LSP MUST NOT include this `extra_fee` TLV if the part does
not have fees deducted (for example, if this is a multipart
payment and the other parts have already paid for the fee).
The `extra_fee` TLV MUST have a length of 8, and the value is a
big-endian 64-bit number containing the number of millisatoshis
deducted from this part.

The client MUST check that the sum of all `extra_fee`s is less
than or equal to the `opening_fee`, and that the actual amount of
each HTLC is equal to the expected amount in the onion minus the
individual `extra_fee`.

`extra_fee` is currently proposed in [bLIP-0025][].

[bLIP-0025]: https://github.com/lightning/blips/pull/25

The LSP MUST also include a `via_scid` (type 65535) TLV to the
`update_add_htlc` message of all forwarded parts.
`via_scid` has type 65535 and length 8, and its value is the
binary encoding of the SCID seen by the LSP, which matches some
`lsps4_scid` the LSP reserved for the client.

Once the corresponding HTLC has been sent by the LSP and claimed
by the client, the LSP removes the corresponding PPP from the
PPP Opening Set.

Once the PPP Opening Set is empty, the LSP removes the set,
and the `lsps4.open_with_ppps` API can be invoked again.

### 8.  Post-opening Normal Operation

The client MUST NOT use the `alias` SCID sent in the 0-conf
channel open (unless it is equal to the permanent SCID, which
is not assured), or the "real" SCID of the channel once
confirmed.
The client MUST use only the SCIDs issued via `lsps4.reserve_scid`
for all invoices and offers it creates.

The LSP SHOULD fail an incoming HTLC if it uses the `alias` or
"real" SCID of the channel as the next hop, with the failure
`unknown_next_peer`, if the `alias` is not the same as the
permanent SCID.

> **Rationale** Some node software implementations do not have
> an API to set the `alias` of the channel.
> The permanent SCID, in any case, represents all the channels
> with the client, not any specific channel.
> It also represents a possible future channel if there is no
> actual channel between the LSP and the client.

If the LSP supports onion messages, it MUST forward any onion
messages that indicate any currently-valid SCID (permanent
or ephemeral) returned from an `lsps4.reserve_scid` call, to
the client that requested that SCID.

Clients that want to receive onion messages SHOULD use an
SCID requested via `lsps4.reserve_scid`.

#### Lightning Forwarding Feerate Change

The LSP MAY change the in-Lightning forwarding fee for
permanent SCIDs at any time.
If so, the LSP MUST inform the client via the
`lsps4.i_changed_permanent_fee_rates` notification, whose
`params` are below (this is the same as the `result` of
`lsps4.reserve_scid`):

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

The LSP MAY change the `ln_base_fee_msat`, `ln_prop_fee`,
or `cltv_expiry_delta`, but MUST NOT change `lsps4_scid`,
`expiry_time`, and `min_final_cltv_expiry_delta`.

The LSP MAY send this notification if and only if the client
has requested a `"permanent"` SCID via `lsps4.reserve_scid`.
The LSP MUST NOT send this notification if the client has not
requested a permanent SCID, or if the permanent SCID has been
expired due to both passing the expiry time and having no more
channels with the client.

All the above fields are required.

`lsps4_scid` is the permanent SCID [<LSPS0.scid>][].
`expiry_time` is the expiry time for the SCID
[<LSPS0.datetime>][].
`min_final_cltv_expiry_delta` is a number of blocks, the same
as the original number of blocks the client used when the
permanent SCID was first called.

`ln_base_fee_msat` is the base fee that the client MUST indicate
as the base fee for the given permanent SCID [<LSPS0.msat>][].
`ln_prop_fee` is the proportional fee the client MUST indicate as
the proportional / parts-per-million fee for the given permanent
SCID [<LSPS0.ppm>][].
`cltv_expiry_delta` is the CLTV difference between the LSP-inbound
timelock and the to-client timelock.

If the LSP ever sends this notification, the LSP MUST re-send
this notification whenever the client reconnects, after it has
called `lsps4.select_version` with a version that supports this
notification, for all future reconnections.
This notification is valid for API versions: 1.

The LSP MAY send this notification on reconnection, after
`lsps4.select_version`, even if it has not changed the
settings.

If the LSP does not ever send this notification (i.e. the LSP
does not ever change the fee rates) then the LSP does not need
to send this notification on reconnection.

> **Rationale** Notifications are never acknowledged by the
> client, and the LSP may have sent a message containing the
> notification but the client may not have received it.
> Thus, if the LSP ever changes the settings of the SCID,
> the client may not be aware of it in case of unreliable
> connection.
>
> An LSP that has a policy of fixed fee rates to all clients
> at all times need not ever send this notification and can
> ignore its existence in this specification.

If an LSP changes the settings, the LSP MUST provide a grace
period of 1 hour, from the first time the LSP delivers this
notification.
During this grace period, the LSP MUST allow use of either
the previous or the new settings.

> **Rationale** Even outside of this specification, the fee
> rates and CLTV-delta settings of a published channel can
> take a significant amount of time to propagate via gossip
> to the rest of the Lightning Network.
> Thus, a node can issue an invoice using current fee rates
> and CLTV-delta of a published channel that has high
> incoming capacity, then the channel counterparty can
> change those settings, making the recently-issued invoice
> incorrect.
>
> Lightning node software generally have a similar grace
> period during which they accept the previous or the new
> settings for channels; thus, a similar behavior is also
> specified here.

[<LSPS0.binary_blob>]: ../LSPS0/common-schemas.md#link-lsps0binary_blob
[<LSPS0.datetime>]: ../LSPS0/common-schemas.md#link-lsps0datetime
[<LSPS0.scid>]: ../LSPS0/common-schemas.md#link-lsps0scid
[<LSPS0.msat>]: ../LSPS0/common-schemas.md#link-lsps0msat
[<LSPS0.ppm>]: ../LSPS0/common-schemas.md#link-lsps0ppm
[BOLT2 Channel Establishment]: https://github.com/lightning/bolts/blob/c4c5a8e5fb30b1b99fa5bb0aba7d0b6b4c831ee5/02-peer-protocol.md#channel-establishment

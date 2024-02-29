# LSPS3: Promise To Unconditionally Fund 0-conf

| Name          | `promise_to_0conf_fund`    |
| ------------- | -------------------------- |
| Version       | N / A                      |
| Status        | For Implementation         |

> **Rationale** Unlike many other LSPS specifications, this protocol does not
> have any versioning support.
> TXIDs are deeply ingrained in the consensus rules of the blockchain layer,
> and `scriptPubKey`s are generic enough that even future address types
> would be expressible in some `scriptPubKey` template.
> Thus, even if the Lightning Network evolves in unforeseen ways, committing
> to a TXID and a `scriptPubKey`, and promising to get the TXID confirmed
> on or before some future block height, should still be sufficient for the
> client to rely on 0-conf transfers.
>
> As a concrete example, suppose that in the future, Lightning Network
> node software supports channel factories.
> When a channel factory is first constructed, any transfers of any channels
> inside it are 0-conf until the channel factory funding transaction is
> confirmed.
> The transaction creating the channel factory will have a fixed TXID, and 
> the output that backs the entire factory will have some known `scriptPubKey`.
> The client and the LSP can simply use the same LSPS3 protocol on the 0-conf
> channel factory construction.
>
> By dropping versioning, the expectation is that this improves participation
> in the boycotting protocol; if multiple versions existed, then there would
> be some participants running on older versions that would be unable to boycott
> LSPs that were proven to be misbehaving using the newer version.

## Motivation

0-conf channels are inherently trust-requiring; without confirmation,
any HTLC hosted in the channel is ephemeral and potentially can be
clawed back without recourse to the HTLC acceptor.

To reduce the trust requirement, we can have the channel opener commit a
promise to ensure that the channel is actually opened before some future
blockheight.

This scheme only works for unconditional 0-conf fundings.
For a channel open, the term 'unconditional' means:

* The LSP completes the `open_channel` / `accept_channel` /
  `funding_created` / `funding_signed` flow and **immediately**
  broadcasts the funding transaction **without** waiting for any
  other event or message from the client.
  * The broadcast follows as soon as it can receive `funding_signed`.
* The LSP commits to getting the funding transaction confirmed before
  some future target blockheight **without** waiting for any
  other event or message from the client.

For example, a JIT Channel (LSPS2) open is unconditional if the
LSP takes on the risk of broadcasting the transaction and immediately
forwarding the HTLC to the client.
A misbehaving client may then wait for the transaction to confirm
deeply before eventually failing the HTLC.
If the LSP is willing to take on this risk, then this scheme may be
used to reduce the trust the client has in the LSP.

> **Rationale** This specification was created during development of
> LSPS2 JIT Channels.
> In particular, prior to this specification, trust models for JIT
> Channels fell into two classifications:
>
> * Client-trusts-LSP: the client provides the preimage before it
>   knows that the LSP has broadcast the channel funding transaction.
>   The LSP defers channel funding transaction broadcast until after
>   the client has provided the preimage in a `update_fulfill_htlc`.
>   The LSP has the ability to not broadcast the channel funding
>   transaction at all, while still able to take incoming funds since
>   the preimage was received from `update_fulfill_htlc`.
> * LSP-trusts-client-trusts-LSP: the LSP immediately broadcasts the
>   funding transaction for a 0-conf channel.
>   The client defers releasing the preimage in an `update_fulfill_htlc`
>   until after it has seen the funding transaction in its local mempool.
>   The client has the ability to never release the preimage, thus
>   forcing the LSP to lcok its funds to a channel that is not paid
>   for, for up to the client-specified `to_self_delay` blocks.
>   However, the LSP also has the ability to claw back the channel open
>   by use of an RBF (such as by using a full-RBF fullnode connected to
>   full-RBF miners) transaction that replaces the funding transaction,
>   or by otherwise bribing miners.
>
> Trust in the LSP is undesirable as it represents risk for the LSP.
> The LSP may be falsely accused of misbehavior, and disproof of that
> misbehavior cannot be provided by the LSP in its defense.
>
> This specification reduces client trust in the LSP, by providing a
> definite proof of misbehavior.
> Absence of that proof of misbehavior is then proof of absence of
> misbehavior (i.e. proof of correct behavior), which reduces the
> risk on the LSP: false accusations cannot provide the proof of
> misbehavior and are thus shown as false.
>
> The LSP can only utilize this API to provide absence of proof of
> misbehavior, in the LSPS2 JIT Channels case, if and only if it
> performs the channel open unconditionally, as defined above
> (i.e. it immediately broadcasts the funding transaction and
> commits to getting the funding transaction confirmed before
> some future date).
> This API specification cannot be used if the LSP wants to wait
> for some event, such as a preimage release, from the client.
>
> Assuming that a sufficient number of network participants
> actually boycott LSPs that violate any promises they made under
> this specification, then an LSPS2 JIT Channel open is truly
> LSP-trusts-client, with the client not needing to trust the
> LSP.
>
> While the client can now defraud the LSP under this model,
> the expectation is that an LSPS2 JIT Channel `opening_fee`
> is smaller than the client payment, thus the loss experienced
> is expected to be smaller.

This specification, in particular, also includes a punishment
mechanism for LSPs that fail to keep their promise to have a
particular funding transaction confirm:

* Clients can report the misbehavior (specifically, failure to
  get a particular funding transaction confirmed before the
  blockheight) to a competitor LSP.
* Competitor LSP closes all channels to the offending LSP and
  prevents all future channels with that LSP, reducing the
  ability of the offending LSP to continue its business (i.e.
  boycott).
* Competitor LSP puts the offending LSP in a blacklist.
* Competitor LSP allows clients to download blacklisted LSPs
  together with proofs of misbehavior, so that clients can
  avoid the misbehaving LSP (i.e. boycott).

### Funding of Other Protocols

While originally designed to promise that 0-conf channel opens
will be confirmed later, this specification is generic enough to
be useable for other cases where something needs to be funded,
and to have the LSP promise that a 0-conf funding will indeed
be confirmed, so that the client can rely on funds that derive
from the 0-conf funding.

## Prerequisitse

The client:

* MUST be able to determine the current blockheight.
* MUST be able to do one of the following:
  * determine if a transaction of specific transaction ID has confirmed
    or not, on or before a specific blockheight.
  * OR determine if a specific `scriptPubKey` is on an output of some
    transaction that has been confirmed, on or before a specific blockheight.
* MUST be able to validate signatures generated using the
  semi-standard [LND `SignMessage` RPC][] / [CLN `signmessage` RPC][] /
  [Eclair `signmessage` RPC][], as per [`signmessage` #specinatweet][].
* MUST maintain a blacklist of LSP node IDs it must not use as an LSP.

[LND `SignMessage` RPC]: https://lightning.engineering/api-docs/api/lnd/lightning/sign-message
[CLN `signmessage` RPC]: https://lightning.readthedocs.io/lightning-signmessage.7.html
[Eclair `signmessage` RPC]: https://acinq.github.io/eclair/#signmessage
[`signmessage` #specinatweet]: https://twitter.com/rusty_twit/status/1182102005914800128

The first two of the above prerequisites are prerequisites for correct
behavior of Lightning Network nodes, and are expected to be available
even for SPV-only nodes.
For instance (**Non-normative**) the client may be an SPV node that
maintains the block header chain, and can query some server for a
proof-of-transaction-confirmation where the server provides the
Merkle proof of the existence of a particular transaction ID in a
particular block in that block header chain.

The second prerequisite above may be implemented via `scriptPubKey`
instead, for example (**Non-normative**) via BIP158, which specifies
block filters that do not include transaction IDs, but do include
`scriptPubKey`s.

The third prerequisite above should be doable with most common node
implementations, and should be implementable based on the open-source
code of those node implementations.

The LSP:

* MUST be able to determine the current blockheight.
* MUST be able to determine if a transaction of specific transaction
  ID has confirmed or not, on or before a specific blockheight.
* MUST be able to generate signatures and validate said signatures
  using the semi-standard [LND `SignMessage` RPC][] /
  [CLN `signmessage` RPC][] / [Eclair `signmessage` RPC][], as per
  [`signmessage` #specinatweet][].
* MUST be able to check Lightning Network gossip.
* MUST maintain a blacklist of competitor LSPs as well as one proof of
  misbehavior of each blacklisted competitor.
  * MUST be able to ensure that it never has channels with a blacklisted
    competitor LSP.
* SHOULD ensure that transactions for unconditional 0-conf fundings have
  a change output under unilateral control of the LSP, so that it can
  CPFP the funding transaction before any promised target future
  blockheight.
  * As the LSP commits to a specific transaction ID, it MUST NOT use
    RBF on the funding transaction; it MAY use RBF on a CPFP child
    transaction.

## Requesting Promise To Ensure 0-conf Channel Open

When an LSP opens a 0-conf channel unconditionally to the client, such
as in an LSPS2 JIT Channels flow, the client SHOULD make the
`lsps3.getpromise` API call.

When using this specification for a 0-conf channel funding, the client MUST
NOT call this API until after it has sent `funding_signed` to the LSP.

The client SHOULD NOT provide any preimages to HTLCs it receives on
the unconditional 0-conf channel until after this call successfully returns.

The parameters of `lsps3.getpromise` are:

```JSON
{
  "funding_txid": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "funding_outnum": 1,
  "funding_scriptpubkey": "a91485b9ff0dcb34cf513d6412c6cf4d76d9dc24013787"
  "funding_amount": "546000"
}
```

`funding_txid` is the transaction ID of the funding transaction.
`funding_outnum` is the output index of the funding output on the funding
transaction.
`funding_scriptpubkey` is the expected `scriptPubKey` of that output index.
`funding_amount` is the expected amount of that output index, in a string
containing a decimal number of millisatoshis.
As the amount refers to an onchain amount, it MUST be divisible by 1000.

When this specification is used for a 0-conf channel funding:

* `funding_txid` is the `funding_txid` received from the `funding_created`
  message.
* `funding_outnum` is the `funding_output_index` received from the
  `funding_created` message.
* `funding_scriptpubkey` is the `scriptPubKey` of the channel being opened,
  as per [BOLT3 Funding Transaction Output][].
* `funding_amount` is the `funding_satoshis` from the `open_channel`
  message, converted to millisatoshis.

[BOLT3 Funding Transaction Output]: https://github.com/lightning/bolts/blob/master/03-transactions.md#funding-transaction-output

All parameters are required.

The LSP:

* MUST check that it created and signed the `funding_txid`, and that this
  transaction is indeed for funding something it is willing to promise
  to unconditionally 0-conf fund.
* MUST check that the specified `funding_outnum` output has the specified
  `funding_scriptpubkey` and is of the amount `funding_amount`.

The LSP then determines a `target_blockheight` for the funding transaction.
This is a future blockheight which the LSP absolutely commits to ensuring
that the funding transaction has been confirmed.
The LSP is liable for punishment under this specification if it fails to
have that specific funding transaction ID confirmed before
`target_blockheight`.

The LSP MUST select a `target_blockheight` that is less than or equal to
the current blockheight, plus 2016.

> **Rationale** The BOLT2 spec has a "SHOULD" requirement that the
> channel opener should ensure the funding transaction confirms within
> 2016 blocks.

The LSP MUST persistently store its earliest selected `target_blockheight`
for a particular `funding_txid` **before** responding to this request.
The LSP MUST ensure that the `funding_txid` confirms on or before the
`target_blockheight`.

The LSP then signs the following JSON string message:

    "LSPS3: DO NOT MANUALLY SIGN THIS MESSAGE: I promise to confirm ${funding_txid} whose output ${funding_outnum} has ${funding_scriptpubkey} of ${funding_amount} on or before ${target_blockheight} on pain of boycott"

Where:

* `${funding_txid}` is the unquoted **lowercase** hex string form of the
  `funding_txid`.
* `${funding_outnum}` is the unquoted decimal string form of `funding_outnum`.
* `${funding_scriptpubkey}` is the unquoted **lowercase** hex string form of
  the `funding_scriptpubkey`.
* `${funding_amount}` is the unquoted decimal string form of `funding_amount`.
* `${target_blockheight}` is the unquoted decimal string form of
  `target_blockheight`.

> **Rationale** [CVE-2019-12998][], [CVE-2019-12999][], and
> [CVE-2019-13000][] exist due to specifications-level bugs
> where the specification did not explicitly mention checking
> that the specified transaction actually has an output that
> matches the expected `scriptPubKey` and amount.
> To avoid a similar problem, the LSP must commit to these
> details, as otherwise the LSP can perform a similar attack
> on promised 0-conf channels it opens.

[CVE-2019-12998]: https://nvd.nist.gov/vuln/detail/CVE-2019-12998
[CVE-2019-12999]: https://nvd.nist.gov/vuln/detail/CVE-2019-12999
[CVE-2019-13000]: https://nvd.nist.gov/vuln/detail/CVE-2019-13000

On success, the LSP returns the result:

```JSON
{
  "target_blockheight": 500000,
  "zbase": "blah"
}
```

`target_blockheight` is the blockheight at which the LSP promises the
funding transaction to already be confirmed.
`zbase` is the zbase-formatted signature, as specified in
[`signmessage` #specinatweet][].

The client MUST check that `target_blockheight` is less than or equal to
the current blockheight, plus 2016.
In case it is greater than current blockheight plus 2016:

* If the `target_blockheight - (current_blockheight + 2016)` is "small",
  SHOULD just wait until `current_blockheight + 2016 == target_blockheight`
  before continuing.
  * The definition of "small" is left up to the client; the client can
    simply not implement this branch and always abort, at the risk that
    the client turns out to simply be a few blocks out-of-date compared to
    the rest of the network and it could have gotten the correct blockheight
    if it only waited for a few more seconds.
* Otherwise, MUST abort any flows involving the funding.
  * For example, if this funds a 0-conf channel open, it should `error`
    that channel and forget it.

The client MUST validate that the `zbase` is indeed a valid signing of
the above specified JSON string message, with the LSP node ID as the
public key.

The client SHOULD store the results of this call in persistent storage
together with the LSP node ID, `funding_txid`, `funding_outnum`,
`funding_scriptpubkey`, and `funding_amount`.

Once the client has persistently stored the results of this call, it
can continue with the flow.
For funding a 0-conf channel open, the client MUST send `channel_ready`
after it has persisted the results of this call.

Possible errors are:

* `unknown_funding` (1) - The LSP does not recognize the given
  transaction ID as a funding transaction for anything it wishes to
  promise, or the `funding_outnum`, `funding_scriptpubkey`, or
  `funding_amount` do not match the transaction.
  * For a 0-conf channel open, the client SHOULD forget the channel
    and ignore any incoming HTLCs on the channel.
* `conditions_apply` (2) - The LSP is waiting for some condition from the
  client before it actually commmits to broadcasting the funding transaction,
  i.e. it recognizes the transaction ID but it is not willing to
  unconditionally fund it.
  * The client fulfilling that condition is unlikely to be atomic with
    the LSP being able to provide this promise, thus the client MUST
    trust the LSP to complete the open even after it fulfills the
    condition (i.e. it cannot rely on this call succeeding after it
    fulfills the condition).
    This specification does not work in case of a conditional 0-conf
    channel open, even after the condition is fulfilled; a misbehaving
    LSP can always fail or outright ignore this call even after the
    condition is fulfilled.

The client MUST monitor that the `funding_txid` confirms on or before
the `target_blockheight`.

Once the client has seen the transaction with the `funding_txid`, it
MUST check that the `funding_outnum` exists and has the specified
`funding_scriptpubkey` and `funding_amount`.
The client MAY defer this check; for example, for a channel funding,
it may check at closing time if a supposedly-vslid closing transaction
is accepted into mempools, as rejection of the closing transaction
when `funding_txid` was confirmed implies the `funding_outnum` does
not exist, or does not have the expected `funding_scriptpubkey` or
`funding_amount`.

> **Rationale** A simple client implementation need not implement a
> full Bitcoin transaction parser if the above check is acceptable
> to be deferred; the check can be done at channel closing time,
> if the client can detect if mempools accept their transaction or
> not.
> If the transaction with `funding_txid` does not match the
> expected output `scriptPubKey` or amount, but is confirmed, then
> the misbehaviour has already been recorded and can be reported
> and checked at any later time.
>
> An alternative implementation would be, instead of having the
> promise commit to a specific TXID output, `scriptPubKey`, and
> amount, is to commit only the TXID and return the actual signed
> transaction to the client.
> However, this would require that the client validate the
> transaction output has the expected `scriptPubKey` and amount,
> and require the client to implement a transaction parser.

### Example Flow With Unconditional 0-conf Channel Open

                            Client                           LSP
                               |                              | Plans funding_amount
                               |                              |
         Learns funding_amount |<---------open_channel--------|
                               |                              |
    Plans funding_scriptpubkey |                              |
                               |                              |
                               |---------accept_channel------>| Learns funding_scriptpubkey
                               |                              |
                               |                              | Plans funding_outnum
                               |                              | Plans funding_txid
                               |                              |
         Learns funding_outnum |<-------funding_created-------|
           Learns funding_txid |                              |
                               |                              |
       Signs 1st commitment tx |                              |
                               |                              |
                               |--------funding_signed------->| Learns signed 1st commitment tx
                               |                              |
                               |<--------channel_ready--------|
                               |                              |
                               |-----req lsps3.getpromise---->|
                               |                              |
                               |                              | Computes zbase
                               |                              |
                  Learns zbase |<----lsps3.getpromise rsp-----|
                               |                              |
               Validates zbase |                              |
              Persists promise |                              |
                               |                              |
                               |---------channel_ready------->|
              Normal operation |                              | Normal operation

## Reporting Proof of Misbehavior

If a client determines that the `funding_txid` failed to confirm on or
before the `target_blockheight`, it can and SHOULD report the LSP to
**other** LSPs that support LSPS0 and LSPS3, once the blockheight is
`target_blockheight + 100`.

> **Rationale** The 100-block requirement is to protect against
> blockchain reorganizations, and is taken from the 100-block
> coinbase maturation requirement of Bitcoin.

The client SHOULD connect to some other LSP that supports LSPS0 and
LSPS3, and use the `lsps3.uploadpom` API, with parameters:

```JSON
{
  "lsp_node_id": "023456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012",
  "funding_txid": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "funding_outnum": 1
  "funding_scriptpubkey": "a91485b9ff0dcb34cf513d6412c6cf4d76d9dc24013787",
  "funding_amount": "546000"
  "target_blockheight": 500000,
  "zbase": "blah"
}
```

All parameters above are required.

`lsp_node_id` is the node ID of the LSP that failed its promise to
confirm the funding transaction ID.
`funding_txid` is the transaction ID that failed to be confirmed, or
does not have an output that matches the expected `scriptPubKey` or
amount.
`funding_outnum` is the output index, which should have the
gven `funding_scriptpubkey` and the given `funding_amount`.
`target_blockheight` is the blockheight at which the offending LSP
had promised to ensure confirmation.
`zbase` is the zbase-formatted signature, as specified in
[`signmessage` #specinatweet][].

The `zbase` signature signs the following JSON string message:

    "LSPS3: DO NOT MANUALLY SIGN THIS MESSAGE: I promise to confirm ${funding_txid} whose output ${funding_outnum} has ${funding_scriptpubkey} of ${funding_amount} on or before ${target_blockheight} on pain of boycott"

Where:

* `${funding_txid}` is the unquoted **lowercase** hex string form of the
  `funding_txid`.
* `${funding_outnum}` is the unquoted decimal string form of `funding_outnum`.
* `${funding_scriptpubkey}` is the unquoted **lowercase** hex string form of
  the `funding_scriptpubkey`.
* `${funding_amount}` is the unquoted decimal string form of `funding_amount`.
* `${target_blockheight}` is the unquoted decimal string form of
  `target_blockheight`.

On receiving this API request, the LSP:

* MUST check if the `lsp_node_id` is already in its blacklist, and if
  so, must succeed this API request immediately without the further
  checks below.
* MUST check that the `lsp_node_id` is not itself.
* MUST check that the current blockheight is at least
  `target_blockheight + 100`.
* MUST check that the `zbase` signs the above message, using the
  `lsp_node_id` as public key.
* MUST check that the given `lsp_node_id` is in fact in the gossip
  map.
* MUST check if at least one of these is true:
  * The specified `funding_txid` did not confirm at or below
    `target_blockheight`.
  * The specified `funding_txid` confirmed, but does not have an
    output indexed at `funding_outnum`.
  * The specified `funding_txid` confirmed and has an output at
    `funding_outnum`, but that output does not match the given
    `funding_scriptpubkey` or `funding_amount`.

> **Rationale** It is pointless to report the LSP to itself, as the
> misbehaving LSP can simply claim success of this API and then never
> actually add itself to its own blacklist.
> Other LSPs, however, have a good incentive to remove competitors that
> have been caught by this mechanism and can therefore encourage
> boycott of offending LSPs by propagating proofs of misbehavior of
> such LSPs.
>
> The offending `lsp_node_id` should be in the gossip
> map, to ensure that LSPs cannot be put into a DoS situation by being
> fed randomly-generated `lsp_node_id`s and their signatures in an
> attempt to overload their blacklist database and thereby prevent
> practical use of boycotting (i.e. forgery of phantom LSPs).
> If the offending `lsp_node_id` has already dropped off the gossip
> map due to having all its published channels closed, then the
> offending LSP has already been successfully punished and can no
> longer victimize future clients.
> If the check did not exist, it would be trivial for clients to
> overload the LSP blacklist database by generating random node
> IDs for non-existent LSPs.

On failure, one of the following errors may be returned:

* `shall_not_witness_against_myself` (1) - The `lsp_node_id` is this LSP.
  <!-- aka `i_plead_the_5th` -->
* `blockheight_too_early` (2) - The LSP view of the blockheight is
  less than `target_blockheight + 100`.
  The `data` field includes a field `current_blockheight` indicating what
  the LSP believes the current blockheight to be.
  * The client SHOULD retry later.
* `unknown_lsp` (3) - the LSP gossip map does not include the given
  `lsp_node_id`.
  * The client SHOULD retry later, unless it has seen the `lsp_node_id`
    no longer has open published channels.
* `invalid_proof_of_misbehaviour` (4) - the `zbase` does not validate
  against the given parameters, or `funding_txid` was confirmed on or
  before `target_blockheight` with the `funding_outnum` output validated
  as having `funding_scriptpubkey` and `funding_amount`.

On success, the LSP returns `{}`.

On success, the LSP:

* MUST put the offending `lsp_node_id` into its blacklist, if it is
  not already in the blacklist.
  * MUST close all channels with blacklisted LSPs.
  * MUST NOT open any channels with blacklisted LSPs.
  * MUST refuse any channels opened by blacklisted LSPs.
  * MUST store `funding_txid`, `funding_outnum`, `funding_scriptpubkey`,
    `funding_amount`, `target_blockheight`, `lsp_node_id`, and `zbase` in
    the blacklist, for future `lsps3.downloadpom` calls.
* SHOULD connect to other LSPs as client and also upload the
  same proof-of-misbehavior to them.

> **Rationale** The LSP that accepts another LSP into its blacklist has a
> strong incentive to propagate and share the proof of misbehaviour with
> clients, as it potentially removes that LSP as a competitor.
> This is tantamount to advising clients to not have anything to do with
> the offending LSP, including making channels with the offending LSP.
> It is thus appropriate for the LSP to also not maintain channels with
> the offending LSP, i.e. follow your own advice and refuse to have
> anything to do with the offending LSP.

## Querying Proof of Misbehavior

Clients can learn of misbehaving LSPs by querying other LSPs with
the `lsps3.downloadpom` call.
The call takes no parameters `{}`.

Routing nodes SHOULD also periodically call this API to any identified
LSPs that support LSPS0 and LSPS3, and close channels with misbehaving
LSPs reported in this mechanism.

`lsps3.downloadpom` will always succeed and has no defined
errors, and will return a response:

```JSON
{
  "poms": [
    {
      "lsp_node_id": "023456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012",
      "funding_txid": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
      "funding_outnum": 1,
      "funding_scriptpubkey": "a91485b9ff0dcb34cf513d6412c6cf4d76d9dc24013787",
      "funding_amount": "54600",
      "target_blockheight": 500000,
      "zbase": "blah"
    }
  ]
}
```

`poms` is an array of `pom` objects, which may be empty.
Each `pom` object has keys:

* `lsp_node_id` - the LSP node ID of the offending LSP.
* `funding_txid` - the funding transaction that failed to be confirmed, or did
  not match the expected output.
* `funding_outnum` - an expected output index of the funding transaction.
* `funding_scriptpubkey` - the expected `scriptPubKey` of the `funding_outnum`.
* `funding_amount` - the expected amount of the `funding_outnum`.
* `target_blockheight` - the promised blockheight at which the funding transaction
  should have been confirmed.
* `zbase` - the signature to a message signed via [`signmessage` #specinatweet][].

The `zbase` signature signs the JSON string message below:

    "LSPS3: DO NOT MANUALLY SIGN THIS MESSAGE: I promise to confirm ${funding_txid} whose output ${funding_outnum} has ${funding_scriptpubkey} of ${funding_amount} on or before ${target_blockheight} on pain of boycott"

Where:

* `${funding_txid}` is the unquoted **lowercase** hex string form of the
  `funding_txid`.
* `${funding_outnum}` is the unquoted decimal string form of `funding_outnum`.
* `${funding_scriptpubkey}` is the unquoted **lowercase** hex string form of
  the `funding_scriptpubkey`.
* `${funding_amount}` is the unquoted decimal string form of `funding_amount`.
* `${target_blockheight}` is the unquoted decimal string form of
  `target_blockheight`.

For each `pom`, the client:

* MUST check if the `lsp_node_id` is already in its blacklist; if so,
  it should skip the succeeding checks for this `pom`.
* MUST check that the `zbase` signs the above message, using the
  `lsp_node_id` as public key.
* MUST check that the `funding_txid` is not confirmed, or was confirmed
  above `target_blockheight`, or was confirmed but did not match the
  expected `funding_outnum`, `funding_scriptpubkey`, or `funding_amount`.

If all checks succeed, the client MUST add the given `lsp_node_id` to
its blacklist.
The client MUST NOT use blacklisted LSPs as LSPs, and SHOULD close
any channels with blacklisted LSPs.

## Rational Behavior

Assuming rational behavior, LSPs are discouraged from defrauding
clients if the following hold true:

* Sufficient number of clients, routing nodes, and competing LSPs
  participate in the boycotting protocol.
* At least one client implementation does in fact monitor and
  report misbehavior of unconditional 0-conf channel opens.
* Total unconfirmed 0-conf channel sizes are below the expected loss
  of funds (due to closing fees and mixing fees) and opportunity (due
  to having its channels closed and being boycotted, and having to
  move funds to a new node ID and re-establishing reputation as LSP).

We should note that exit scams are prevalent in Bitcoin and similar
currencies, and proofs of misbehaviour like this scheme do not provide
cryptographic protection against exit scams.
An exit scam is a scheme where some entity with custodial ownership
of customer funds (or even partially-custodial ownership, as in the
0-conf channel case) first builds up reputation with customers, and
then irrevocably burns that reputation in a single theft event of
all custodially-held customer funds.
While the reputation is permanently lost, the single theft event is
often lucrative enough that the loss in reputation is more than
compensated for.

In the context of LSPS3, for a sufficiently large number of
currently-unconfirmed 0-conf channels totalling a sufficiently large
amount, the LSP may decide that future operation as an honest LSP is
less lucrative than simply RBF-ing the currently-unconfirmed 0-conf
channels and having all its channels closed, and mixing its funds to
obscure the destination.
Such a scheme may also be repeated by the same operator repeatedly
building up and destroying its reputation, completely destroying the
LSP ecosystem entirely.

It would be desirable to have some separate mechanism by which LSPs
which engage in exit scams may be punished, to increase the necessary
threshold where an exit scam becomes lucrative.
Such a mechanism is beyond the scope of this specification.

Although this proof-of-misbehaviour scheme cannot protect against exit
scams, it does force scam attempts towards exit scams; in particular,
it protects against LSPs continuously stealing from a small percentage
of its clients in the hope of being beneath notice, and protects
against false accusations towards LSPs.

# LSPS2 JIT Channel Negotiation

| Name    | `jit_channels`           |
|---------|--------------------------|
| Version | 1                        |
| Status  | For Implementation       |

> **Note** Nodes that naively prepayment probe can cause LSPs using LSPS2 to open channels early.
> Naive prepayment probing logic is being updated to greatly reduce the possibility of this happening.
>
> Prepayment probing is a technique by which a payer would replace
> the payment hash with a random number, and then attempt payment to the payee
> (which in the context of this specification would be the client).
> Once the payer received an error from the payee that this was an unrecognized
> hash, it knows that there was a viable route to the payee, and the fees
> involved in the payment, and can ask the human user whether to continue with
> the real payment or not.
>
> Using the "LSP trusts client" model, a naive probe that goes all the way to the payee would cause the
> LSP to open a channel to the client. If the payee never receives a payment over that channel, the LSP
> would not receive payment for the channel open. Note that it is possible for this to not be the case
> in the "Client trusts LSP" model, where the LSP can hold the funding tx until receiving a correct
> preimage.
>
> Payers using prepayment probing do not receive extra benefits to probe private channel
> route hints. A better way to prepayment probe is to ensure probing only goes to the last
> public hop on the invoice. This can be accomplished by checking the graph for the destination
> pubkey and stopping before private pubkeys. This logic is being implemented in LND as well
> as other major payer's probing implementations.

## Motivation

A "JIT Channel" is a channel opened in response to an incoming payment
from the public network to a client, via the LSP.
This allows a client with no Lightning channels to start receiving on
Lightning, and have the cost of their inbound liquidity be deducted from their
first received payment.

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
The 'payer' is who pays for the initial receive of the client.

## Flow (LSP Trusts Client Model)

Overview:

* The client determines that it would like to receive over Lightning, but
  it has no inbound capacity, and the client cannot pre-buy this capacity
  (for instance, it also has no outbound capacity or on-chain funds).
* The client determines the parameters of a particular LSP via a
  `lsps2.get_info` request.
  * The LSP indicates how long those parameters are valid for,
    which defines a timeout for the rest of this flow; if the
    flow does not complete in that timeout, the parameters are no
    longer valid and the LSP MAY refuse to open the channel.
* The client requests for a JIT channel, possibly specifying how much it
  will receive, via a `lsps2.buy` request.
* The LSP returns an SCID, which identifies this request to the LSP.
* The client generates an invoice, which includes the above SCID and the
  LSP node ID as a route hint.
* The client hands the invoice to whoever it will receive funds from.
* The payment is forwarded to the LSP.
* The LSP recognizes the next hop SCID as being a JIT channel request,
  and opens a 0-confirmation channel to the client, which must be
  connected to the LSP at that time.
* The LSP forwards the payment to the client, deducting the channel
  opening fee.
* The client claims the payment.

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

### 1. API Information

`lsps2.get_info` is the entry point for each client using the API.
It indicates any limits the LSP imposes, and parameters for payment.

The client MUST request `lsps2.get_info` to read the `opening_fee` of the
LSP and its related parameters.

`lsps2.get_info` takes the parameters:

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

`lsps2.get_info` has the following errors defined (error code numbers
in parentheses):

* `unrecognized_or_stale_token` (2) - the client provided the `token`,
  and the LSP does not recognize it, or the token has expired.

Example `lsps2.get_info` result:

```JSON
{
    "opening_fee_params_menu": [
        {
            "min_fee_msat": "546000",
            "proportional": 1200,
            "valid_until": "2023-02-23T08:47:30.511Z",
            "min_lifetime": 1008,
            "max_client_to_self_delay": 2016,
            "min_payment_size_msat": "1000",
            "max_payment_size_msat": "1000000",
            "promise": "abcdefghijklmnopqrstuvwxyz"
        },
        {
            "min_fee_msat": "1092000",
            "proportional": 2400,
            "valid_until": "2023-02-27T21:23:57.984Z",
            "min_lifetime": 1008,
            "max_client_to_self_delay": 2016,
            "min_payment_size_msat": "1000",
            "max_payment_size_msat": "1000000",
            "promise": "abcdefghijklmnopqrstuvwxyz"
        }
    ]
}
```
The `opening_fee_params_menu` is an array of `opening_fee_params`
objects.
The LSP MAY return an empty array; in which case, the client currently
cannot use JIT Channels with this LSP.

An `opening_fee_params` object describes how much the LSP will charge for a
channel open, what payment sizes it will accept, and until when the described
parameters will be considered valid.
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
* `min_payment_size_msat` and `max_payment_size_msat` are the limits of the
  payment size, inclusive.
  These are strings containing decimal numbers, in millisatoshis
  [<LSPS0.msat>][].
  The payment size is the amount that the payer is guaranteed to be able to send
  to the client, not including the forwarding fees of nodes along the way.
* `promise` is an arbitrary LSP-generated string that proves to the LSP that
  it has promised a specific `opening_fee_params` with the specific
  `min_fee_msat`, `proportional`, `valid_until`, `min_lifetime`, and
  `max_client_to_self_delay`.

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

> **Rationale** Clients that expect this version of LSPS2 will
> compute the `opening_fee` in the manner indicated in this
> specification, and any additional fields may imply an extension
> that may mislead the client into computing the wrong
> `opening_fee`.
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

The client now takes the `opening_fee_params` and the expected
`payment_size_msat`, to compute the `opening_fee`, and determine if the
resulting `opening_fee` is reasonable.

#### Computing The `opening_fee`

For a given `payment_size_msat` (in millisatoshis) and a selected
`opening_fee_params` object, both client and LSP MUST compute the
`opening_fee` (in millisatoshis) as follows:

    opening_fee = ((payment_size_msat * proportional) + 999999) / 10000000
    if opening_fee < min_fee_msat:
        opening_fee = min_fee_msat

* All numbers MUST be computed in unsigned 64-bit integers.
  * Clients and LSPs MAY use arbitrary-precision integers, such as
    Haskell `Integer` type or bignums in Lisp-like languages, IF AND
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

### 2.  Request JIT Channel

The client constructs a request body for a `lsps2.buy` request,
depending on their need.

The client is identified by the LSP from the BOLT8 connection
that this LSPS API is used on, which identifies and authenticates
the client node ID (also known as client identity public key).
The LSP identifies the client as "connected" if there is a BOLT8
tunnel that has been identified as having the same peer as the
client node ID.

Example `lsps2.buy` request parameters:

```JSON
{
    "opening_fee_params": {
        "min_fee_msat": "546000",
        "proportional": 1200,
        "valid_until": "2023-02-23T08:47:30.511Z",
        "min_lifetime": 1008,
        "max_client_to_self_delay": 2016,
        "min_payment_size_msat": "1000",
        "max_payment_size_msat": "1000000",
        "promise": "abcdefghijklmnopqrstuvwxyz"
    },
    "payment_size_msat": "42000"
}
```

`opening_fee_params` is the object acquired from the previous
step.
Clients MUST copy it verbatim from an entry of `opening_fee_params_menu`
from a result of a `lsps2.get_info` call.
LSPs MUST check that the `opening_fee_params.promise` does in fact
prove that it previously promised the specified `opening_fee_params`.
LSPs MUST check that the `opening_fee_params.valid_until` is not a
past datetime.

`payment_size_msat` is an *optional* amount denominated in millisatoshis
that the client wants to receive [<LSPS0.msat>][]:

* If the client wants to issue a variable-amount invoice ("0-amount
  invoice") the client MUST leave out this parameter, but MUST indicate
  to the payer to **NOT** use multipath payments.
  * "no-MPP+var-invoice" mode.
* If the client wants to issue a fixed-amount invoice, the client
  SHOULD provide this parameter, and if it does, MAY indicate to the
  payer to **USE** multipath payments.
  * "MPP+fixed-invoice" mode.

The client MUST ensure that `payment_size_msat` is within the previous
`min_payment_size_msat` and `max_payment_size_msat` parameters from the LSP.
The LSP MUST validate that the `payment_size_msat` is within the previous
`min_payment_size_msat` and `max_payment_size_msat` parameters from the LSP.

* If the `payment_size_msat` is specified in the request, the LSP:
    * MUST compute the `opening_fee` and check that the computation
      did not hit an overflow failure.
      * MUST check that the resulting `opening_fee` is strictly less
        than the `payment_size_msat`.
    * SHOULD check that it has sufficient incoming liquidity from the
      public network to be able to receive at least `payment_size_msat`.
* otherwise:
    * SHOULD validate that the size of a received variable-amount payment is
      within the previous `min_payment_size_msat` and `max_payment_size_msat`
      parameters before forwarding the payment.

The following errors are specified for `lsps2.buy`:

* `invalid_opening_fee_params` (2) - the `valid_until` field
  of the `opening_fee_params` is already past, **OR** the `promise`
  did not match the parameters.
* `payment_size_too_small` (3) - the `payment_size_msat` was specified,
  and the resulting `opening_fee` is equal or greater than the
  `payment_size_msat`.
* `payment_size_too_large` (4) - the `payment_size_msat` was specified,
  and the LSP hit an overflow error while calculating the
  `opening_fee`, **OR** the LSP has insufficient incoming liquidity
  from the public network to receive the `payment_size_msat`.
* [LSPS0.client_rejected_error][] (1) - The LSP rejected the client.

If there were no errors, the LSP then provides a normal
result to the `lsps2.buy` API.

Example successful `lsps2.buy` result:

```JSON
{
    "jit_channel_scid": "1x4815x29451",
    "lsp_cltv_expiry_delta" : 144,
    "client_trusts_lsp": false
}
```

`jit_channel_scid` is the SCID to use for creating the route hint in
the invoice [<LSPS0.scid>][],
and `lsp_cltv_expiry_delta` is the CLTV delta for that route hint hop.

`client_trusts_lsp` is an *optional* Boolean.
If not specified, it defaults to `false`.
If specified and `true`, the client MUST trust the LSP to actually
create and confirm a valid channel funding transaction.

The client MAY abort the flow if the LSP specified `client_trusts_lsp`
as `true` in the result.

### 3.  Invoice Generation

The client MAY then generate a [BOLT11][] invoice.

[BOLT11]: https://github.com/lightning/bolts/blob/f7dcc32694b8cd4f3a1768b904f58cb177168f29/11-payment-encoding.md

If the client generates a BOLT11 invoice, the client MUST set
the following:

* The sum of `timestamp` and `x`/`expiry` fields is equal to
  or less than `opening_fee_params.valid_until`.
* `n` is the client node ID, or any public key whose private
  key the client knows.
* `r` contains a single route hint with a single hop:
  * `pubkey` is the LSP public node ID.
  * `short_channel_id` is the `jit_channel_scid`.
  * `fee_base_msat` and `fee_proportional_millionths` are 0.
  * `cltv_expiry_delta` is `lsp_cltv_expiry_delta`.
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
* Depending on if the client specified a `payment_size_msat` in
  the previous `lsps2.buy` call:
  * If `payment_size_msat` specified ("MPP+fixed-invoice" mode),
    set the invoice amount to `payment_size_msat` and enable
    the `basic_mpp` feature.
  * If `payment_size_msat` unspecified ("no-MPP+var-invoice" mode),
    do not specify an invoice amount, and disable the
    `basic_mpp` feature.

All other fields are to be filled in by the client.

> **Rationale** In-Lightning fee rates can be folded into the
> `opening_fee` for the initial payment.
> The channel is not open yet, thus the client cannot know
> what the actual fee rates the LSP would impose at the time
> this invoice is generated, so it must be fixed at this
> point.
> Multipart payments are specifically disallowed if
> `payment_size_msat` is unspecified, as the total amount of a
> multipart payment is encrypted in the onion such that only
> the receiver (the client) can decode it, and the LSP cannot
> thus determine what the total amount from the payer would be
> without help from the client.
>
> Bitcoin blocks may be mined at any time.
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
* The `jit_channel_scid`.
* The LSP node ID.
* The `lsp_cltv_expiry_delta`.
* The expiry for the payment.
* Whether it can use MPP or not.

However, the client MAY provide this information to the payer
by means other than BOLT11 invoice, including in a form that
is not decodable by the payer, but which can be decoded by
another LN participant.

The only necessary requirement is for the LSP to see an incoming
HTLC onion that indicates the `jit_channel_scid` as the next hop,
and as long as the client is able to cause the payer to do this,
it is is compatible with this protocol.

> **Non-normative** For example, the client node ID, the
> `jit_channel_scid`, the LSP node ID, and the
> `lsp_cltv_expiry_delta` can be encoded as the last hop of
> a blinded path, for example in a BOLT12 offer or invoice.
> The payer would not have direct access to the information,
> but the introduction point of the blinded path would be able
> to start the decoding of the blinded path.

### 4.  Payment

The client MAY issue the generated invoice to the payer of the
client, or equivalently inform the payer of the necessary
information via other means.

The payer can then generate an outgoing payment part to
deliver the paid amount to the LSP.
If the client selected "MPP+fixed-invoice" mode, the payer can
generate multiple outgoing payments parts.

In "MPP+fixed-invoice" mode, the LSP, if it receives a forward
where the next hop is `jit_channel_scid`:

* SHOULD wait for payment parts of a single payment hash until all
  parts sum up to at least `payment_size_msat`.
  * SHOULD group together payment parts of a single payment hash.
  * SHOULD hold all payment parts active for at least 90 seconds,
    starting from the arrival of the first payment part
    (**Rationale** [BOLT 4 Basic Multi-Part Payments Requirements][]
    requires the final hop to wait at least 60 seconds; this
    replicates that requirement, and adds 30 seconds for
    additional LSP-client communication overhead).
  * MAY ignore new blocks being mined.
    (**Rationale** the requirement to add +2 to the `c` /
    `min_final_cltv_expiry_delta` gives enough protection for
    up to two blocks being mined, from the time the payer sent
    out the payment, to the time the LSP is able to deliver the
    HTLCs inside a new JIT channel.)
  * MUST fail with `temporary_channel_failure` if it implements
    the above timeout, and the timeout is reached.
  * SHOULD, if it errored `temporary_channel_failure` due to the
    above timeout, forget about the failed payment parts, and if
    another payment part arrives after failing, SHOULD consider
    it as beginning another set of payment parts for a payment.
  * MUST also have, for all pending payment parts, a separate timeout
    at `opening_fee_params.valid_until`, described below.
* MUST NOT fail with `mpp_timeout`, as the final hop is the client
  and that error is only allowed at the final hop.
* MUST fail with `unknown_next_peer` if all parts have **NOT**
  summed up to at least `payment_size_msat` **AND**
  `opening_fee_params.valid_until` has passed.
* MAY fail with `unknown_next_peer` if it receives too many payment
  parts.
* When all conditions below are true, MUST continue to the next
  step:
  * The current time is strictly less than
    `opening_fee_params.valid_until`.
  * All received payment parts sum up to at least `payment_size_msat`
    plus `opening_fee`.
  * The client is connected.

[BOLT 4 Basic Multi-Part Payment Requirements]: https://github.com/lightning/bolts/blob/803a532c49be2f152c7f2dbaa0ec7d4c23a6013d/04-onion-routing.md#requirements-1

In "no-MPP+var-invoice" mode, the LSP, if it receives a forward
where the next hop is `jit_channel_scid`, before
`opening_fee_params.valid_until`:

* MUST set `payment_size_msat` as the forwarded value of the HTLC.
* MUST compute the `opening_fee` based on the `payment_size_msat`, and
  if the computation overflows, MUST fail with `unknown_next_peer`.
* MUST check that `opening_fee + htlc_minimum_msat <= payment_size_msat`,
  and if that fails, MUST fail with `unknown_next_peer`.

### 5.  Channel Opening And Forwarding

The LSP requests a channel open to the client via standard
[BOLT2 Channel Establishment][] flow.

[BOLT2 Channel Establishment]: https://github.com/lightning/bolts/blob/c4c5a8e5fb30b1b99fa5bb0aba7d0b6b4c831ee5/02-peer-protocol.md#channel-establishment

The LSP selects the channel size.
The LSP MUST ensure the channel is large enough to transmit the
`payment_size_msat - opening_fee`, plus any reserve needed on the LSP
side.

The LSP and client MUST negotiate these options and parameters in
their [BOLT2 Channel Establishment][] flow:

* `option_zeroconf` is set.
* `option_scid_alias` is set.
* `announce_channel` is `false`.
* The client-provided `to_self_delay` is less than or equal to
  the agreed-upon `max_client_to_self_delay`.

> **Rationale** `option_zeroconf` is set so that the payment can
> be immediately forwarded to the client and resolved as soon as
> the opening negotiation completes; if the client and LSP had to
> wait for confirmation, the incoming payment would have to be
> locked for an extended period, which is indistinguishable from
> a channel jamming attack.
> `option_zeroconf` cannot be set unless `announce_channel` is
> `false`.
> `option_scid_alias` needs to be set so that the channel can be
> referred to on future invoices before the channel confirms.
>
> Future revisions of this API may be able to utilize Asynchronous
> Receive (currently being developed as of this version) to
> be able to get around the `option_zeroconf` requirement, by
> treating the receiver / client as offline until the client is
> online *and* the JIT channel to it is deeply confirmed, if the
> LSP and client negotiate a non-`option_zeroconf` channel for
> the JIT channel.
> A non-`option_zeroconf` JIT channel could also be later
> published with `announce_channel = true`.

Clients MAY reject the channel open attempt if any other
parameters or options are not desired by the client (for example,
it MAY reject non-zero channel reserve requirements imposed on
the client by the LSP).
Rejection is via the `error` Lightning BOLT message.

LSPs MUST fail the payment with `unknown_next_peer` in case the
client rejects the channel open via an `error`.

If the client disconnects before the LSP can receive `funding_signed`
from the client, the LSP MUST fail the incoming payment parts with
`temporary_channel_failure`, then return to the previous step (waiting
for payment).

> **Rationale** The client might have crashed at this point, and
> recovery might take an inordinate amount of time.
> However, use of the `temporary_channel_failure` error informs the
> payer that the error is temporary and it can retry later, if the
> client is online at that point.
>
> Client disconnection before the LSP can receive `funding_signed`
> will prevent the LSP from broadcasting the funding transaction
> and the LSP may re-allocate the UTXOs spent in the funding
> transaction for a different payment.
> Thus, the correct behavior is to abort the payment, so that the
> LSP funds may be re-used in a new channel to the same client if
> the payer retries, or so that the LSP funds may be re-used for a
> different client if the payer gives up on paying the client.

Clients MUST NOT expect that `jit_channel_scid` is the same as
the `alias` SCID.
Clients MUST NOT expect that `jit_channel_scid` is different from
the `alias` SCID.

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

Depending on the `client_trusts_lsp` from the `lsps2.buy` result,
after the client has sent `funding_signed` and the LSP is willing
to broadcast the funding transaction:

* If `client_trusts_lsp` is specified and `true`:
  * The LSP MAY wait for the client to send the preimage to the
    incoming payment via a `update_fulfill_htlc` before broadcasting
    the funding transaction.
  * The client MUST NOT wait for the LSP to broadcast the funding
    transaction before sending the preimage via a
    `update_fulfill_htlc`.
  * The client and LSP MUST immediately send `channel_ready`.
* If `client_trusts_lsp` is unspecified or `false`:
  * The client MAY wait for the funding transaction to appear in
    its mempool or the mempool of a trusted node, or confirmed in a
    block with any depth, AND confirm that the funding transaction
    output is correct, before sending `channel_ready`.
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
new channel, the LSP then forwards payment parts to the client.
In "no-MPP+var-invoice" mode there is only one payment part.

The LSP generates non-standard forwards, where the amount received
by the client is smaller than specified in the onion;
the client MUST accept the non-standard forward(s), provided they
sum to at least `payment_size_msat - opening_fee`.

The LSP MUST ensure that each forwarded part forwards at least
`htlc_minimum_msat` millisatoshis.
The LSP MUST ensure that all parts forwarded sum up to at least
`payment_size_msat - opening_fee`.

The LSP MUST include an `extra_fee` (type 65537) TLV to the
`update_add_htlc` message of forwarded parts that have had
fees deducted.
The LSP MUST NOT include this `extra_fee` TLV if the part does
not have fees deducted (for example, if this is a multipart
payment and the other parts have already paid for the fee).
The `extra_fee` TLV MUST have a length of 8, and the value is a
big-endian 64-bit number containing the number of millisatoshis
deducted from this part.

The client MUST check that the sum of all `extra_fee`s in a
single multipart payment (or the single part of a non-multipart
payment) is less than or equal to the `opening_fee`, and that the
actual amount of each HTLC is equal to the expected amount in the
onion minus the individual `extra_fee`.

`extra_fee` is currently proposed in [bLIP-0025][].

[bLIP-0025]: https://github.com/lightning/blips/pull/25

For example, the LSP can use the algorithm below to determine
how much to forward for each incoming payment part, while ensuring
that both the above properties are preserved.

* Put the payment parts in an array in arbitrary order.
* For each part:
  * Get the `part_amount`, the incoming amount for the part.
  * If `part_amount - htlc_minimum_msat >= opening_fee`, then forward
    `part_amount - opening_fee` for this part, then forward
    the rest of the parts verbatim, if any.
  * Else subtract `opening_fee = opening_fee - (part_amount - htlc_minimum_msat)`,
    then forward `htlc_minimum_msat` for this part.
* The above loop will not reach past the last part unless
  the number of parts times `htlc_minimum_msat` exceeds the
  `payment_size_msat`.
  This case should be checked beforehand and cause the payment
  to be failed due to too many payment parts.

Once the client has received payment parts that sum up to at
least `payment_size_msat - opening_fee`, it MUST claim at least
one payment part.
Once that is done, the protocol flow is considered complete.

The parts that the client receives are non-standard, as the amount
in the onion would not be the same as the amount sent by the LSP.
Clients MUST suppress such checks for parts it recognizes as having
been received to pay for a JIT Channel open.
Clients would need some mechanism to recognize such payments, such as
by flagging such payments in its invoice database, putting such
payments in a separate invoice database, or adding special flags inside
the `metadata` or `payment_secret` of the invoice, or by changing how
`payment_secret` is computed based on whether the invoice is to pay for
a JIT Channel or not and trying both ways.

### 6.  Post-opening Normal Operation

After the channel has been opened and an `alias` has been specified,
the client MUST use the `alias` for future invoices on the
channel, as normal for users of zero-confirmation channels.
Note that use of `alias` forces the `alias` to always be used, even
after the channel is confirmed.

Invoices issued after the channel open MUST use the fee rates
negotiated for the new channel.

> **Rationale** While it is desirable that the client would be able
> to issue multiple invoices before the channel is opened, this may
> require that the `alias` be identical to the `jit_channel_scid`,
> which might not be feasible under some setups.
> In particular, `jit_channel_scid` may be stored in a database that
> has strict time limits from the `opening_fee_params.valid_until`,
> while the `alias` SCID would remain valid while the channel is
> open.
> In addition, the "normal" Lightning base and proportional channel
> fees would need to be applied to the initial payment(s).
> This version of this API therefore requires that the client first
> be paid once via the `jit_channel_scid` before it can be issue
> further invoices with the `alias`.
>
> Future revisions of this API may allow for more flexibility on the
> client side, including support for variable-amount invoices,
> multiple incoming payments being stalled until the `opening_fee`
> is achieved and the channel can be opened, and so on.

LSPs MUST consider the `jit_channel_scid` as yet another alias
for the specified client, up to until the `valid_until` time
selected.
LSPs MUST consider this alias not just for forwarded payments, but
also for onion messages.

[<LSPS0.msat>]: ../LSPS0/common-schemas.md#link-lsps0msat
[<LSPS0.ppm>]: ../LSPS0/common-schemas.md#link-lsps0ppm
[<LSPS0.datetime>]: ../LSPS0/common-schemas.md#link-lsps0datetime
[<LSPS0.scid>]: ../LSPS0/common-schemas.md#link-lsps0scid
[LSPS0.client_rejected_error]: ../LSPS0/common-schemas.md#link-lsps0client_rejected_error
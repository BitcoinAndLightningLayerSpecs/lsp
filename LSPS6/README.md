# LSPS6 Unlinkable Service Token Issuance

| Name    | `ln_service_token`        |
| ------- | ------------------------- |
| Version | 1                         |
| Status  | For Implementation        |

## Motivation

An LSP might want to provide additional Lightning- or
Bitcoin-related services beyond merely providing liquidity and
access to the Lightning Network.
For example, clients may need to access some SPV server in order
to reduce bandwidth load at the client.

However, there is an obvious economic reason for the LSP to
restrict the provision of such services to clients it has economic
ties with (i.e. a channel or a promise of a channel, such as a
pending LSPS1 or LSPS2 channel request).
The simplest implementation is to demand that a purported client
sign, using its node ID, some string, before the LSP provides such
an additional service.

This simplest implementation has the flaw that the LSP is now
aware of exactly which node the client controls.
This may allow the LSP to deny a vital service, such as block
information, to support an attack on the specific node ID.
Although no LSP is expected to attack its clients, external
hackers may gain access to the LSP hardware and cause such attacks
to occur.
Reducing the information that an LSP has about its clients makes
it less attractive as a hacking target, thus protecting the
clients.

This LSPS describes a standard for how an LSP may provide service
tokens, gratis or for a fee, to clients, and how clients can use
the service tokens to gain access to additional services.

This LSPS unlinks two security concepts:

* Authorization: giving permission to some entity to use a
  service.
* Authentication: validating that an entity has permission to
  use a service.

Service tokens generated during authorization are blinded by the
client.
The client can then unblind the token during the subsequent
authentication.
This blinding step prevents the server from linking a token
provided during authorization from when the client uses the token
during authentication.

### Actors

The 'LSP' is the entire entity that provides both access to the
Lightning Network, and additional services.
The 'client' is the entity to which the LSP provides such access
and additional services.

The 'LSP-node' is the part of the LSP entity that is a Lightning
Network node, which the client has a channel (or promise of a
channel) with, and which provides the client access to the
Lightning Network.
It also serves as the authorizer.

The 'LSP-server' is the part of the LSP entity that provides
one or more of the additional services provided by the LSP.
It includes an authenticator of the authorization provided by the
LSP-node.

## Cryptographic Scheme

The blinded service token scheme is based on [PrivacyPass][], with
the following differences:

* The curve is SECP256K1.
  (**Rationale** This is the standard curve used in Bitcoin, and
  unlike the NIST P-256 / SECP256R1 curve, its parameters are
  stringently derived to achieve Koblitz curve properties; in
  theory, NIST could have chosen the supposedly-random parameters
  of NIST P-256 /  SECP256R1 to have some subtle flaw.)
* The hash function is SHA-256 from the SHA-2 family.
  (**Rationale** The SHA-2 family is considered secure even with
  the advent of the SHA-3 family, which is considered an
  alternative and not a replacement to SHA-2, and SHA-256 from the
  SHA-2 family is widely used in Bitcoin.)
* Hash-to-a-point is done by SHA-256 of the input, then treating
  the resulting hash as the X coordinate of a point on the
  SECP256K1 curve with an even Y coordinate; if the X coordinate
  is not on the curve, then hash the hash again ad infinitum,
  until we get a hash where, if it is the X coordinate of a point,
  lies on the curve.
  (**Rationale** The standard `bitcoin-core/secp256k1` project
  includes a function to read in a point in "compressed form",
  where the X coordinate is given and only the sign of the Y
  coordinate is encoded; by prepending `02` to the result of the
  hash, the result can be fed directly into this function to
  check if it encodes a point on the curve.
  While this becomes a variable-time algorithm, the input to the
  hash-to-a-point operation is completely public information and
  the timing does not leak any private data.)
* The blinding is done additively instead of multiplicatively.
  (**Rationale** The standard `bitcoin-core/secp256k1` project
  exposes APIs to perform additive inverse / negation of both
  scalars and field elements / points, but does not expose any API
  to perform multiplicative inverse / division by scalar, which
  would be needed to unblind the tokens.)
* The discrete log equality ("DLEQ") proof is based on
  [this sketch](https://asecuritysite.com/encryption/logeq).
* The CSPRNG used in batched DLEQ is ChaCha20.

[PrivacyPass]: https://privacypass.github.io/protocol/

### Authorization Protocol

First, the client and the LSP-node establishes the service public
key.
This is some public key (a point on SECP256K1) that the LSP-node
"signs" tokens for, and which the LSP-server will later accept as a
service tokem if signed using that public key.
The client, in particular, needs to ensure that the LSP uses the
same service public key for all clients, and does not use a
special key for each client.

The LSP-node publishes the point `S` (the service public key), and
knows the private key `s` such that `S = s * G`, where `G` is the
standard generator point for SECP256K1.

```
Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8
G = SEC-decode('0279BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798')
```

When requesting authorization, the client generates a random
256-bit (32 bytes) number, `t`, the token.

The client then performs hash-to-a-point on `t`:

* Set `input = t`.
* In a loop:
  * Set `x` to `SHA-256(input)`.
  * If `x`, when used as an X coordinate, is a point on the
    SECP256K1 curve (`y * y = x * x * x + 7`), then exit this
    loop and return the point where the Y coordinate is even and
    the X coordinate is `x`.
    * (**Non-normative**) When using `bitcoin-core/secp256k1`,
      you can prepend a `02` byte to the hash, then pass the
      33-byte buffer to `secp256k1_ec_pubkey_parse`; if it returns
      non-0, then the resulting point lies on the SECP256K1 curve
      and has an even Y coordinate.
  * Otherwise, set `input = x` and continue the loop.

Call the result of the above operation `T`.

The client then generates a second random 256-bit number, `b`,
the blinding factor.
It then adds `b * G` to the above point `T`.

The client then sends `b * G + T` to the server.

The LSP-node then multiplies the given point by its secret key `s`
and returns `s * (b * G + T)`, called `C` or the blinded token
signature.

The LSP-node also provides a proof of discrete log equality,
proving that for `s * G` (`S`) and `s * (b * G + T)` (`C`), the
`s` is the same.

The proof of discrete log equality is generated this way:

* Generate random 256-bit number `k`.
* Compute `A` as `k * G`.
* Compute `B` as `k * (b * G + T)`.
* Compute `e` as the SHA-256 hash of the concatenation of the
  following:
  * The 33-byte SEC encoding of `A`.
  * The 33-byte SEC encoding of `B`.
  * The 33-byte SEC encoding of `S` (the server public key).
  * The 33-byte SEC encoding of `C` (the blinded token signature).
* Compute `d` as `k + e * s`, where `s` is the server private key.

The LSP-node then sends `C` (the blinded token signature) and
`(e, d)` (the proof of discrete log equality) to the client.

The client then first validates that the proof of discrete log
equality is valid.

* Compute `A'` as `d * G - e * S`.
* Compute `B'` as `d * (b * G + T) - e * C`.
* Compute `e'` as the SHA-256 hash of the concatenation of the
  following:
  * The 33-byte SEC encoding of `A'`.
  * The 33-byte SEC encoding of `B'`.
  * The 33-byte SEC encoding of `S`.
  * The 33-byte SEC encoding of `C`.
* If `e'` equals `e`, then validation succeeds, otherwise,
  validation fails.

Then, once the validation succeeds, the client can unblind `C`,
by calculating `C - b * S`.
This results in `s * T`.

The client then stores `(t, s * T)` as the token.

#### Batched Authorization

A single client may request for multiple tokens in a single
authorization request.

In that case, the client generates multiple `t[i]` and multiple
`b[i]`, and hashes each `t[i]` onto a point `T[i]`, where `i`
is a numeric 0-based index, with `n` tokens to be generated
(such that `0 <= i <= n - 1`).
The client then sends `(b[i] * G + T[i])` to the LSP-node for
authorization.

The LSP-node, once it has decided to authorize the client and
issue the specified number of tokens, then generates `C[i]`
for each point, by calculating `s * (b[i] * G + T[i])` for all
`i`.

The LSP-node, in this context, provides a batched proof of
discrete log equivalance.
This is a single succinct proof that all the returned points
`C[i]` were computed by multiplying the server private key `s` to
the corresponding `(b[i] * G + T[i])`.

To do so, the LSP-node computes an aggregate `C[all]`, by
the following process:

* Compute `z` as the SHA-256 hash of the concatenation of each
  `C[i]`, in the order of the index `i`.
* Compute `q[i]`:
  * Start with a plain text containing `n * 32` bytes of value
    `0x00`, where `n` is the number of tokens to issue.
  * Encrypt the plain text with ChaCha20 stream cipher, with the
    `z` as the symmetric key, and the nonce as all `0x00` bytes.
  * Get `q[i]` as the resulting ciphertext at indices `i * 32`
    to `(i * 32) + 31`, interpreted as a 256-bit big-endian
    number.
* Compute `C[all]` as the sum of `q[i] * C[i]` for all `i`.
* Compute the sum of `q[i] * (b[i] * G + T[i])` for all `i`.

The LSP-node then constructs the basic proof of discrete log
equivalance using `C[all]` and the sum of
`q[i] * (b[i] * G + T[i])` for all `i`, using those values to
compute the `e` and `B` in the proof.

* Compute `B` as `k * (sum of (q[i] * (b[i] * G + T)))`.
* Replace the `C` in the hash operation of `e` with `C[all]`.

Similarly, on validation, the client computes `C[all]` and the
sum of `q[i] * (b[i] * G + T[i])` for all `i`, using those
values to compute `e'` and `B'` in the validation.

* Compute `B'` as `d * (sum of (q[i] * (b[i] * G + T))) - e * C[all]`.
* Replace the `C` in the hash operation of `e` with `C[all]`.

### Authentication Protocol

The LSP-server issues a challenge string, `m`, to the client.
The challenge string SHOULD be a one-time-use string that is
issued each time the LSP-server wants to authenticate some
client or some use of the service.
A Fiat-Shamir transform can also be used to generate the
challenge string from the hash of some data in the request.

> **Rationale** A one-time-use challenge string prevents an
> token hijacking attack.
> If a third party is able to see the unencrypted communication
> between client and LSP-server, and the challenge string is a
> fixed constant or otherwise changes rarely, then the third
> party can also use the token by simply copying the data sent
> by the client.
>
> In contexts where the client and LSP-server already use an
> encrypted communication (where the encryption keys are
> rotated every time the authentication is made) then it is
> acceptable for the challenge string to be a known constant
> shared by the client and LSP-server.
> This can remove an additional request-response cycle in
> the protocol.
>
> For cases where the client uses a token for a single
> request-response RPC-type interaction with the LSP-server, and
> where the request parameters include cryptographic randomness
> that the client has an incentive to change for each request, it
> is possible to generate the challenge string via the hash of the
> parameters to the request, i.e. use a Fiat-Shamir transform to
> generate the challenge string.
> For example, if a token is used to request the storage of a
> single encrypted blob of data, the hash of the blob can be used
> as part of the challenge string, and only a single message from
> the client to the LSP-server is needed.

The client then selects a token `(t, s * T)` to use for
authentication.
It then computes the HMAC of the challenge string `m`, using
`s * T` as the key:

* Compute `h` as the SHA-256 of the 33-byte compressed SEC
  encoding of `s * T`.
* Calculate HMAC-SHA-256 using `h` as the private key and
  `m` as the message:
  * Calculate the SHA-256 of the concatenation of:
    * 32 bytes of `0x5C` XORed with the 32 bytes of `h`.
    * 32 bytes of `0x5C`.
    * The 32 bytes SHA-256 of the concatenation of:
      * 32 bytes of `0x36` XORed with the 32 bytes of `h`.
      * 32 bytes of `0x36`.
      * The message `m`.

The client then sends `(t, HMAC(s * T, m))` (and possibly `m` as
well as any additional information needed for the server to
validate `m`, if the transport is stateless) to the LSP-server.

The LSP-server then validates the token:

* Compute `s * T`:
  * Calculate `T` by hash-to-a-point of `t`, described above.
  * Multiply `T` by the private key `s` to get `s * T`.
* Calculate `h'` as the SHA-256 of the 33-byte compressed SEC
  encoding of `s * T`.
* Calculate HMAC-SHA-256 using `h'` as the private key and
  `m` as the message, described above.
* If the HMAC-SHA-256 result is the same as the value sent by
  the client, the token is valid and the client is authorized.

The LSP-server MUST add `t` to a set or map of already-used
tokens.
Depending on the use-case, the LSP-server SHOULD reject
already-used `t`s.

### LSP Key Rotation

The key `S` used in both LSP-node authorization and LSP-server
authentication MUST NOT be the LSP-node node ID.

The key `S` MAY be rotated by the LSP periodically.
The LSP MUST NOT rotate `S` faster than once every 7 days.

The LSP-server MUST accept tokens signed by the most recent `S`
key, and MAY accept tokens signed by some number of the most
recent `S` keys.

> **Rationale** The tokens described here cannot contain any
> information other than "this client was authorized".
> In particular, it cannot contain additional information such
> as expiry.
> The only way to implement token expiration is to rotate the
> server key `S` periodically and to accept tokens that were
> signed using a sufficiently-recent key `S`.
>
> Without token expiration, a patient attacker may collect
> tokens over several years, including acquiring tokens from
> hacked or neglected clients (such as by diving into disposed
> storage devices), and then use a large number of tokens to
> overload the LSP-server.
>
> Tokens signed with a particular `S` are linked to that `S`.
> Thus, if the LSP rotate keys too often, it would be able to
> determine if a client acquired the tokens at a particular
> time frame.
> By restricting how often the LSP can rotate keys, such linking
> is reduced.

* The LSP-node signs with a specific server key, the "current
  service key".
  * The LSP-node restricts the number of tokens it issues to
    clients for the current service key.
    Once the LSP-node rotates the current service key to a new
    one, it resets the number of tokens it has issued to a
    client.
  * The LSP publishes this key in a location accessible to all, so
    that clients can ensure that the service key is not unique to
    them (which would let the LSP link the tokens to specific
    clients).
* The LSP-server accepts tokens of the current service key, or
  the past N most recent service keys, with N selected by the LSP.
  * The LSP-node communicates to the client a datetime at which
    the LSP-server will stop accepting authorized service tokens
    for the current authorization.
  * When the LSP-server stops accepting an authorized service
    token for a particular previous service key, the LSP-server
    MAY delete the `t`s recorded for that key, as the entire
    set of tokens for that key will already be invalid.

### Token Usage Patterns

> **Non-normative** These patterns are recommendations.

Tokens issued with this scheme may be used in various ways:

* Single-use tokens.
  * An authorization allows the client to make one "use" of one
    specific service.
    For instance, it may make one query, or make one record
    insertion.
  * The LSP rejects tokens it has seen before (i.e. `t` was
    already recorded).
  * This is the recommended use pattern.
* Account-enablement tokens.
  * The client maintains a pseudonymous (i.e. if public keys
    are involved, MUST NOT use a public key that can be
    publicly derived from their client node ID) account with the
    LSP-server.
  * The client needs to show a token before it can access the
    account on log-in.
  * The LSP maintains a mapping of tokens to client pseudonymous
    accounts.
    An account may reuse a `t` that it already used, but other
    accounts may not use a `t` that was already used by another
    account.
  * The LSP MAY remove or lock an account if the client has not
    accessed the account after a while.
  * This is not recommended, but may fit existing services with
    "log in" semantics better.
    Activity within an account can still be correlated under
    this use-pattern, and potentially correlated with other
    activity the client has with the LSP-node.

### Implementing With `bitcoin-core/secp256k1`

> **Non-normative** In principle, this does not constitute a
> requirement to use `bitcoin-core/secp256k1` to implement this
> specification.
>
> In practice, `bitcoin-core/secp256k1` is a highly-optimized
> constant-time implementation of the math needed for SECP256K1,
> and has bindings in many languages.

In the header `secp256k1.h` of the `bitcoin-core/secp256k1`
[library][libsecp256k1], scalars are called `seckey`, while points
on the SECP256K1 curve are called `pubkey`.

[libsecp256k1]: https://github.com/bitcoin-core/secp256k1

The library has its own dedicated type, `secp256k1_pubkey`, for
points, while scalars are represented by `unsigned char [32]`.

To multiply a scalar by the standard generator point `G` (as in
`S = s * G`), use the function `secp256k1_ec_pubkey_create`.

In the hash-to-a-point operation, there is a need to map an X
coordinate to the SECP256K1 curve, and if it lies on the curve,
to return the point where the Y coordinate is even.
This is implemented by prepending the byte `0x02` to the 32-byte
big-endian representation of the X coordinate, then  feeding
the buffer to `sepc256k1_ec_pubkey_parse` with a `inputlen` of
33.
If the function returns non-zero, then the X coordiante does
indeed lie on the curve and the corresponding point structure
has been initialized correctly.

To multiply a scalar by the standard generator point `G` and
then add it to another point (as in `b * G + T`), use the
function `secp256k1_ec_pubkey_tweak_add`.

To multiply a scalar by an arbitrary point (as in `b * S`),
use the function `secp256k1_ec_pubkey_tweak_mul`.

To negate a scalar (as in `-b`), use the function
`secp256k1_ec_seckey_negate`.

To add arbitrary points together, use the function
`secp256k1_ec_pubkey_combine`.

For example, to compute `C - b * S`, which unblinds the
token issued by the server and returns `s * T`:

```C
/* return 0 on failure / invalid inputs.  */
int
unblind(/*in*/  secp256k1_context const*   ctx,
        /*out*/ secp256k1_pubkey*          s_times_T,
        /*in*/  secp256k1_pubkey const*    C,
        /*in*/  unsigned char const        b          [32],
        /*in*/  secp256k1_pubkey const*    S)
{
  /* -b */
  unsigned char negative_b[32];
  memcpy(negative_b, b, 32);
  if (!secp256k1_ec_seckey_negate(ctx, negative_b))
    return 0;

  /* -b * S */
  secp256k1_pubkey negative_b_times_S;
  negative_b_times_S = *S;
  if (!secp256k1_ec_pubkey_tweak_mul(ctx, &negative_b_times_S, negative_b))
    return 0;

  /* C - b * S */
  secp256k1_pubkey const* ins[2];
  ins[0] = C;
  ins[1] = &negative_b_times_S;
  if (!secp256k1_ec_pubkey_combine(ctx, s_times_T, ins, 2))
    return 0;

  return 1;
}
```

### Test Vectors

TODO

## LSPS Gratis Authorization API

A client may use the LSPS0 protocol to request for service tokens
gratis (at no cost) from the LSP-node.

A client calls the `lsps6.get_gratis_service` RPC to get service
tokens, with parameters:

```JSON
{
  "type": "vss",
  "blinded_tokens": [
    "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
    "0379be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"
  ]
}
```

`type` is an arbitrary string, describing the service that the
client wants to get.

As of this version, the following service `type`s are defined.
Future versions of this LSPS MAY add new valid values for `type`.

* `"vss"` - Access to an access-restricted versioned storage
  system (VSS) server.

`blinded_tokens` is a JSON array of strings, containing the hex
representation of the compressed SEC serializations of the blinded
points `b * G + T`, as described in the section [Cryptographic
Scheme][].
The array may be empty, which is interpreted as a query as to
whether the LSP would provide this service gratis to the client.

[Cryptographic Scheme]: #cryptographic-scheme

The number of tokens in the JSON array is limited by the `type`.

* `"vss"` - Only one token may be given.
  The token is only issued once for each service key rotation
  (i.e. the LSP-node rejects attempts at getting additional
  tokens until the current service key has been rotated out
  from authorization requests).

The client MUST NOT call `lsps6.get_gratis_service` unless one
of the following is true:

* The client currently has at least one channel that has completed
  opening (up to `funding_signed`) with the LSP-node, and which
  has not started any unilateral or mutual close process.
* The client has a pending channel open agreement via some other
  LSPS that opens channels with a client.
  * For example, if the client has an LSPS2 invoice that is
    currently unpaid, but has not reached `valid_until` and is
    therefore still payable.

The LSP-node MUST validate that any of the above conditions are
true before servicing the request.
If all above conditions are false, the LSP-node MUST fail the
RPC with a `not_a_client` error.
The LSP-node MAY fail the RPC with other policies, such as not
providing a particular service gratis to any client, or to the
specific client.

`lsps6.get_gratis_service` has the following errors defined.
Error `code` numbers are in parentheses:

* `no_service`  (1) - The LSP-node does not recognize `type`, or
  the client is not allowed to get this service gratis.
* `not_a_client` (2) - The LSP-node does not recognize the client
  as being a "real" client, such as by having a channel or a
  promise to build a channel, or some LSP-node-defined policy.
* `too_many_issued` (3) - The LSP-node has already issued valid
  gratis tokens to the client, or the length of `blinded_tokens`
  is longer than expected.

On success, `lsps6.get_gratis_service` has a `result`:

```JSON
{
  "server_pubkey": "027100442c3b79f606f80f322d98d499eefcb060599efc5d4ecb00209c2cb54190",
  "server_pubkey_public": "https://vss.example.org/pubkeys",
  "server": "https://vss.example.org/",
  "issued_tokens": [
    "027100442c3b79f606f80f322d98d499eefcb060599efc5d4ecb00209c2cb54191",
    "027100442c3b79f606f80f322d98d499eefcb060599efc5d4ecb00209c2cb54192"
  ],
  "dleq": {
    "d": "0336f3c07adc6c7ae81db3ea1fdd60813eab29c418c08e5b3d3ee7dd228254d1",
    "e": "92d6b07e78ecfa00fe6e975ad1c22fe57fabfb28c785531f6086341a9ab88f39"
  },
  "valid_until": "2023-03-11T19:42:21.999Z"
}
```

`server_pubkey` is a JSON string, a hex representation of the
33-byte SEC compressed serialization of the LSP public key for
this token issuance.
This is the `S` in the [Cryptographic Scheme][] above.

`server_pubkey_public` is a JSON string, containing a public
commitment to the above key as an HTTPS URL.
The client MAY validate the `server_pubkey` against the public
commitment in the following way:

* Use a `GET` request on the specified URL.
  * SHOULD use a `GET` request via some proxy or other method
    that prevents the LSP from providing specific output to
    specific geographical locations.
* Validate that the `GET` request returns a body with MIME type
  `text/plain`.
* Parse the body as a set of SECP256K1 points, in hex
  representations of the 33-byte compressed SEC serialization,
  with points separated by at least one ASCII whitespace character
  (SPACE 32, TAB 9, CR 13, LF 10, VT 11, FF 12).
* Check that the body has no more than 4 points.
* Check that the given `server_pubkey` exists in the body.

> **Rationale** The public commitment is allowed to have multiple
> points as HTTP caching may cause the client to view a stale
> version of the public commitment.
> An LSP that intends to rotate its service key "soon" might
> want to first propagate a commitment that contains its next
> service key, then some time after the rotation, may retain
> the previous key in the commitment in case of a slow client
> whose request was provided before the rotation but which is
> able to access the public commitment after the rotation.

The client MAY validate that the `server_pbukey` for a particular
service `type` does not change more often than once every 7 days.

`server` is a JSON string.
Its format is determined by the `type` in the parameters to this
call.

* `"type": "vss"` - the HTTPS URL of a restricted-access VSS
  server.
  The token may be used to "log in" to the VSS server and keep the
  client-stored data alive.

`issued_tokens` is an array of strings, each string being the hex
representation of the 33-byte compressed SEC serialization of each
returned point, and corresponds to `C` or `C[i]` as described in
[Cryptographic Scheme][].
The `i`th entry of `issued_tokens` corresponds to the `i`th entry
in `blinded_tokens`.
The client MUST validate that `issued_tokens` has the same length
as `blinded_tokens`.

`dleq` is a dictionary containing two fields, `d` and `e`,
corresponding to those terms in the proof of discrete log equality
(DLEQ) as described in [Cryptographic Scheme][].
`d` is the hex representation of the big-endian format of the
256-bit number `d`.
`e` is the hex representation of the SHA-256 hash `e`.
If `blinded_tokens` was empty, then `d` and `e` may be any 256-bit
numbers.

The client MUST validate `dleq` if it gave a non-empty
`blinded_tokens`, as described in [Cryptographic Scheme][].
If `blinded_tokens` was of length 1, then it uses the plain
valiation.
Otherwise, it uses batched validation.

`valid_until` is a [LSPS0.datetime][], the time at which the
LSP-server will no longer accept any issued token for the
current service key.

[LSPS.datetime]: ../LSPS0/common-schemas.md#lsps0-link-datetime

After validation, the client SHOULD unblind the issued tokens
and store both `t` and `C` for the token.

### Issuance Restrictions

Depending on the `type` of service, the length of `blinded_tokens`,
as well as the number of times the client can call
`lsps6.get_gratis_service` with blinded tokens, is limited.

* `"type": "vss"`
  * `blinded_tokens` may be of length at most 1.
  * For each unique `server_pubkey`, the client may request a
    single token up to 1 time.

## LSPS Paid Authorization API

As of this version of this LSPS, there is no defined method by
which clients may purchase service tokens.

A future version of this LSPS MAY provide a set of new APIs
by which clients may purchase a paid tier from an LSP, or
purchase service from an LSP they have no channels with.

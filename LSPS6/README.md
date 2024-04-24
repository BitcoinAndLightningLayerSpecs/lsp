# LSPS6

| Name    | `.well-known` |
| ------- |---------------|
| Version | 1             |
| Status  | Draft         |

## Motivation

The purpose is to provide a standard way for clients to obtain extra information from the LSP.
There might be information that is not available through regular LSPS messages.
An example for such information could be the pricing strategy for LSPS1.
Optionally there is also a mechanism to show authoritatively that the information
is coming from the LSP in order to prevent spoofing.

### Actors

The 'LSP' is the API provider, and acts as the server.
The 'client' is the API consumer.

## Well-Known URI

[Well-known URIs][] are a way to discover information about a service.
The LSP SHOULD serve a well-known URI `/.well-known/bitcoin` to allow for public key discovery.

```json
{
  "icon": "https://acme.com/icon.png",
  "logo": "https://acme.com/logo.png",
  "email": "support@acme.com",
  "nostr": "npub1w...s2mfp9",
  "twitter": "@acme",
  "telegram": "@acme",
  "lightning": {
    "mainnet": {
      "public_keys": [
        "0326e6...2833a3",
        "0305f5...2e4e45"
      ]
    },
    "testnet": {
      "public_keys": [
        "03f060...869c00"
      ]
    },
    ...
  }
}
```

- `icon` LSP MAY include the URL of its icon.
  Icon is a squarely shaped smaller representation of the logo.
  Often the icon is the logo without the text to fit in a square shape.
- `logo` LSP MAY include the URL of its logo.
- `email` LSP MAY include its support email.
- `nostr` LSP MAY include its  npub according to [NIP-19][].
- `twitter` LSP MAY include its Twitter handle starting with an `@` sign.
- `telegram` LSP MAY include its Telegram handle starting with an `@` sign.
- `lightning`
  - `mainnet` LSP SHOULD include its mainnet lightning node public keys.
  - `testnet` LSP MAY include its testnet lightning node public keys.
  - `...` LSP MAY include its public keys for other network types.

[Well-known URIs]: https://datatracker.ietf.org/doc/html/rfc8615
[NIP-19]: https://github.com/nostr-protocol/nips/blob/master/19.md

## API

### lsps6.get_info

`lsps6.get_info` provides generic information about the LSP.

The client SHOULD use the data in `url` up until the [TLD][] and append `/.well-known/bitcoin`.
So assuming the URL is `https://acme.com/lsp`, the client MUST use `https://acme.com/.well-known/bitcoin`.
To confirm authority the client MUST crawl this URL and
match the public key from the LSP with the public key from the well-known URI.

[TLD]: https://en.wikipedia.org/wiki/Top-level_domain

Any information that is available in the response of `lsps6.get_info`
should override the information in the well-known URI as it's more authoritative.

If there is a piece of information unavailable,
the client SHOULD use the information from the well-known URI as a fallback.

**Request** No parameters needed.

**Response**

```JSON
{
  "url": "https://acme.com/lsp",
  "well_known": true,
  "icon": "https://acme.com/lsp-icon.png",
  "logo": "https://acme.com/lsp-logo.png",
  "email": "lsp-support@acme.com",
  "nostr": "npub1w...s2mfp9",
  "twitter": "@acme-lsp",
  "telegram": "@acme-lsp"
}
```

- `url` LSP SHOULD include the URL of its website.
- `well_known` LSP MAY include a boolean to indicate if the LSP has a well-known URI.
The default is that it does not support it.
- `icon` LSP MAY include the URL of its icon.
Icon is a squarely shaped smaller representation of the logo.
Often the icon is the logo without the text to fit in a square shape.
- `logo` LSP MAY include the URL of its logo.
- `email` LSP MAY include its support email.
- `nostr` LSP MAY include its npub according to [NIP-19][].
- `twitter` LSP MAY include its Twitter handle starting with an `@` sign.
- `telegram` LSP MAY include its Telegram handle starting with an `@` sign.

### lsps6.get_lsp_info

`lsps6.get_lsp_info` provides generic information about an LSPS spec.

**Request**
```JSON
{
  "spec": "LSPS1"
}
```

**Response**

```JSON
{
  "short": "Short description",
  "long": "Longer description of the details"
}
```

LSP MUST always respond even when there is no support for a spec. LSP MUST include the following fields
(field values can be empty):
- `short` is a short description of the details with no formatting. With a maximum of 64 characters.
- `long` is a long description of the details with md formatting. With a maximum of 1024 characters.

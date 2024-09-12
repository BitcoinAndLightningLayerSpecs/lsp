# LSPS6 LSP meta-data

| Name    | `meta-data` |
| ------- |---------------|
| Version | 1             |
| Status  | Draft         |

LSPS6 requires [BOLT8][] as a transport layer.

[BOLT8]: https://github.com/lightning/bolts/blob/master/08-transport.md

## Motivation

Not all information is available through regular LSPS messages.
In order to provide more transparency, LSPS6 provides the ability to deliver extra information.
This extra information can be in the form of a URL, icon, logo and/or a support email address.
We need to be conscious about information spoofing.

### Actors

The 'LSP' is the API provider, and acts as the server.
The 'client' is the API consumer.

## DNS Public Key Verification

[BIP353][] provides a way to discover payment instructions over DNS.
Analogous to this the LSP SHOULD create a DNS TXT records in the form of
`lsps.acme.com. 3600 IN TXT "public_key:0326e6...2833a3"` for public key verification.
If the LSP has multiple lightning network nodes, then the LSP SHOULD create multiple corresponding TXT records.
If the LSP has multiple domains which is wishes to reference to, then the LSP SHOULD create multiple corresponding TXT records for each domain.
In the given example `0326e6...2833a3` should be replaced with the lightning network nodes' public key of the LSP (in zbase32 format).
The LSP must replace `acme.com` in `lsps.acme.com.` with their own domain.
Any TXT record not starting with `public_key:` can be ignored by the client.

The response from [lsps6.get_info](#lsps6get_info) may contain URLs and/or an email address in it's response.
When the LSP indicates support for DNS Public Key Verification via the flag is_dns_verification_available then
the client MUST confirm authenticity by verify the public key(s) from the domain's DNS record(s)
with the public key that sends the [BOLT8][] messages.
The client MUST verify all distinct domains.

[BIP353]: https://github.com/bitcoin/bips/blob/master/bip-0353.mediawiki

## API

### lsps6.get_info

`lsps6.get_info` provides generic information about the LSP like icon, logo, etc. Omitted fields are considered as not supported/available.
Any provided URLs should be valid and reachable. So starting with `http(s)://`.

**Request** No parameters needed.

**Response**

```JSON
{
  "url": "https://acme.com/lsp",
  "is_dns_verification_available": true,
  "icon": "https://acme.com/lsp-icon.png",
  "logo": "https://acme.com/lsp-logo.png",
  "email": "lsp-support@acme.com"
}
```

- `url` LSP SHOULD include the URL to its website with a maximum length of 64 characters.
  The LSP can choose to include a URL to a specific page that provides more information about the LSP or simple a top level domain.
- `is_dns_verification_available` is an **optional** boolean field, if present it's purpose is to indicate if the LSP supports DNS Public Key Verification.
- `icon` is an **optional** field, if present it contains the URL to an icon with a maximum length of 128 characters.
  Icon is a squarely shaped smaller representation of the logo.
  Often the icon is the logo without the text to fit in a square shape.
- `logo` is an **optional** field, if present it contains the URL to a logo with a maximum length of 128 characters.
- `email` is an **optional** field, if present it contains the support email address.

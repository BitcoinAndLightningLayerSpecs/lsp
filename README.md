# Lightning Service Provider Spec
API specifications for Lightning Service Providers (LSP)

The goal of this repository is to provide a unified API specification for Lightning Service Providers (LSP) to create interoperability between different Lightning Network wallets/clients and LSPs.

The group meets every second week on Wednesday 11am UTC. The meeting is open to everyone. Please join the [Telegram group][] for more information.

- [Telegram group][]
- [Meeting transcriptions](https://github.com/BitcoinAndLightningLayerSpecs/meetings)

[Telegram group]: https://t.me/LSPstandards

## Status Specification

All LSPS specifications include a "Status" field.
"Status" can be one of the following:

* "Draft" - The specification is still under active development and
  subject to change. Implementation is not recommended at this
  time.
* "For Implementation" - The specification has been widely reviewed by
  LSPS participants, is believed to have addressed all raised
  issues, and LSPS participants recommend this specification to be
  implemented.
* "Stable" - The specification has been implemented by at least one
  client and one LSP, which are developed by at least two different
  organizations or open-source project teams, and both development
  teams have reported interoperability without further modifications
  or clarifications of the specification.

## Specs

### **LSPS0** [Transport Layer](LSPS0/README.md)
Describes the basics of how clients and LSPs communicate to each other.

### **LSPS1** [Channel Request](LSPS1/README.md)
A channel purchase API to buy channels from an LSP.

### **LSPS2** [JIT Channels](LSPS2/README.md)
Describes how a client can buy channels from an LSP, by paying via a deduction from their incoming payments, creating a channel just-in-time to receive the incoming payment.

## Services
List of Lightning Service Providers in alphabetic order that currently or will support LSP specs in future.

(If you would like to get added to this list, please open a PR and update the table below)

| Name         | Specs       | Status |
| ------------ | ----------- | ------ |
| Blocktank    | -           | -      |
| Breez        | -           | -      |
| c=           | -           | -      |

## Responsible Disclosure

As [CVE-2019-12998][], [CVE-2019-12999][], and [CVE-2019-13000][] show, it
is possible to have CVE-level bugs in the specification.

[CVE-2019-12998]: https://nvd.nist.gov/vuln/detail/CVE-2019-12998
[CVE-2019-12999]: https://nvd.nist.gov/vuln/detail/CVE-2019-12999
[CVE-2019-13000]: https://nvd.nist.gov/vuln/detail/CVE-2019-13000

Please report any security-critical issues regarding LSPS to the contacts
below.
When reporting, please encrypt your security-critical bug report via PGP /
GPG and send to the emails below, with the subject `[LSPS Bug]`.

* ZmnSCPxj jxPCSnmZ <zmnscpxj@cequals.xyz>
* Yaacov Akiba Slama <yaslama@breez.technology>
* Reza <reza@synonym.to>

Import the public keys from the [responsible disclosure public
keys](./responsible-disclosure-public-keys.asc) file.

If necessary, also include your own PGP public key when sending a
bug report, so that we can reply privately.

For example, you could write your bug report into a text file:

```sh
cat > lsps999-cve-bug-report.txt <<TEXT
In LSPS999, section "Some Section Name", the description of some
process lacks a check of some value.
If this check is not performed by the client, then a malicious
LSP can provide an invalid value, which will lead to the client
expecting some other thing, and then losing all their funds.

I recommend putting a check of this value explicitly in the
specification, so that clients following the written
specification are not vulnerable to funds loss.
TEXT
```

Then, import the above public keys:

```sh
gpg --import responsible-disclosure-public-keys.asc
```

And then encrypt the file so that recipients can read it:

```sh
gpg --armor --encrypt --recipient zmnscpxj@cequals.xyz --recipient yaslama@breez.technology --recipient reza@synonym.to lsps999-cve-bug-report.txt
```

This will generate a `lsps999-cve-bug-report.txt.asc` file, which
you can email to the above recipients, with the subject
`[LSPS Bug]`.

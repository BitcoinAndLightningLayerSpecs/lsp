# Lightning Service Provider Spec
API specifications for Lightning Service Providers

The current scope is limited to channel sales and services specifically about your channel and the peer that provides it.

Once the important aspects are covered, we may work on liquidity marketplace concepts, and other wallet or info services.


## Status Specification

All LSPS specifications include a "Status" field.
"Status" can be one of the following:

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

### **LSPS.1** [Channel Request](channel-request.md)
A unified channel request API standard for services to buy channels from LSP. This spec supports both just in time channel openings and pre paid channels



## Services
List of Lightning Service Providers in alphabetic order that currently or will support LSP specs in future.

(If you would like to get added to this list, please open a PR and update the table below)

| Name | Specs | Status |
| ---- | ----------- | ------ |
| Blocktank | LSPS1 | Soon |
| Breez | [Breez](https://github.com/breez/lspd/blob/master/rpc/lspd.md) / LSPS1 | In production / Soon |
| LNBIG | ? |  ?  |
| SwissRouting | ? |  ?  |
| TBD | ? |  ?  |
| ZeroFeeRouting | ? |  ?  |



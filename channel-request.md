# LSPS1 channel request

`Version: 0.0.2`

The specification is defined in [LSPS1.yaml](./OpenAPI/LSPS1.yaml) and can be viewed in an OpenAPI editors like the [Swagger Editor](https://editor.swagger.io/).


## Base API

The base api consists of 4 endpoints:
- `GET /lsp/channels`: General API information.
- `POST /lsp/channels`: Create channel request.
- `GET /lsp/channels/{id}`: Get request state.
- `POST /lsp/channels/{id}/open`: Synchronously open channel.

The request is split between 2 phases: Purchasing a channel and opening a channel.

## Features

The base api can be extended by additional features. Feature either can extend the base api or add additional endpoints. Base api needs to stay backward-compatible though.

Each supported feature must to be listed in `GET /lsp/channels`.`features` either as a `string` or as a object.

```json
features: [
    "local_balance",
    {
        "name": "onchain_payments",
        ...Additional settings
    }
]
```

### lnurl_lud2

The feature `lnurl_lud2` adds a new endpoint `GET /lsp/channels/{id}/lnurl` that provides an [lnurl channel request](https://github.com/lnurl/luds/blob/luds/02.md) according to [LUD2](https://github.com/lnurl/luds/blob/luds/02.md).

### local_balance

The feature `local_balance` allows `POST /lsp/channels`.`local_balance` to be greater 0. Otherwise it is 0 by default.

### refunds

The feature `refunds` adds two endpoints to automatically manage refunds.

- `POST /lsp/channels/{id}/refund` allows a user to claim the refund with a Lightning invoice or an onchain address.
- `GET /lsp/channels/{id}/refund` shows the refund state and available amount.

### jit_channels

The feature `jit_channels` adds the ability to to register a node and receive a route hint to add the channel Just-In-Time.

### onchain_payments

The feature `onchain_payments` indicates if this LSP is willing to receive payments onchain. The feature has 1 required setting:

- `min_satoshi` indicates at what value the LSP is willing to receive payments onchain.

In case the LSP doesn't support onchain payments, all values related to onchain payments will be `null`.

### /lsp/channel

#### GET
##### Summary

Request an inbound channel

##### Description

Request an inbound channel with a specific size and duration.


### /lsp/channel

#### GET
##### Summary

Get information about a channel order

##### Description

Get information about a channel order

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| id | query | order_id provided in response to channel request or scid | Yes | string |

##### Response

| Name | Description | Schema |
| ---- | ----------- | ------ |
| order_id | The order id | string |
| created_at | Number of seconds since epoch when this order was created | number |
| local_balance | Local balance in sats requested by client | number |
| remote_balance | Remote balance in sats requested by client | number |
| channel_expiry | Channel expiry in weeks requested by client | number |
| channel_expiry_ts | Number of seconds since epoch when the channel can be closed | number |
| order_expiry_ts | Number of seconds since epoch when this order can still be paid | number |
| order_total | The total fee plus the local_balance requested | number |
| fee_total | The total fee the lsp will charge to open this channel | number |
| fee_per_payment | For intercepted payment of fee, fee taken from each payment until fee is paid | number |
| temporary_scid | The scid user puts in the route hint of invoice to identify order | string |
| scid | The scid of the channel if already established | string |
| lsp_connection_info | pubkey or pubkey@host:port of the lsp node | string |
| ln_invoice | A lightning bolt11 invoice to pay the fee for this channel open | string |
| btc_address | An on-chain bitcoin address to pay the fee for this channel open | string | 
| lnurl_channel | A way to request the open via lnurl after the order is paid | string |
| amount_paid | Amount paid by client so far in sats | number |
| node_connection_info | The node_connection_info for the node to open the channel to | string |
| channel_open_tx | The txid of the channel funding tx once it is broadcast | string |
| state | The state of the order | string |
| onchain_payments | A list of payments received to btc_address on-chain | object[] |


###### Order state enum

| State          	| Description                                                                           	|
|----------------	|---------------------------------------------------------------------------------------	|
| CREATED       	| The order has been created but the user hasn't paid yet.                               	|
| UNDER_FUNDED      | Funding received but under paid.                                                       	|
| PENDING        	| Order is paid but the channel has not been opened yet.                                	|
| OPENING       	| The opening transaction has been broadcasted. 0conf might skip directly to OPENED.     	|
| OPENED         	| Channel is open and has the necessary block confirmations.                            	|
| CLOSING           | The closing transaction has been broadcasted.                                             |
| CLOSED            | Channel is closed and has the necessary block confirmations.                              |
| FAILED         	| Any error. For example, the LSP couldn't connect to the target node.                  	|
| REFUNDED       	| Payment has been refunded.                                                            	|
| OFFER_EXPIRED  	| Order has not been paid and offer has therefore expired.                              	|



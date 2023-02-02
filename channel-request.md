# LSP channel request


## Version: 0.0.2

### /lsp/channel

#### POST
##### Summary

Request an inbound channel

##### Description

Request an inbound channel with a specific size and duration.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ------ |
| coupon_code | body | pubkey or pubkey@host:port | Yes | string |
| remote_balance | body | Inbound liquidity amount in satoshis | No | integer |
| local_balance | body | Outbound liqudity amount in satoshis | No | integer |
| on_chain_fee_rate | body | On-chain fee rate of the channel opening transaction in satoshis per vbyte | No | number |
| channel_expiry | body | Channel expiration in weeks | No | integer |


##### Response

| Name | Description | Schema |
| ---- | ----------- | ------ |
| order_total | The total fee plus the local_balance requested | number |
| fee_total | The total fee the lsp will charge to open this channel | number |
| fee_per_payment | For intercepted payment of fee, fee taken from each payment until fee is paid | number |
| temporary_scid | The scid user puts in the route hint of invoice to identify order | string |
| lsp_connection_info | pubkey or pubkey@host:port | string |
| ln_invoice | A lightning bolt11 invoice to pay the fee for this channel open | string |
| btc_address | An on-chain bitcoin address to pay the fee for this channel open | string |
| order_id | An lsp generated order id used to look-up the status of this request | string |
| lnurl_channel | A way to request the open via lnurl after the order is paid | string |

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



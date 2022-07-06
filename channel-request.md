# LSP channel request


## Version: 0.0.1

### /lsp/channel

#### POST
##### Summary

Request an inbound channel

##### Description

Request an inbound channel with a specific size and duration.

##### Parameters

| Name | Located in | Description | Required | Default | Schema |
| ---- | ---------- | ----------- | -------- | ------- | ------ |
| remote_balance | body | Inbound liquidity amount in satoshis | Yes | | integer |
| local_balance | body | Outbound liqudity amount in satoshis | No | 0 | integer |
| channel_expiry | body | Channel expiration in weeks | Yes | | integer |
| node_connection_info | body | pubkey@host:port | Yes | | string |


##### Response

| Name | Description | Schema |
| ---- | ----------- | ------ |
| order_total | The total fee plus the local_balance requested | number |
| fee_total | The total fee the lsp will charge to open this channel | number |
| fee_per_payment | For intercepted payment of fee, fee taken from each payment until fee is paid | number |
| scid | The scid user puts in the route hint of invoice to identify order | string |
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
| order_id | query | The order_id provided in response to channel request | Yes | string |

##### Response

| Name | Description | Schema |
| ---- | ----------- | ------ |
| id | The order id | string |
| created_at | Number of seconds since epoch when this order was created | number |
| local_balance | Local balance in sats requested by client | number |
| remote_balance | Remote balance in sats requested by client | number |
| channel_expiry | Channel expiry in weeks requested by client | number |
| channel_expiry_ts | Number of seconds since epoch when the channel can be closed | number |
| order_expiry_ts | Number of seconds since epoch when this order can still be paid | number |
| order_total | The total fee plus the local_balance requested | number |
| fee_total | The total fee the lsp will charge to open this channel | number |
| fee_per_payment | For intercepted payment of fee, fee taken from each payment until fee is paid | number |
| scid | The scid user puts in the route hint of invoice to identify order | string |
| ln_invoice | A lightning bolt11 invoice to pay the fee for this channel open | string |
| btc_address | An on-chain bitcoin address to pay the fee for this channel open | string | 
| order_id | An lsp generated order id used to look-up the status of this request | string |
| lnurl_channel | A way to request the open via lnurl after the order is paid | string |
| amount_paid | Amount paid by client so far in sats | number |
| node_connection_info | The node_connection_info for the node to open the channel to | string |
| channel_open_tx | The txid of the channel funding tx once it is broadcast | string |
| state | The state of the order | string |
| onchain_payments | A list of payments received to btc_address on-chain | object[] |
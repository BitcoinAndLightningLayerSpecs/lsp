# LSP channel request


## Version: 0.0.1

### /node/info

#### GET
##### Summary

Returns general service information about LSP

##### Description

Returns information about LSP Lightning node and services on offer.

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | Node and service info |

### /channel/buy

#### POST
##### Summary

Request a channel to purchase.

##### Description

Request a channel to purchase.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| Channel request | body | Channel to purchase. | Yes | object |

##### Responses

| Code | Description | Schema |
| ---- | ----------- | ------ |
| 200 | Channel quote | [ChannelQuote](#channelquote) |

### /channel/manual_finalise

#### POST
##### Summary

Finalise a purchased channel

##### Description

Set the node that LSP will open a channel to after paying for your channel.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| Channel request | body | Channel to purchase. | Yes | object |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | Channel claimed |

### /channel/order

#### GET
##### Summary

Get an order

##### Description

Get all information regarding a channel order

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| order_id | query | Order id. | Yes | string |

##### Responses

| Code | Description | Schema |
| ---- | ----------- | ------ |
| 200 | Channel quote | [ChannelOrder](#channelorder) |

### /lnurl/channel

#### GET
##### Summary

LN URL connect to node

##### Description

LNURL Channel

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| order_id | query | Required for LNURL connect | Yes | string |
| k1 | query | Required for LNURL callback | Yes | string |
| remote_id | query | Required for LNURL callback. Remote node address of form node_key@ip_address:port_number | Yes | string |

##### Responses

| Code | Description | Schema |
| ---- | ----------- | ------ |
| 200 | LNURL connect  | [LNURLConnect](#lnurlconnect) |

### Models

#### LNURLConnect

| Name | Type | Description | Required |
| ---- | ---- | ----------- | -------- |
| k1 | string | order id | Yes |
| tag | string |  | Yes |
| callback | string | A second-level URL which would initiate an OpenChannel message from target LN node | Yes |
| uri | string | LSP node info | Yes |
| status | string | Response status<br>_Enum:_ `"OK"`, `"ERROR"` | Yes |
| reason | string | Error reason | Yes |

#### ChannelQuote

| Name | Type | Description | Required |
| ---- | ---- | ----------- | -------- |
| order_id | string |  | Yes |
| ln_invoice | string |  | Yes |
| total_amount | integer |  | Yes |
| btc_address | string |  | Yes |
| lnurl_channel | string |  | Yes |

#### ChannelOrder

| Name | Type | Description | Required |
| ---- | ---- | ----------- | -------- |
| _id | string | Order id | Yes |
| local_balance | integer |  | Yes |
| remote_balance | integer |  | Yes |
| channel_expiry | integer | Channel expiry is in weeks. | Yes |
| channel_expiry_ts | integer | LSP has the righ to close the channel after this time | Yes |
| order_expiry | integer | order is valid until this time | Yes |
| total_amount | integer | total amount payable by customer | Yes |
| btc_address | string | Destination address for on chain payments | Yes |
| created_at | integer | Time that the order was created | Yes |
| amount_received | number | how much satoshi orders has recieved | Yes |
| remote_node | object |  | Yes |
| channel_open_tx | object |  | Yes |
| purchase_invoice | string |  | Yes |
| lnurl | object | LNUrl channel object | Yes |
| state | [OrderStates](#orderstates) |  | Yes |
| onchain_payments | [ object ] |  | Yes |

#### OrderStates

Order state can be one of the following

| Name | Type | Description | Required |
| ---- | ---- | ----------- | -------- |
| CREATED | number | Order has been created | Yes |
| PAID | number | Order has been paid | Yes |
| URI_SET | number | Order has been paid and node uri is set | Yes |
| OPENING | number | Lightning channel is opening | Yes |
| CLOSING | number | Lightning channel is closing | Yes |
| GIVE_UP | number | Gave up opening channel | Yes |
| CLOSED | number | Lightning channel has been closed | Yes |
| OPEN | number | Lightning channel is open | Yes |

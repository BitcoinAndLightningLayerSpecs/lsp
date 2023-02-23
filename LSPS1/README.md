# LSPS1 channel request

| Name    	| base_api                      |
|---------	|------------------------------	|
| Version 	| 2                             |
| OpenApi 	| [LSPS1.yaml](./LSPS1.yaml) 	|



## Motivation

The goal of this specification is to provide a standardized LSP API to purchase channels. Two client types are supported:

**Wallets** that purchase a channel from the LSP directly.

**Websites** that provide a UI for channel purchases. This should enable channel marketplaces and LSP channel purchase websites.

To allow everybody to adopt this specification, we define a **HTTP** api. We purposefully decided against a lightning native api to not have to rely on any lightning implementation developers time and therefore reduced adoption potential.

Some requirements are subtle; we have tried to highlight motivations and reasoning behind the results you see here. I'm sure we've fallen short; if you find any part confusing or wrong, please contact us and help us improve.

## Definitions

### Datetime

All datetimes are represented as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) in the form of `2023-02-23T08:47:30.511Z` UTC.

> **Rational** ISO8601 is human readable, it's a standard endorsed by [W3C](http://www.w3.org/TR/NOTE-datetime), and [RFC3339](https://www.rfc-editor.org/rfc/rfc3339), sortable, and supported widely.

### Satoshi

All satoshi values MUST be represented as string and NOT integer values. Make sure you check your integer limits.

> **Rational** Plenty of json parsers use a 32bit signed integer for integer parsing. Max safe value is 2,147,483,647; 2,147,483,647sat = BTC21.474,836,47 which is too low.

### Actors

`LSP` is the API provider. `User` is the user of the API.

### Channel sides

Channel sides are seen from the user point of view. `local_balance` are the funds on the user side. `remote_balance` are the funds on the LSP side.

## Extensions

The API is split between the base api and possible extensions. The base api MUST be implemented. Extensions MAY be implemented and are defined in [extensions](./extensions/).

All specs are defined in the [OpenAPI](https://www.openapis.org/about) format. It can be viewed with various editors, most notably the [Swagger Editor](https://editor.swagger.io/) or the [vscode-openapi](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi) extension. All the OpenAPI documents include detailed descriptions of every field.

| Extension                              	    | Version 	|
|----------------------------------------------	|---------	|
| [lnurl_lud2](./extensions/lnurl_lud2/)        | 1       	|
| [refunds](./extensions/refunds/)          	| 1       	|
| [jit_channels](./extensions/jit_channels/) 	| 1       	|

## Flow

### API information

`GET /lsp/channels` is the entrypoint for each client using the api. It lists the version of the api and all supported extensions in a dictionary.

An extension in `GET /lsp/channels`.`options` MUST include the properties in `AbstractExtensionOptions` and MAY have additional properties based on the individual extensions requirement.

```yaml
AbstractExtensionOptions: # All extensions derive from this schema.
    type: object
    properties:
        version:
            type: integer
            example: 1
            default: 1
            nullable: true
            description: Version of this extension. Integer starting at 1 counting up.
```

The user MUST pull `GET /lsp/channels` to read

- the `version` of the api AND `options` and therefore prove compatibility.
- `options` properties to determine the api boundries.

The user MAY abort the flow here.

Example `GET /lsp/channels` response: 
```JSON
{
  "version": 2,
  "website": "http://example.com/contact",
  "extensions": {
    "base_api": {
      "version": 1,
      "max_local_balance_satoshi": "0",
      "max_remote_balance_satoshi": "100000000",
      "min_required_onchain_satoshi": null,
      "max_channel_expiry_weeks": 24
    }
  }
}
```

#### Base api options

The base api itself has multiple properties that MUST be defined.

```json
"options": {
    "base_api": {
        "version": 1,
        "max_local_balance_satoshi": "0",
        "max_remote_balance_satoshi": "100000000",
        "min_required_onchain_satoshi": null,
        "max_channel_expiry_weeks": 24
    }
}
```

- `version` MUST be 1.
- `max_local_balance_satoshi` MUST be the maximum number of satoshi that the LSP is willing to push to the user. MAY be 0 if pushing satoshi is unsupported.
- `max_remote_balance_satoshi` MUST be the maximum number of satoshi that the LSP is willing to contribute to the remote balance.
- `min_required_onchain_satoshi` MUST be the number of satoshi (`order_total_satoshi` see below) that are required for the user to pay funds onchain. The LSP MUST allow onchain payments equal or above this value. MAY be null if onchain payments are unsupported.
- `max_channel_expiry_weeks` MUST be the maximum length in weeks a channel can be leased for.


### 2. Create Order
Client constructs the request body depending on their needs. 
- Client MUST respect the base_api.options. 
- Client MUST cals `POST /lsp/channels`.

**Example request body**

```json
{
  "order": {
    "remote_balance_satoshi": "5000000",
    "local_balance_satoshi": "2000000",
    "onchain_fee_rate": 1,
    "channel_expiry_weeks": 12,
    "coupon_code": ""
  },
  "open": {
    "announce": true,
    "node_connection_string_or_pubkey": "03d4e028a0d4a90868ec202ab684fb0085779defea9ca7553e06146557631eec20@3.33.236.230:9735"
  }
}
```

- `order.remote_balance_satoshi` must be 1 or greater. Must be below or equal `base_api.max_remote_balance_satoshi`.
- `order.local_balance_satoshi` must be 0 or greater. Must be below or equal `base_api.max_local_balance_satoshi`. Todo: Rejection error message.
- `order.onchain_fee_rate` must be 1 or higher. The LSP may increase this value depending on the onchain fee environment.
- `order.channel_expiry_weeks` must be 1 or greater. Must be below or equal `base_api.max_channel_expiry_weeks`.
- `order.coupon_code` must be a string, null, or not defined at all.
- `open` MAY be null or undefined. This info can be provided with `POST /lsp/channel/{id}/update` later.
    - `announce` If the channel should be announced to the network. MUST be boolean.
    - `node_connection_string_or_pubkey` MUST be a node_connection_string or a pubkey. If pubkey the user MUST establish a peer connection with the LSP after the payment. Otherwise the LSP may not be able to establish a peer connection with the node.

**Example response body**

HTTP Code: 201 CREATED

```json
{
  "id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c",
  "state": "AWAITING_PAYMENT",
  "remote_balance_satoshi": "5000000",
  "local_balance_satoshi": "2000000",
  "onchain_fee_rate": 1,
  "channel_expiry_weeks": 12,
  "coupon_code": "",
  "lsp_node_connection_strings": [
    "03864ef025fde8fb587d989186ce6a4a186895ee44a926bfc370e2c366597a3f8f@3.33.236.230:9735",
    "03864ef025fde8fb587d989186ce6a4a186895ee44a926bfc370e2c366597a3f8f@gwdllz5g7vky2q4gr45zguvoajzf33czreca3a3exosftx72ekppkuqd.onion:9735"
  ],
  "created_at": "2012-04-23T18:25:43.511Z",
  "expires_at": "2015-01-25T19:29:44.612Z",
  "open": {
    "announce": true,
    "node_connection_string_or_pubkey": "03d4e028a0d4a90868ec202ab684fb0085779defea9ca7553e06146557631eec20@3.33.236.230:9735",
    "state": "PENDING",
    "fail_reason": null
  },
  "payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_satoshi": "8888",
    "order_total_satoshi": "2008888",
    "lightning_invoice": "lnbc252u1p3aht9ysp580g4633gd2x9lc5al0wd8wx0mpn9748jeyz46kqjrpxn52uhfpjqpp5qgf67tcqmuqehzgjm8mzya90h73deafvr4m5705l5u5l4r05l8cqdpud3h8ymm4w3jhytnpwpczqmt0de6xsmre2pkxzm3qydmkzdjrdev9s7zhgfaqxqyjw5qcqpjrzjqt6xptnd85lpqnu2lefq4cx070v5cdwzh2xlvmdgnu7gqp4zvkus5zapryqqx9qqqyqqqqqqqqqqqcsq9q9qyysgqen77vu8xqjelum24hgjpgfdgfgx4q0nehhalcmuggt32japhjuksq9jv6eksjfnppm4hrzsgyxt8y8xacxut9qv3fpyetz8t7tsymygq8yzn05",
    "btc_address": "bc1p5uvtaxzkjwvey2tfy49k5vtqfpjmrgm09cvs88ezyy8h2zv7jhas9tu4yr",
    "onchain_payments": [
      {
        "outpoint": "0301e0480b374b32851a9462db29dc19fe830a7f7d7a88b81612b9d42099c0ae:1",
        "satoshi": "1200",
        "confirmed": false
           }
    ]
  },
  "channel": null
}
```

**User**
- MUST validate `onchain_fee_rate` because the server may have changed the value depending on the onchain fee environment. 
- SHOULD validate the `fee_total_satoshi` is reasonable.
- SHOULD validate `fee_total_satoshi` + `local_balance_satoshi` = `order_total_satoshi`.
- MAY abort the flow after.

**Errors**

- 400 Bad request - Request body validation error.

### 3. Payment

This section describes the payment object returned by `POST /lsp/channel` and `GET /lsp/channel/{id}`. The user MUST pay the `lightning_invoice` OR the `btc_address`. Using both methods may lead to the loss of funds.

Example payment object:
```json
"payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_satoshi": "8888",
    "order_total_satoshi": "2008888",
    "lightning_invoice": "lnbc252u1p3aht9ysp580g4633gd2x9lc5al0wd8wx0mpn97...",
    "btc_address": "bc1p5uvtaxzkjwvey2tfy49k5vtqfpjmrgm09cvs88ezyy8h2zv7jhas9tu4yr",
    "onchain_payments": [
        {
        "outpoint": "0301e0480b374b32851a9462db29dc19fe830a7f7d7a88b81612b9d42099c0ae:1",
        "satoshi": "1200",
        "confirmed": false
            }
    ]
},
```

- `state` MUST be one of these values:
    - `EXPECT_PAYMENT` Payment expected.
    - `HOLD` Lighting payment arrived, preimage NOT released.
    - `PAID` Lightning payment arrived, preimage released OR full `order_total_satoshi` onchain payment arrived.
- `fee_total_satoshi` MUST be the total fee the LSP will charge to open this channel in satoshi.
- `order_total_satoshi` MUST be the fee_total_satoshi plus the local_balance_satoshi requested in satoshi.
- `ln_invoice` 
    - MUST be a Lightning Bolt 11 for order_total_satoshi. 
    - Invoice MUST be a [HOLD invoice](https://bitcoinops.org/en/topics/hold-invoices/).
    - The `ln_invoice` MUST give the same amount as in `order_total`.
- `btc_address` 
    - MUST be a bitcoin address the user can pay the order_total to if `base_api.min_required_onchain_satoshi` is above or equal order_total. 
            - MAY be a bech32 version 0 ("SegWit") or bech32m version 1 ("Taproot") address. The server SHOULD NOT provide any other address type. 
            - The client MAY support other address types.
    - MUST be null if `base_api.min_required_onchain_satoshi` is null and therefore unsupported.
- `onchain_payments` 
    - MUST contain all incoming/confirmed outpoints to btc_address. 
    - `outpoint` MUST be an outpoint in the form of [txid:vout](https://btcinformation.org/en/glossary/outpoint).
    - `satoshi` MUST contain the received satoshi as string.
    - `confirmed` MUST contain a boolean if the LSP sees the transaction as confirmed. This MAY be instantly (zeroconf) or MAY happen after the required block confirmations.
    - MUST always be `[]` if `base_api.min_required_onchain_satoshi` is null and therefore unsupported.



#### 3.1 Lightning Payment Flow

**User**

- MUST pay the `lightning_invoice`.
- SHOULD pull `GET /lsp/channel/{id}` to check the success of the payment.
- The user gets refunded automatically in case the channel open fails.

**LSP**

- MUST change the payment state to `HOLD` when the payment arrives.
- MUST set open.state to `PENDING`.
- MUST start then channel open flow.
- If the channel has been opened successfully
    - MUST release the preimage and therefore complete the payment.
    - MUST set the payment state to `PAID`.
- If the payment times out and the channel has NOT been opened
    - MUST reject the payment.
    - MUST set the payment state to `EXPECT_PAYMENT`.
- If the order expired and the channel has NOT been opened
    - MUST reject the payment.
    - MUST set the payment state to `FAILED`.


#### 3.2 Onchain Payment Flow

**User**

- MUST pay `order_total_satoshi` to `btc_address`.
- SHOULD pull `GET /lsp/channel/{id}` to check the success of the payment.

**LSP**

- MUST monitor the blockchain and update `onchain_payments`.
- MUST decide when transactions are confirmed and set them as `confirmed: true`.
- MUST change the payment state to `PAID` when the payment is confirmed.
- MUST set open.state to `PENDING`.
- MUST start then channel open flow.


TODO: Onchain refunds.

### 4 Channel Open

After the open.state switched to `PENDING` AND the user provided `node_id` and `announce`, the LSP MUST attempt a channel open.


#### 4.1 Establish Peer Connection

**User**

- SHOULD establish a peer connection with one of the provided node connection strings in `lsp_node_connection_strings`.
- MUST establish a peer connection to `lsp_node_connection_strings` if the node is not publicly available, for example behind a NAT.
- MUST establish a peer connection to `lsp_node_connection_strings` if `node_connection_string_or_pubkey` only contains a pubkey.


**LSP**

- MUST establish a peer connection to node_connection_string_or_pubkey IF the user provided a connection string.
- MUST wait on the user establishing a peer connection IF the user only provided a pubkey.
- MAY establish a peer connection to node_connection_string_or_pubkey IF the user only provided a pubkey but the connection information is available in the Lightning Gossip.
- MAY try multiple times.

In case the connection attempt failed
- MUST set open.state to `FAILED` and open.fail_reason to `PEERING_FAILED`.

#### 4.1 Open attempt

**LSP**

- MUST attempt a channel open to `node_connection_string_or_pubkey`.
    - MUST respect the `announce` flag.
    - MUST open the channel with a capacity of `remote_balance_satoshi` + `local_balance_satoshi`.
        - MAY overprovision.
    - MUST push `local_balance_satoshi` to the user.
    - MUST use `onchain_fee_rate` or higher.

In case the channel open succeeded
- MUST set open.state to `SUCCESS` and open.fail_reason to `null`.
- MUST update the channel object.

In case the channel open failed
- MUST set open.state to `FAILED` and open.fail_reason to `CHANNEL_REJECTED_BY_DESTINATION`.


### 5 Update order

**User**
- MAY update the `node_id` and `announce` property at any time before the channel is opened.
    - MUST call `POST /lsp/channel/{id}/update` to update the properties.

> **Rational** The user may provide this information only after the order has been created OR the user may need to correct the provided connection string.

**LSP**



- `channel` must contain the opened channel information. Must be null if the channel opening transaction has not been published yet.
    - `state` Must be one of these values:
        - `OPENING` Opening transaction published.
        - `OPENED` Channel is open. LSP must allow payments. May be zero conf and therefore immediately.
        - `CLOSED` Closing transaction has been published.
    - `opened_at` must be a ISO8601 datetime when the opening transaction has been published.
    - `open_transaction` must be the txid:vout of the opening transaction.
    - `scid` must be the short channel id. Must be null before the channel is confirmed.
    - `expires_at` must be a ISO8601 datetime when the channel may be closed by the LSP. Must respect channel_expiry_weeks.
    - `closing_transaction` must be the txid:vout of the closing transaction.
    - `closed_at` must be a ISO8601 datetime when the closing transaction has been published.
    - `node_id` must be the node id of the user node.
    - `lsp_node_id` must be the node id if the lsp node.

    




### 3. Pay order

The client must either pay the `ln_invoice` OR `btc_address`.

**ln_invoice** 
1. Client must pay the `ln_invoice`. 
2. The LSP must HOLD the payment and only release the preimage if the channel is opened successfully.
3. The LSP must change the `payment.status` to `PAID` when receiving the payment.
4. In case the channel has not been opened, the LSP must reject the payment either before the timeout or when the order expires_at. The LSP sets `payment.status` to `REFUNDED`.


**btc_address** 
1. Client must pay the number of satoshi defined in order_total to the btc_address.
2. LSP must listen for payments to btc_address and decided how many block confirmations are required to accept the payment. Zeroconf allowed.
3. LSP must change the payment.status to `PAID` when the required number of satoshi has been confirmed.
4. In case the channel has not been opened, the LSP must set `payment.status` to `REFUND_AVAILABLE`.


### 4. Wait on payment confirmation




---

2. Client must call `POST /lsp/channels` with the values demanded to create an order. The response includes prices, a lightning invoice and a bitcoin address in case the extension `onchain_payment` is supported.
3. Client may pay the order with their method of choice before the order expires.
4. Client should pull `GET /lsp/channels/{id}` to determine the status of their payment. As soon as `payment.state` switches to `PAID` they can attempt a channel open.
5. Client must call `POST /lsp/channels/{id}/open` with the nessecary parameters for a channel open. The LSP must directly attempt a channel open and responds with the result. The client gets direct feedback if the attempt succeeded.
6. The `channel` object in `GET /lsp/channels/{id}` must show the state of the channel. `expiry_ts` shows when the LSP is allowed to close the channel again.


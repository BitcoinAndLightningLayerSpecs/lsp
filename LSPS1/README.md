# LSPS1 Channel Request

| Name    	| channel_request               |
|---------	|------------------------------	|
| Version 	| 2                             |
| Status    | For Implementation            |


## Motivation

The goal of this specification is to provide a standardized LSP API for wallets to purchase a channel from the LSP directly.

## Definitions

### Datetime

All datetimes are represented as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) in the form of `2023-02-23T08:47:30.511Z` UTC.

> **Rationale** ISO8601 is human readable, it's a standard endorsed by [W3C](http://www.w3.org/TR/NOTE-datetime), and [RFC3339](https://www.rfc-editor.org/rfc/rfc3339), sortable, and supported widely.

### Satoshi

All satoshi values MUST be represented as a string and NOT integer values. Make sure you check your integer limits to avoid overflows.

> **Rationale** Plenty of json parsers use a 32bit signed integer for integer parsing. Max safe value is 2,147,483,647; 2,147,483,647sat = BTC21.474,836,47 which is too low.

### Node id

- MUST be a node id (pubkey). For example: "0200000000a3eff613189ca6c4070c89206ad658e286751eca1f29262948247a5f". See [BOLT08](https://github.com/lightning/bolts/blob/master/08-transport.md?plain=1#L5).

### Onchain address

- All onchain addresses MUST be a [bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki) version 0 ("SegWit") or [bech32m](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki) version 1 ("Taproot") address or null.
- The lsp/client MAY support other address types.

> **Rationale** Onchain addresses MUST be null in case onchain payments are unsupported. This MUST be clearly indicated though. See options dictionary.

### Actors

`LSP` is the API provider. `Client` is the client of the API.

### Channel sides

`client_balance` are the funds on the client side. `lsp_balance` are the funds on the LSP side.

### Overprovisioning

The LSP is allowed to overprovision channels/onchain-payments/onchain-fees as long as it benefits the client.

> **Rationale** The lsp may need to "bin" UTXOs.

### Errors

LSP will return errors according to the JSONRPC 2.0 specification (see [LSPS0 Error Handling](https://github.com/BitcoinAndLightningLayerSpecs/lsp/tree/main/LSPS0#error-handling)).

**LSP**
- MAY return an `code` that is not listed above.

**Client**
- MUST be able to handle an `code` that is not listed
  above.
  - SHOULD report an unrecognized error code simply as
    "unrecognized error code".
  - **Rationale** This allows clients written for older versions
    of this specification to work with LSPs written for newer
    versions.
- SHOULD NOT display an unrecognized `code`, `message` or `data` to the user.
  **Rationale** Unrecognized `code`s may be misinterpreted
  or misunderstood by users.

Any error MAY include a `message` field that is a human-readable string. The `message` field is intended to provide a short description of the error. The `message` field is not intended to be parsed by the client or shown to the user. It is intended as a developer hint to help the client developer debug the error.

## Overview

1. Client calls `lsps1.info` to get the LSP's API version and options.
2. Client calls `lsps1.create_order` to create an order.
3. Client pays the order either onchain or offchain.
4. LSP opens the channel as soon as they payment is confirmed.
5. LSP refunds the client in case the channel open failed.

## API

### 1. lsps1.info

| JSONRPC Method | lsps1.info |
|--------        |------------|
| Idempotent     | Yes        |


`lsps1.info` is the entrypoint for each client using the api. It lists the supported versions of the api and all options in a dictionary.
The client MUST call `lsps1.info` first.

**Response** 

```JSON
{
  "versions": [2],
  "website": "http://example.com/contact",
  "options": {
      "minimum_depth": 0,
      "supports_zero_channel_reserve": true,
      "min_required_onchain_satoshi": null,
      "max_channel_expiry_blocks": 20160,
      "min_client_balance_satoshi": "20000",
      "max_client_balance_satoshi": "100000000",
      "min_lsp_balance_satoshi": "0",
      "max_lsp_balance_satoshi": "100000000",
      "min_channel_balance_satoshi": "50000",
      "max_channel_balance_satoshi": "100000000"
  }
}
```



- `versions` *List of integers*: List of all supported API versions by the LSP.
  - Client MUST compare the version of the api and therefore ensure compatibility.
- `website` *string*: Website of the LSP.
- `options` *dictionary*: Dictionary of all options supported by the LSP.
  - `minimum_depth` *integer*: Number of blocks it requires to send `channel_ready` (previously `funding_locked`, aka number of confirmations).
    - MAY be 0 to allow 0conf channels.
    - MUST be 0 or greater.
  - `supports_zero_channel_reserve` *boolean*: Indicates if the LSP supports [zeroreserve](https://github.com/ElementsProject/lightning/pull/5315).
  - `min_required_onchain_satoshi` *satoshi or null*: Indicates the minimum amount of satoshi (`order_total_satoshi` see below) that is required for the LSP to accept a payment onchain.
    - The LSP MUST allow onchain payments equal or above this value. 
    - MUST be 0 or greater.
    - MAY be null if onchain payments are NOT supported.
  - `max_channel_expiry_blocks` *integer*: The maximum number of blocks a channel can be leased for.
    - MUST be 1 or greater.
  - `min_client_balance_satoshi` *satoshi*: Minimum number of satoshi that the client MUST request.
    - MUST be 0 or greater.
  - `max_client_balance_satoshi` *satoshi*: Maximum number of satoshi that the client MUST request.
    - MUST be 0 or greater.
  - `min_lsp_balance_satoshi` *satoshi*: Minimum number of satoshi that the LSP will provide to the channel.
    - MUST be 0 or greater.
  - `max_lsp_balance_satoshi` *satoshi*: Maximum number of satoshi that the LSP will provide to the channel.
    - MUST be 0 or greater.
  - `min_channel_balance_satoshi` *satoshi*: Minimal channel size calculated by the sum of the requested `client_balance_satoshi` and `lsp_balance_satoshi`.
    - MUST be 0 or greater.
  - `max_channel_balance_satoshi` *satoshi*: Maximum channel size calculated by the sum of the requested `client_balance_satoshi` and `lsp_balance_satoshi`
    - MUST be 0 or greater.

Every `min/max` options pair MUST ensure that `min <= max`.


**Errors** No additional errors are defined for this method.

### 2. lsps1.create_order 

| JSONRPC Method     | lsps1.create_order |
|--------            |------------        |
| Idempotent         | No                 |


The request is constructed depending on the client's needs. 

**Request**

```json
{
  "api_version": 2,
  "order": {
    "lsp_balance_satoshi": "5000000",
    "client_balance_satoshi": "2000000",
    "confirms_within_blocks": 1,
    "channel_expiry_blocks": 144,
    "coupon_code": "",
    "refund_onchain_address": "bc1qvmsy0f3yyes6z9jvddk8xqwznndmdwapvrc0xrmhd3vqj5rhdrrq6hz49h"
  },
  "open": {
    "announce": true,
    "node_id": "03d4e028a0d4a90868ec202ab684fb0085779defea9ca7553e06146557631eec20"
  }
}
```



- `api_version` *integer* API version that the client wants to work with.
  - MUST be `2` for this version of the spec. 
  - MUST match one of the versions listed in `lsps1.info.versions`.
- `order` MUST be provided.
    - `lsp_balance_satoshi` *satoshi* How many satoshi the LSP will provide on their side.
      - MUST be 1 or greater. 
      - MUST be equal or below `base_api.max_lsp_balance_satoshi`.
      - MUST be equal or greater `base_api.min_lsp_balance_satoshi`.
    - `client_balance_satoshi` *satoshi* How many satoshi the client will provide on their side. The client send these funds to the LSP. The LSP will push these funds back to the client.
      - MUST be 0 or greater. 
      - MUST be below or equal `base_api.max_client_balance_satoshi`.
      - MUST be greater or equal `base_api.min_client_balance_satoshi`.
    - `confirms_within_blocks` *integer* Number of blocks the client wants to wait maximally for the channel to be confirmed.
      - MUST be 0 or greater.
      - LSP MAY always confirm the channel faster than requested.
    - `channel_expiry_blocks` *integer* How long the channel is leased for in block time.
      - MUST be 1 or greater. 
      - MUST be below or equal `base_api.max_channel_expiry_blocks`.
    - `coupon_code` *string* Code that the client wants to use to claim a discount.
      - Client MAY omit this field.
    - `refund_onchain_address` *Onchain address* Address where the LSP will send the funds if the order fails.
      - Client MAY omit this field.
      - LSP MUST disable onchain payments if the client did omit this field.
- `open` MUST be provided.
    - `announce` *boolean* If the channel should be announced to the network (also known as public).
    - `node_id` *node_id* LSP will open the channel to this node.


> **Rationale client_balance_satoshi** Client MAY want to have initial spending balance on their wallet or start with a balanced channel.

> **Rationale coupon_code** Client MAY provide a coupon_code to claim a discount on the order. 

The client MUST check if [option_support_large_channel](https://bitcoinops.org/en/topics/large-channels/) is enabled before they order a channel larger than 16,777,216 satoshi.

**Response**

```json
{
  "id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c",
  "api_version": 2,
  "state": "AWAITING_PAYMENT", ?????
  "lsp_balance_satoshi": "5000000",
  "client_balance_satoshi": "2000000",
  "confirms_within_blocks": 1,
  "channel_expiry_blocks": 12,
  "coupon_code": "",
  "created_at": "2012-04-23T18:25:43.511Z",
  "expires_at": "2015-01-25T19:29:44.612Z",
  "open": {
    "announce": true,
    "node_id": "03d4e028a0d4a90868ec202ab684fb0085779defea9ca7553e06146557631eec20",
    "state": "PENDING",
    "fail_reason": null
  },
  "payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_satoshi": "8888",
    "order_total_satoshi": "2008888",
    "lightning_invoice": "lnbc252u1p3aht9ysp580g4633gd2x9lc5al0wd8wx0mpn9748jeyz46kqjrpxn52uhfpjqpp5qgf67tcqmuqehzgjm8mzya90h73deafvr4m5705l5u5l4r05l8cqdpud3h8ymm4w3jhytnpwpczqmt0de6xsmre2pkxzm3qydmkzdjrdev9s7zhgfaqxqyjw5qcqpjrzjqt6xptnd85lpqnu2lefq4cx070v5cdwzh2xlvmdgnu7gqp4zvkus5zapryqqx9qqqyqqqqqqqqqqqcsq9q9qyysgqen77vu8xqjelum24hgjpgfdgfgx4q0nehhalcmuggt32japhjuksq9jv6eksjfnppm4hrzsgyxt8y8xacxut9qv3fpyetz8t7tsymygq8yzn05",
    "onchain_address": "bc1p5uvtaxzkjwvey2tfy49k5vtqfpjmrgm09cvs88ezyy8h2zv7jhas9tu4yr",
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

- `id` *string* An lsp generated order id.
  - MUST be unique.
- `api_version` *integer* Version of the api that has been used to create the order.
- `lsp_balance_satoshi` *satoshi* Mirrored from the request.
- `client_balance_satoshi` *satoshi* Mirrored from the request.
- `confirms_within_blocks` *integer* Mirrored from the request.
- `channel_expiry_blocks` *integer* Mirrored from the request.
- `coupon_code` *string* Mirrored from the request.
  - MUST be an empty string if the coupon_code was not provided.
- `created_at` *datetime* Datetime when the order was created.
- `expires_at` *datetime* Datetime when the order expires.
- `open` describes channel open information.
  - `announce` *boolean* Mirrored from the request.
  - `node_id` *node_id* Mirrored from the request.
  - `state` *string enum* Current state of the channel open.
    - `PENDING` LSP is waiting for the client to pay.
    - `OPENING` LSP is opening the channel.
    - `OPEN` Channel is open.
    - `FAILED` Channel open failed.


**Client**
- SHOULD validate the `fee_total_satoshi` is reasonable.
- SHOULD validate `fee_total_satoshi` + `client_balance_satoshi` = `order_total_satoshi`.
- MAY abort the flow here.

**Errors**

| Code   | Message         | Data | Description |
| ----   | -------         | ----------- | ---- |
| -32602 | Invalid params  | {"property": %invalid_property%, "message": %human_message% }    | Invalid method parameter(s). |
| 1000   | Option mismatch |  {"property": %option_mismatch_property%, "message": %human_message% }   | The order doesnt match the options defined in `lsps1.info.options`. |
| 1001   | Client rejected |  {"message": %human_message% }   | The LSP rejected the client. |

- LSP MUST validate the order against the options defined in `lsps1.info.options`. LSP MUST return an `1000` error in case of a mismatch.
  - %option_mismatch_property% MUST be one of the fields in `lsps1.info.options`.
  - Example: `{ "property": "min_client_balance_satoshi" }`.

- LSP MUST validate the request fields. LSP MUST return a `-32602` error in case of an invalid request field.
  - %invalid_property% MUST be one of the fields in the request body. MUST use `.` to separate nested fields.
  - Example: `{ "property": "open.node_id", "message": "Invalid pubkey" }`.

- LSP MUST validate the `coupon_code` field and return an error if the coupon is invalid.

> **Rationale coupon_code validation** The client should be informed if the coupon code is invalid. Ignoring the invalid code and creating an invoice without the discount is not good UX. Ignoring the invalid code will also NOT prevent anybody bruteforcing the coupon code because the client will still detect if the LSP has given a discount.

- LSP MAY reject a client by it's node_id or IP. In this case, the LSP MUST return a `1001` error.
  - %human_message% MAY simply be "Client rejected".
  - Example: `{ "message": "Node id banned." }`.


### 2.1 lsps1.get_order 

| JSONRPC Method | lsps1.get_order |
|--------        |------------     |
| Idempotent     | Yes             |

The client MAY check the current status of the order at any point.

**Request**

```json
{
  "order_id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c"
}
```

**Response** is the same as defined in `lsps1.create_order`.

**Errors**

| Code   | Message | Data      | Description |
| ----   | ------- | ---       | ----------- |
| 404    | Not found  | {}        | Order with the requested order_id has not been found. |


### 3. Payment

This section describes the payment object returned by `lsps1.create_order` and `lsps1.get_order`. The client MUST pay the `lightning_invoice` OR the `onchain_address`. Using both methods MAY lead to the loss of funds.

> **Rationale** Onchain Payments are required for payments with higher amounts, especially to push client_balance_satoshi to the client. Onchain payments are also useful to onboard new users to Lightining. Lightning payments are the preferred way to do payments because they are quick and easily refundable.

**Before the payment**

**Payment object**

```json
"payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_satoshi": "8888",
    "order_total_satoshi": "2008888",
    "lightning_invoice": "lnbc252u1p3aht9ysp580g4633gd2x9lc5al0wd8wx0mpn97...",
    "onchain_address": "bc1p5uvtaxzkjwvey2tfy49k5vtqfpjmrgm09cvs88ezyy8h2zv7jhas9tu4yr",
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
    - `REFUNDED` Lightning payment or onchain payment has been refunded.
- `fee_total_satoshi` MUST be the total fee the LSP will charge to open this channel in satoshi.
- `order_total_satoshi` MUST be the fee_total_satoshi plus the client_balance_satoshi requested in satoshi.
- `ln_invoice` 
    - MUST be a Lightning BOLT 11 invoice for an amount of `order_total_satoshi`. 
    - Invoice MUST be a [HOLD invoice](https://bitcoinops.org/en/topics/hold-invoices/).
    - The `ln_invoice` MUST give the same amount as in `order_total_satoshi`.
- `onchain_address` 
    - MUST be an onchain address the client can pay the order_total_satoshi to.
    - MUST be null 
      - if `options.min_required_onchain_satoshi` is above or equal order_total.
      - if `options.min_required_onchain_satoshi` is null and therefore not supported.
      - if `refund_onchain_address` is null.
- `onchain_address_confirms` 
    - MUST be the number of confirmations that the server will accept before considering the payment as confirmed. If 0, this indicates that the server accepts 0-confirmation payments to this address.
    - MUST be positive or 0.
- `onchain_payments`
    - MUST contain all incoming/confirmed outpoints to onchain_address. 
    - `outpoint` MUST be an outpoint in the form of [txid:vout](https://btcinformation.org/en/glossary/outpoint).
    - `satoshi` MUST contain the received satoshi.
    - `confirmed` MUST contain a boolean if the LSP sees the transaction as confirmed.


#### 3.1 Lightning Payment

**Client**

- MUST pay the `lightning_invoice`.
- SHOULD pull `lsps1.get_order` to check the success of the payment.
- The client gets refunded automatically in case the channel open failed, the order expired, or the payment timed out.

**LSP**

- MUST change the payment state to `HOLD` when the payment arrived.
- MUST set open.state to `PENDING`.
- If the channel has been opened successfully
    - MUST release the preimage and therefore complete the payment.
    - MUST set the payment state to `PAID`.
- If the payment times out and the channel failed to open or the order expired:
    - MUST reject the payment.
    - MUST set the payment state to `REFUNDED`.



#### 3.2 Onchain Payment Flow

**Client**

- MUST pay `order_total_satoshi` to `onchain_address`.
- MAY pull `lsps1.get_order` to check the success of the payment.

**LSP**

- MUST monitor the blockchain and update `onchain_payments`.
- MUST set the transaction as confirmed after `onchain_address_confirms` confirmations.
- MUST change the payment state to `PAID` when all the payments for the order are confirmed.
- MUST set open.state to `PENDING`.
- If the order expired and the channel has NOT been opened, OR the channel open failed.
    - MUST refund the client to `refund_onchain_address`.
      - The number of satoshi to refund 
        - MUST be `order_total_satoshi` MINUS transaction size * onchain_fee_rate. The LSP MUST choose a reasonable fee rate.
        - MAY overprovision.
      - The LSP MUST bump the fees in case the transaction doesn't resolve within 6hrs.
    - MUST set the payment state to `REFUNDED`.



### 4 Channel Open

The LSP MUST open the channel under the following conditions:
- The open.state switched to `PENDING`

**LSP**
- MUST wait for a peer connection before attempting a channel open.
- MUST attempt a channel open to `node_id`.
    - MUST respect the `announce` flag.
    - MUST open the channel with a capacity of `lsp_balance_satoshi` + `client_balance_satoshi`.
        - MAY overprovision.
    - MUST push `client_balance_satoshi` to the client.
        - MAY overprovision.
    - MUST use a high enough onchain fee rate to ensure the funding transaction confirms within `confirms_within_blocks` after the client paid the order.
        - MAY overprovision.
- MUST send `channel_ready` after the funding transaction has `minimum_depth` confirmations.
- MUST allow zero channel reserves if `supports_zero_channel_reserve`.

In case the channel open succeeded
- MUST set open.state to `SUCCESS` and open.fail_reason to `null`.
- MUST update the channel object.

In case the channel open failed
- MUST set open.state to `FAILED` and open.fail_reason to `CHANNEL_REJECTED_BY_DESTINATION`.


##### Batching

**LSP**

- MAY open channels in batches, opening multiple channels in one transaction.
  - In this case, the LSP MUST still ensure that the funding transaction gets confirmed after a maximum of `confirms_within_blocks` after the client payment completed.


### 5 Channel Object

Todo: Describe channel object. Might be simplified or even unnecessary.


- `channel` MUST contain the opened channel information. MUST be null if the channel opening transaction has not been published yet.
    - `state` Must be one of these values:
        - `OPENING` Opening transaction published.
        - `OPENED` Channel is open. LSP must allow payments. May be zero conf and therefore immediately.
        - `CLOSED` Closing transaction has been published.
    - `opened_at` MUST be a datetime when the opening transaction has been published.
    - `open_transaction` MUST be the id the opening transaction.
    - `scid` MUST be the short channel id. MUST be null before the channel is confirmed onchain.
    - `expires_at` MUST be a datetime when the channel may be closed by the LSP. MUST respect `channel_expiry_blocks`. MAY overprovision.
    - `closing_transaction` MUST be the id of the closing transaction.
    - `closed_at` MUST be a datetime when the closing transaction has been published.
    - `client_pubkey` MUST be the node id of the client node.
    - `lsp_pubkey` MUST be the node id if the lsp node.


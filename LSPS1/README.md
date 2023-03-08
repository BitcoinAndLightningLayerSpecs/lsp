# LSPS1 channel request

| Name    	| base_api                      |
|---------	|------------------------------	|
| Version 	| 2                             |
| OpenApi 	| [LSPS1.yaml](./LSPS1.yaml) 	|



## Motivation

The goal of this specification is to provide a standardized LSP API to purchase channels. Two client types are supported:

**Wallets** that purchase a channel from the LSP directly.


## OpenAPI

All specs are defined in the [OpenAPI](https://www.openapis.org/about) format. It can be viewed with various editors, most notably the [Swagger Editor](https://editor.swagger.io/) or the [vscode-openapi](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi) extension. All the OpenAPI documents include detailed descriptions of every field.

## Definitions

### Datetime

All datetimes are represented as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) in the form of `2023-02-23T08:47:30.511Z` UTC.

> **Rationale** ISO8601 is human readable, it's a standard endorsed by [W3C](http://www.w3.org/TR/NOTE-datetime), and [RFC3339](https://www.rfc-editor.org/rfc/rfc3339), sortable, and supported widely.

### Satoshi

All satoshi values MUST be represented as a string and NOT integer values. Make sure you check your integer limits to avoid overflows.

> **Rationale** Plenty of json parsers use a 32bit signed integer for integer parsing. Max safe value is 2,147,483,647; 2,147,483,647sat = BTC21.474,836,47 which is too low.

### Actors

`LSP` is the API provider. `User` is the user of the API.

### Channel sides

Channel sides are seen from the user point of view. `user_balance` are the funds on the user side. `lsp_balance` are the funds on the LSP side.

### Sync vs Async Channel Open

This spec supports two ways to open a channel:

**Sync** The channel is opened by calling `POST /lsp/channel/{id}/open`. The user node_id is provided with the open call.
Success/error messages are provided in the call response.

**Async** The channel is opened as soon as the payment is confirmed. Success/error messages are provided with `GET /lsp/channel/{id}`.

## Flow

### API information

`GET /lsp/channels` is the entrypoint for each client using the api. It lists the version of the api and all supported extensions in a dictionary.

The user MUST pull `GET /lsp/channels` to read

- the `version` of the api AND version of `options` and therefore prove compatibility.
- `options` properties to determine the api boundries.

Example `GET /lsp/channels` response: 
```JSON
{
  "version": 2,
  "website": "http://example.com/contact",
  "extensions": {
    "base_api": {
      "version": 1,
      "max_user_balance_satoshi": "0",
      "max_lsp_balance_satoshi": "100000000",
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
        "max_user_balance_satoshi": "0",
        "max_lsp_balance_satoshi": "100000000",
        "min_required_onchain_satoshi": null,
        "max_channel_expiry_weeks": 24
    }
}
```

- `version` MUST be 1.
- `max_user_balance_satoshi` MUST be the maximum number of satoshi that the LSP is willing to push to the user. MUST be 0 or a positive integer.
- `max_lsp_balance_satoshi` MUST be the maximum number of satoshi that the LSP is willing to contribute to the their balance.  MUST be 1 or greater.
- `min_required_onchain_satoshi` MUST be the number of satoshi (`order_total_satoshi` see below) that are required for the user to pay funds onchain. The LSP MUST allow onchain payments equal or above this value. MAY be null if onchain payments are NOT supported.
- `max_channel_expiry_weeks` MUST be the maximum length in weeks a channel can be leased for. MUST be 1 or greater.

The user MAY abort the flow here.

### 2. Create Order

The user constructs the request body depending on their needs. 

- The user MUST check if [option_support_large_channel](https://bitcoinops.org/en/topics/large-channels/) is enabled before they order a channel larger than BTC0.16777216.
- The user MUST call `POST /lsp/channels` to create an order.

**Example request body**

```json
{
  "order": {
    "lsp_balance_satoshi": "5000000",
    "user_balance_satoshi": "2000000",
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

- `order` object MUST be provided.
    - `lsp_balance_satoshi` MUST be 1 or greater. MUST be below or equal `base_api.max_lsp_balance_satoshi`.
    - `user_balance_satoshi` MUST be 0 or greater. MUST be below or equal `base_api.max_user_balance_satoshi`. Todo: Rejection error message.
    - `onchain_fee_rate` MUST be 1 or higher. The LSP may increase this value depending on the onchain fee environment. MAY be unspecified, the LSP will determine the fee rate. 
    - `channel_expiry_weeks` MUST be 1 or greater. MUST be below or equal `base_api.max_channel_expiry_weeks`.
    - `coupon_code` MUST be a string, null, or not defined at all.
- `open` determines if the channel open is Sync or Async. MUST be provided if Asyc. MUST be null if Sync.
    - `announce` If the channel should be announced to the network. MUST be boolean.
    - `node_connection_string_or_pubkey` MUST be a node_connection_string or a pubkey.



> **Rationale user_balance_satoshi** User may want to have initial spending balance on their wallet or start with a balanced channel.


**Example response body**

HTTP Code: 201 CREATED

```json
{
  "id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c",
  "state": "AWAITING_PAYMENT",
  "lsp_balance_satoshi": "5000000",
  "user_balance_satoshi": "2000000",
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
- MUST validate `onchain_fee_rate` because the server may have increased the value depending on the onchain fee environment. 
- SHOULD validate the `fee_total_satoshi` is reasonable.
- SHOULD validate `fee_total_satoshi` + `user_balance_satoshi` = `order_total_satoshi`.
- MAY abort the flow after.

**Errors**

- 400 Bad request - Request body validation error.

Todo: Define error type better. [Zmn proposal](https://github.com/BitcoinAndLightningLayerSpecs/lsp/pull/21/files#diff-603325abb5c270c90ec7c4c60eec7cb1aae620a8155519c65f974ba33ee63c54R346)

### 3. Payment

This section describes the payment object returned by `POST /lsp/channel` and `GET /lsp/channel/{id}`. The user MUST pay the `lightning_invoice` OR the `btc_address`. Using both methods MAY lead to the loss of funds.

> **Rationale** Onchain Payments are required for payments with higher amounts, especially to push user_balance_satoshi to the user. Onchain payments are also useful to onboard new user to Lightining. Lightning payments are the preferred way to do payments because they are quick and easily refundable.

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
- `order_total_satoshi` MUST be the fee_total_satoshi plus the user_balance_satoshi requested in satoshi.
- `ln_invoice` 
    - MUST be a Lightning Bolt 11 for order_total_satoshi. 
    - Invoice MUST be a [HOLD invoice](https://bitcoinops.org/en/topics/hold-invoices/).
    - The `ln_invoice` MUST give the same amount as in `order_total_satoshi`.
- `btc_address` 
    - MUST be a bitcoin address the user can pay the order_total_satoshi to if `base_api.min_required_onchain_satoshi` is above or equal order_total. 
            - MAY be a bech32 version 0 ("SegWit") or bech32m version 1 ("Taproot") address. The server SHOULD NOT provide any other address type. 
            - The client MAY support other address types.
    - MUST be null if `base_api.min_required_onchain_satoshi` is null and therefore not supported.
- `btc_address_confirms` MUST be the number of confirmations that the server
will accept before considering the payment as confirmed.
    - If 0, this indicates that the server accepts 0-confirmation payments to
this address.
    - MUST be positive or 0.
- `onchain_payments` 
    - MUST contain all incoming/confirmed outpoints to btc_address. 
    - `outpoint` MUST be an outpoint in the form of [txid:vout](https://btcinformation.org/en/glossary/outpoint).
    - `satoshi` MUST contain the received satoshi.
    - `confirmed` MUST contain a boolean if the LSP sees the transaction as confirmed.
    - MUST always be `[]` if `base_api.min_required_onchain_satoshi` is null and therefore unsupported.



#### 3.1 Lightning Payment Flow

**User**

- MUST pay the `lightning_invoice`.
- SHOULD pull `GET /lsp/channel/{id}` to check the success of the payment.
- The user gets refunded automatically in case the channel open failed, the order expired, or the payment timed out.

**LSP**

- MUST change the payment state to `HOLD` when the payment arrives.
- MUST set open.state to `PENDING`.
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
- MUST set the transaction as confirmed after `btc_address_confirms` confirmations.
- MUST change the payment state to `PAID` when the payment is confirmed.
- MUST set open.state to `PENDING`.


TODO: Onchain refunds. Ideas: 
- (Async) User provides a refund_btc_address at the order creation.
- (Sync) User provides a refund_btc_address afterwards.


### 4 Channel Open

The LSP MUST open the channel under the following conditions

**Async** 
- The open.state switched to `PENDING`

**Sync**
- The open.state switched to `PENDING` AND the user calls `POST /lsp/channel/{id}/open`.



#### 4.1 Establish Peer Connection

**User**

If `node_connection_string_or_pubkey` contains a full connection string:
- SHOULD open a peer connection before the channel is being opened.
- SHOULD establish a peer connection with one of the provided node connection strings in `lsp_node_connection_strings`.

If `node_connection_string_or_pubkey` only contains a pubkey OR the node is not publicly available:
- MUST open a peer connection before the channel is being opened.
- MUST establish a peer connection with one of the provided node connection strings in `lsp_node_connection_strings`.


**LSP**


- MUST establish a peer connection to `node_connection_string_or_pubkey` IF the user provided a full connection string.
- MUST wait on the user establishing a peer connection IF the user only provided a pubkey.
- MAY establish a peer connection to `node_connection_string_or_pubkey` IF the user only provided a pubkey and the connection information is available in the Lightning Gossip.
- MAY try multiple times.

In case the connection attempt failed
- MUST set open.state to `FAILED` and open.fail_reason to `PEERING_FAILED`.

#### 4.1 Open attempt

**LSP**

- MUST attempt a channel open to `node_connection_string_or_pubkey`.
    - MUST respect the `announce` flag.
    - MUST open the channel with a capacity of `lsp_balance_satoshi` + `user_balance_satoshi`.
        - MAY overprovision.
    - MUST push `user_balance_satoshi` to the user.
        - MAY overprovision.
    - MUST use `onchain_fee_rate`.
        - MAY overprovision.

In case the channel open succeeded
- MUST set open.state to `SUCCESS` and open.fail_reason to `null`.
- MUST update the channel object.

In case the channel open failed
- MUST set open.state to `FAILED` and open.fail_reason to `CHANNEL_REJECTED_BY_DESTINATION`.

### 5 Channel Object

Todo: Describe channel object. Might be simplified or simply unnecessary.


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

    




### Open Questions

- How do we handle channel batching? [Channel batching by Zmn](https://github.com/BitcoinAndLightningLayerSpecs/lsp/pull/21/files#diff-603325abb5c270c90ec7c4c60eec7cb1aae620a8155519c65f974ba33ee63c54R319)
- How long is the LSP allowed to wait for the channel open (async case)?
- How to handle 0conf channels? [Zmn proposal](https://github.com/BitcoinAndLightningLayerSpecs/lsp/pull/21/files#diff-603325abb5c270c90ec7c4c60eec7cb1aae620a8155519c65f974ba33ee63c54R147).
- Can we make the order stateless? [Zmn proposal](https://github.com/BitcoinAndLightningLayerSpecs/lsp/pull/21/files#diff-603325abb5c270c90ec7c4c60eec7cb1aae620a8155519c65f974ba33ee63c54R269) *Severin: Would be cool. Worst case a DDoS can also be prevented with classic rate limiting.
- LNURL can use `POST /lsp/channel/{id}/open` with additional query parameters.
# LSPS1 Channel Request


| Name    	| `channel_request`             |
|---------- |------------------------------	|
| Version 	| 1                             |
| Status    | For Implementation            |


## Motivation

The goal of this specification is to provide a standardized LSP API for wallets to purchase a channel from the LSP directly.

## Definitions

### Data types

This specification uses data types defined in [LSPS0 Common Schemas][LSPS0.common_schemas].

### Actors

`LSP` is the API provider. `Client` is the client of the API.

### Channel sides

`client_balance` are the funds on the client side. `lsp_balance` are the funds on the LSP side.

### Overprovisioning

The LSP is allowed to overprovision channels/on-chain-payments/on-chain-fees as long as it benefits the client.

> **Rationale** The LSP may need to "bin" UTXOs.

### Errors

LSP will return errors according to the [JSON-RPC 2.0](https://www.jsonrpc.org/specification) specification (see [LSPS0 Error Handling](https://github.com/BitcoinAndLightningLayerSpecs/lsp/tree/main/LSPS0#error-handling)).

## Trust model

In general, LSPS1 is developed on the basis that the client trust the LSP to deliver the promised goods. Channel purchases are not atomic and therefore the client risks not getting the promised goods if the LSP is malicious.

## Order Flow Overview


* Client calls `lsps1.get_info` to get the LSP's options.
* Client calls `lsps1.create_order` to create an order.
* Client pays the order either on-chain or off-chain.
* LSP opens the channel as soon as they payment is confirmed.
  * Channel open failed: LSP refunds the client.


## API

### 1. lsps1.get_info

| JSON-RPC Method | lsps1.get_info |
|---------------- |----------- |
| Idempotent      | Yes        |


`lsps1.get_info` is the entrypoint for each client using the API. It lists all options in a dictionary. 

- The LSP SHOULD NOT change the values in `lsps1.get_info` more than once per day.

> **Rationale Change frequency** The LSP should not change values in `lsps1.get_info` too frequently. Lightning Explorers may scrape these values and provide an overview of all LSPs. If the values change too frequently, Lightning Explorers may not be able to keep up with the changes. Changing them a maximum of once a day gives explorer enough time to scrape. Once a day has been chosen as it is a similar rate-limit that core-lightning puts on the lightning gossip.

The client MUST call `lsps1.get_info` first.

**Request** No parameters needed.

**Response** 

```JSON
{
  "website": "http://example.com/contact",
  "options": {
      "minimum_channel_confirmations": 0,
      "minimum_onchain_payment_confirmations": 1,
      "supports_zero_channel_reserve": true,
      "min_onchain_payment_size_sat": null,
      "max_channel_expiry_blocks": 20160,
      "min_initial_client_balance_sat": "20000",
      "max_initial_client_balance_sat": "100000000",
      "min_initial_lsp_balance_sat": "0",
      "max_initial_lsp_balance_sat": "100000000",
      "min_channel_balance_sat": "50000",
      "max_channel_balance_sat": "100000000"
  }
}
```

- `website <string>` Website of the LSP.
  - MUST be at most 256 characters long.
- `options <object>` All options supported by the LSP.
  - `minimum_channel_confirmations <unit8>` Minimum number of block confirmations before the LSP accepts a channel as confirmed and sends [channel_ready](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md#the-channel_ready-message) (previously `funding_locked`).
    - MAY be 0 to allow 0conf channels.
    - MUST be 0 or greater.
  - `minimum_onchain_payment_confirmations <unit8>` Minimum number of block confirmations before the LSP accepts an on-chain payment as confirmed. This is a lower bound. The LSP MAY increase this value by responding with a different value in `lsps1.create_order.onchain_block_confirmations_required` depending on the size of the channels and risk management.
    - MAY be 0 to allow 0conf payments.
    - MUST be 0 or greater.
  - `supports_zero_channel_reserve <boolean>` Indicates if the LSP supports [zeroreserve](https://github.com/ElementsProject/lightning/pull/5315).
  - `min_onchain_payment_size_sat` <[LSPS0.sat][] or `null`> Indicates the minimum amount of satoshi (`order_total_sat`) that is required for the LSP to accept a payment on-chain.
    - The LSP MUST allow on-chain payments equal or above this value. 
    - MUST be 0 or greater.
    - MAY be `null` if on-chain payments are NOT supported.
  - `max_channel_expiry_blocks <uint32>` The maximum number of blocks a channel can be leased for.
    - MUST be 1 or greater.
  - `min_initial_client_balance_sat` <[LSPS0.sat][]> Minimum number of satoshi that the client MUST request.
    - MUST be 0 or greater.
  - `max_initial_client_balance_sat` <[LSPS0.sat][]> Maximum number of satoshi that the client MUST request.
    - MUST be 0 or greater.
  - `min_initial_lsp_balance_sat` <[LSPS0.sat][]> Minimum number of satoshi that the LSP will provide to the channel.
    - MUST be 0 or greater.
  - `max_initial_lsp_balance_sat` <[LSPS0.sat][]> Maximum number of satoshi that the LSP will provide to the channel.
    - MUST be 0 or greater.
  - `min_channel_balance_sat` <[LSPS0.sat][]> Minimal channel size.
    - MUST be 0 or greater.
  - `max_channel_balance_sat` <[LSPS0.sat][]> Maximum channel size.
    - MUST be 0 or greater.

Every `min/max` options pair MUST ensure that `min <= max`.

**Errors** No additional errors are defined for this method.

### 2. lsps1.create_order 

| JSON-RPC Method     | lsps1.create_order |
|-------------------- |------------------- |
| Idempotent          | No                 |


The request is constructed depending on the client's needs. 

**Request**

```json
{
  "lsp_balance_sat": "5000000",
  "client_balance_sat": "2000000",
  "confirms_within_blocks": 1,
  "channel_expiry_blocks": 144,
  "token": "",
  "refund_onchain_address": "bc1qvmsy0f3yyes6z9jvddk8xqwznndmdwapvrc0xrmhd3vqj5rhdrrq6hz49h",
  "announceChannel": true
}
```


- `lsp_balance_sat` <[LSPS0.sat][]> How many satoshi the LSP will provide on their side.
  - MUST be 1 or greater. 
  - MUST be equal or below `base_api.max_initial_lsp_balance_sat`.
  - MUST be equal or greater `base_api.min_initial_lsp_balance_sat`.
- `client_balance_sat` <[LSPS0.sat][]> How many satoshi the client will provide on their side. The client send these funds to the LSP. The LSP will push these funds back to the client.
  - MUST be 0 or greater. 
  - MUST be below or equal `base_api.max_initial_client_balance_sat`.
  - MUST be greater or equal `base_api.min_initial_client_balance_sat`.
- `confirms_within_blocks <uint8>` Number of blocks the client wants to wait maximally for the channel to be confirmed.
  - MUST be 0 or greater.
  - LSP MAY always confirm the channel faster than requested.
- `channel_expiry_blocks <uint32>` How long the channel is leased for in block time.
  - MUST be 1 or greater. 
  - MUST be below or equal `base_api.max_channel_expiry_blocks`.
- `token <string>` Field for arbitrary data like a coupon code or a authentication token.
  - Client MAY omit this field.
- `refund_onchain_address` <[LSPS0.onchain_address][]> Address where the LSP will send the funds if the order fails.
  - Client MAY omit this field.
  - LSP MUST disable on-chain payments if the client omits this field.
- `announceChannel <boolean>` If the channel should be announced to the network (also known as public/private channels).


> **Rationale `client_balance_sat`** Client MAY want to have initial spending balance on their wallet or start with a balanced channel.


The client MUST check if [option_support_large_channel](https://bitcoinops.org/en/topics/large-channels/) is enabled before they order a channel larger than 16,777,216 satoshi.

**Response**

```json
{
  "order_id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c",
  "lsp_balance_sat": "5000000",
  "client_balance_sat": "2000000",
  "confirms_within_blocks": 1,
  "channel_expiry_blocks": 12,
  "token": "",
  "created_at": "2012-04-23T18:25:43.511Z",
  "expires_at": "2015-01-25T19:29:44.612Z",
  "announceChannel": true,
  "order_state": "CREATED",
  "payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_sat": "8888",
    "order_total_sat": "2008888",
    "lightning_invoice": "lnbc252u1p3aht9ysp580g4633gd2x9lc5al0wd8wx0mpn9748jeyz46kqjrpxn52uhfpjqpp5qgf67tcqmuqehzgjm8mzya90h73deafvr4m5705l5u5l4r05l8cqdpud3h8ymm4w3jhytnpwpczqmt0de6xsmre2pkxzm3qydmkzdjrdev9s7zhgfaqxqyjw5qcqpjrzjqt6xptnd85lpqnu2lefq4cx070v5cdwzh2xlvmdgnu7gqp4zvkus5zapryqqx9qqqyqqqqqqqqqqqcsq9q9qyysgqen77vu8xqjelum24hgjpgfdgfgx4q0nehhalcmuggt32japhjuksq9jv6eksjfnppm4hrzsgyxt8y8xacxut9qv3fpyetz8t7tsymygq8yzn05",
    "onchain_address": "bc1p5uvtaxzkjwvey2tfy49k5vtqfpjmrgm09cvs88ezyy8h2zv7jhas9tu4yr",
    "onchain_block_confirmations_required": 0,
    "onchain_payment": null
  },
  "channel": null
}
```

- `order_id <string>` Id of this specific order.
  - MUST be unique.
  - MUST be at most 64 characters long.
  - SHOULD be a valid [UUID version 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) (aka random UUID).
- `lsp_balance_sat` <[LSPS0.sat][]> Mirrored from the request.
- `client_balance_sat` <[LSPS0.sat][]> Mirrored from the request.
- `confirms_within_blocks <uint8>` Mirrored from the request.
- `channel_expiry_blocks <uint32>` Mirrored from the request.
- `token <string>` Mirrored from the request.
  - MUST be an empty string if the token was not provided.
- `announceChannel <boolean>` Mirrored from the request.
- `created_at` <[LSPS0.datetime][]> Datetime when the order was created.
- `expires_at` <[LSPS0.datetime][]> Datetime when the order expires.
- `order_state <string enum>` Current state of the order.
  - `CREATED` Order has been created. Default value.
  - `COMPLETED` LSP has published funding transaction.
  - `FAILED` Order failed.
- `payment <object>` Contains everything about payments, see [3. Payment](#3-payment).
- `channel <object or null>` Contains information about the channel, see [4 Channel](4-channel).
  - MUST be `null` if the channel funding transaction is not published yet.
  

**Client**
- SHOULD validate the `fee_total_sat` is reasonable.
- SHOULD validate `fee_total_sat` + `client_balance_sat` = `order_total_sat`.
- MAY abort the flow here.

**Errors**

| Code   | Message         | Data | Description |
| ----   | -------         | ----------- | ---- |
| -32602 | Invalid params  | {"property": %invalid_property%, "message": %human_message% }    | Invalid method parameter(s). |
| 1000   | Option mismatch |  {"property": %option_mismatch_property%, "message": %human_message% }   | The order doesnt match the options defined in `lsps1.get_info.options`. |
| 1001   | Client rejected |  {"message": %human_message% }   | The LSP rejected the client. |

- LSP MUST validate the order against the options defined in `lsps1.get_info.options`. LSP MUST return an `1000` error in case of a mismatch.
  - `%option_mismatch_property%` MUST be one of the fields in `lsps1.get_info.options`.
  - Example: `{ "property": "min_initial_client_balance_sat" }`.

- LSP MUST validate the request fields. LSP MUST return a `-32602` error in case of an invalid request field.
  - `%invalid_property%` MUST be one of the fields in the request body. MUST use `.` to separate nested fields.
  - Example: `{ "property": "announceChannel", "message": "Not a boolean" }`.

- LSP MUST validate the `token` field and return an error if the token is invalid.

> **Rationale `token` validation** The client should be informed if the token is invalid. Ignoring the invalid token and creating an order without the potentially discount or other side effect is not good UX. Ignoring the invalid token will also NOT prevent anybody bruteforcing the token because the client will still detect if the LSP has given a discount.

- LSP MAY reject a client by it's node_id or IP. In this case, the LSP MUST return a `1001` error.
  - %human_message% MAY simply be "Client rejected".
  - Example: `{ "message": "Client rejected" }`.

> **Rationale Client rejected** LSPs can reject a client for example for misbehaviour. LSPs can reject a node on two levels: Prevent a peer connection OR disable order creation. Preventing a peer connection might not work in case you still want to allow other functions to keep working, for example an existing channel.


### 2.1 lsps1.get_order 

| JSON-RPC Method | lsps1.get_order |
|---------------- |---------------- |
| Idempotent      | Yes             |

The client MAY check the current status of the order at any point.

**Request**

```json
{
  "order_id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c"
}
```

**Response** is the same as defined in `lsps1.create_order`.

**Errors**

| Code   | Message   | Data    | Description                                           |
| ------ | --------- | ------- | ----------------------------------------------------- |
| 404    | Not found | {}      | Order with the requested order_id has not been found. |


### 3. Payment

This section describes the `payment` object returned by `lsps1.create_order` and `lsps1.get_order`. The client MUST pay the `bolt11_invoice` OR the `onchain_address`. Using both methods MAY lead to the loss of funds.

> **Rationale** On-chain payments are required for payments with higher amounts, especially to push `client_balance_sat` to the client. On-chain payments are also useful to onboard new users to Lightining. On the other hand, Lightning payments are the preferred way to do payments because they are quick and easily refundable.


**Payment object**

```json
"payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_sat": "8888",
    "order_total_sat": "2008888",
    "bolt11_invoice": "lnbc252u1p3aht9ysp580g4633gd2x9lc5al0wd8wx0mpn97...",
    "onchain_address": "bc1p5uvtaxzkjwvey2tfy49k5vtqfpjmrgm09cvs88ezyy8h2zv7jhas9tu4yr",
    "onchain_block_confirmations_required": 1,
    "minimum_fee_for_0conf": 253,
    "onchain_payment": {
        "outpoint": "0301e0480b374b32851a9462db29dc19fe830a7f7d7a88b81612b9d42099c0ae:1",
        "sat": "1200",
        "confirmed": false
    }
},
```

- `state <string enum>` MUST be one of these values:
    - `EXPECT_PAYMENT` Payment expected.
    - `HOLD` Lighting payment arrived, preimage NOT released.
    - `PAID` Lightning payment arrived, preimage released OR full `order_total_sat` on-chain payment arrived.
    - `REFUNDED` Lightning payment or on-chain payment has been refunded.
- `fee_total_sat` <[LSPS0.sat][]> The total fee the LSP will charge to open this channel in satoshi.
- `order_total_sat` <[LSPS0.sat][]> What the client needs to pay in total to open the requested channel.
  - MUST be the `fee_total_sat` plus the `client_balance_sat` requested in satoshi.
- `bolt11_invoice <string>`
    - MUST be a [Lightning BOLT 11 invoice](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) for the number of `order_total_sat`. 
    - Invoice MUST be a [HOLD invoice](https://bitcoinops.org/en/topics/hold-invoices/).
    - MUST be at most 2048 characters long.
- `onchain_address` <[LSPS0.onchain_address][]> On-chain address the client can pay the `order_total_sat` to
    - MUST be `null` IF one of the following is true:
      - `options.min_onchain_payment_size_sat` is greater than `order_total_sat`.
      - `options.min_onchain_payment_size_sat` is `null` and on-chain payments are therefore not supported.
      - `refund_onchain_address` is `null`.
- `required_onchain_block_confirmations <uint8>` Minimum number of block confirmations that are required for the on-chain payment to be considered confirmed.
    - MUST be equal or greater than `options.minimum_onchain_payment_confirmations`.
    - MUST be `null` if `onchain_address` is `null`.
- `minimum_fee_for_0conf <LSPS0.onchain_fee>` Fee rate for on-chain payment in case the client wants the payment to be confirmed without a confirmation.
    - MUST be `null` if `onchain_address` is `null`.
    - MUST be `null` if `required_onchain_block_confirmations` is greater than 0.
    - SHOULD choose a high enough fee to lower the risk of a double spend.
- `onchain_payment <object>` Detected on-chain payment.
    - MUST contain the incoming/confirmed outpoint to `onchain_address`.
    - MAY be null if no payment has been detected.
    - `outpoint` <LSPS0.outpoint> MUST be an outpoint in the form of [txid:vout](https://btcinformation.org/en/glossary/outpoint).
    - `sat` <[LSPS0.sat][]> MUST contain the received satoshi.
    - `confirmed <boolean>` Indicates if the LSP sees the transaction as confirmed.

> **Rationale `required_onchain_block_confirmations`** The main risk for an LSP is that the client pays the on-chain payment and then double spends the transaction. This is especially critical in case the client requested a high `client_balance`. Opening a 0conf channel alone has no risk attached to it IF the on-chain payment is confirmed. Therefore, the LSP can mitigate this risk by waiting for a certain number of block confirmations before opening the channel.

> **Rationale `minimum_fee_for_0conf`** The client MAY want to have instant confirmation of the on-chain payment. The LSP can mitigate the risk of a double spend by requiring a high fee rate. The client can then decide if he wants to pay the high fee rate or wait for the on-chain payment to be confirmed once.

#### 3.1 Lightning Payment

**Client**

- MUST pay the `bolt11_invoice`.
- SHOULD pull `lsps1.get_order` to check the success of the payment.
- The client gets refunded automatically in case the channel open failed, the order expires, or just before the payment times out.

**LSP**

- MUST change the `payment.state` to `HOLD` when the payment arrived.
- If the channel has been opened successfully
    - MUST release the preimage and therefore complete the payment.
    - MUST set the `payment.state` to `PAID`.
- If the channel failed to open or the order expired or shortly before the payment times out:
    - MUST reject the payment.
    - MUST set the `payment.state` to `REFUNDED`.



#### 3.2 Onchain Payment Flow

**Client**

- MUST pay `order_total_sat` to `onchain_address`.
- MAY pull `lsps1.get_order` to check the success of the payment.

**LSP** Payment confirmation

- MUST monitor the blockchain and update `onchain_payment`.
- IF `onchain_block_confirmations_required` is 0 and incoming transaction fee is greater than `minimum_fee_for_0conf`:
  - SHOULD set the transaction as confirmed.
- IF `onchain_block_confirmations_required` is equal or greater than 1:
  - SHOULD set the transaction as confirmed after `onchain_block_confirmations_required` confirmations.
- MAY always set the transaction as confirmed before `onchain_block_confirmations_required` confirmations.
- In rare circumstances, MAY take longer than `onchain_block_confirmations_required` confirmations in case the LSP doesn't trust the transaction.
- MUST always set the transaction as confirmed after 6 block confirmations.


**LSP** State change

- MUST change the `payment.state` to `PAID` when the payment for the order is confirmed.
- If the order expired and the channel has NOT been opened, OR the channel open failed.
    - MUST refund the client to `refund_onchain_address`.
      - The number of satoshi to refund 
        - MUST be `order_total_sat` MINUS transaction size * onchain fee rate. The LSP MUST choose a reasonable fee rate.
        - MAY overprovision.
      - The LSP MUST bump the fee in case the transaction doesn't resolve within 6 hours.
    - MUST set the payment `payment.state` to `REFUNDED`.


**0conf risk management**

Every LSP that accepts 0conf transactions is responsible to do their own risk management. `minimum_onchain_payment_confirmations` and `minimum_fee_for_0conf` are the risk management tools the LSP has. The main risk lies in the fact that the client can double spend the on-chain payment. Opening a 0conf channel is not risky for the LSP anymore if the LSP is sure of the on-chain payment.

`minimum_onchain_payment_confirmations` and `minimum_fee_for_0conf` are the best estimates of the LSP makes at the time of the order creation. These estimates MAY change over time so the LSP MAY confirm a transaction earlier or later. Therefore, the client MUST NOT rely on the LSP confirming the transaction within the given time frame.


### 4. Channel

**Channel object**

```json
"channel": {
  "state": "OPENED",
  "funded_at": "2012-04-23T18:25:43.511Z",
  "funding_outpoint": "0301e0480b374b32851a9462db29dc19fe830a7f7d7a88b81612b9d42099c0ae:0",
  "scid": "643904x1419x1",
  "expires_at": "2012-04-23T18:25:43.511Z",
  "closing_transaction": "0301e0480b374b32851a9462db29dc19fe830a7f7d7a88b81612b9d42099c0ae",
  "closed_at": "2012-04-23T18:25:43.511Z"
}
```


- `channel <object>` Contains channel information. 
  - MUST be `null` if the channel opening transaction has not been published yet.
  - `state <string enum>` MUST be one of these values:
      - `OPENING` Opening transaction published.
      - `OPENED` Channel is open. LSP must allow payments. May be zero conf and therefore immediate.
      - `CLOSED` Closing transaction has been published.
  - `funded_at` <[LSPS0.datetime][]> Datetime when the funding transaction has been published.
  - `funding_outpoint` <[LSPS0.outpoint][]> Outpoint of the [funding transaction](https://github.com/lightning/bolts/blob/master/03-transactions.md#funding-transaction-output).
  - `scid` <[LSPS0.scid][] or null> Short channel id. 
    - MUST be `null` before the channel is confirmed on-chain.
  - `expires_at` <[LSPS0.datetime][]> Ealierst datetime when the channel MAY be closed by the LSP. 
    - MUST respect `channel_expiry_blocks`. 
    - MAY overprovision.
  - `closing_transaction` <[LSPS0.txid][] or null> txId of the [closing transaction](https://github.com/lightning/bolts/blob/master/03-transactions.md#closing-transaction).
  - `closed_at` <[LSPS0.datetime][] or null> Datetime when the closing transaction has been published.
    - MUST be `null` before the channel is closed.


#### 4.1 Channel Open

The LSP MUST open the channel under the following conditions:
- `payment.state` is `HOLD` (lightning) or `PAID` (on-chain).

**LSP**
- MUST wait for a peer connection before attempting a channel open.
- MUST attempt a channel open.
    - MUST respect the `announceChannel` flag.
    - MUST open the channel with at least a capacity of `lsp_balance_sat` + `client_balance_sat`.
        - MAY overprovision.
    - MUST push `client_balance_sat` to the client.
        - MAY overprovision.
    - MUST use a high enough on-chain fee rate to ensure the funding transaction confirms within `confirms_within_blocks` after the client paid the order.
        - MAY overprovision.
- MUST send `channel_ready` after the funding transaction has `minimum_channel_confirmations`.
- MUST allow zero channel reserves if `supports_zero_channel_reserve`.

In case the channel open succeeds
- MUST set `order_state` to `COMPLETED`.
- MUST update the channel object.

In case the channel open failed
- MUST set `order_state` to `FAILED`.
- MUST issue a refund.


##### Note about Channel Batching

**LSP**

- MAY open channels in batches, opening multiple channels in one transaction.
  - In this case, the LSP MUST still ensure that the funding transaction gets confirmed after a maximum of `confirms_within_blocks` after the client payment completed.

> **Rationale `confirms_within_blocks`** We use `confirms_within_blocks` instead of `fee_rate` to allow the LSP to batch the channel. For example, if a client orders a channel within 5 blocks, the LSP may wait to publish the funding transaction for 3 blocks to batch channels and add a fee to the funding transaction to ensure it confirms within 2 blocks.




[LSPS0.common_schemas]: ../LSPS0/common-schemas.md
[LSPS0.sat]: ../LSPS0/common-schemas.md#link-lsps0sat
[LSPS0.onchain_address]: ../LSPS0/common-schemas.md#link-lsps0onchain_address
[LSPS0.datetime]: ../LSPS0/common-schemas.md#link-lsps0datetime
[LSPS0.outpoint]: ../LSPS0/common-schemas.md#link-lsps0outpoint
[LSPS0.scid]: ../LSPS0/common-schemas.md#link-lsps0scid
[LSPS0.txid]: ../LSPS0/common-schemas.md#link-lsps0txid

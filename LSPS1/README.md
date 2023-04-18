# LSPS1 channel request

| Name    	| base_api                      |
|---------	|------------------------------	|
| Version 	| 2                             |
| OpenApi 	| [LSPS1.yaml](./LSPS1.yaml) 	|



## Motivation

The goal of this specification is to provide a standardized LSP API for wallets to purchase a channel from the LSP directly.

## OpenAPI

All specs are defined in the [OpenAPI](https://www.openapis.org/about) format. It can be viewed with various editors, most notably the [Swagger Editor](https://editor.swagger.io/) or the [vscode-openapi](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi) extension. All the OpenAPI documents include detailed descriptions of every field.

## Definitions

### Datetime

All datetimes are represented as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) in the form of `2023-02-23T08:47:30.511Z` UTC.

> **Rationale** ISO8601 is human readable, it's a standard endorsed by [W3C](http://www.w3.org/TR/NOTE-datetime), and [RFC3339](https://www.rfc-editor.org/rfc/rfc3339), sortable, and supported widely.

### Satoshi

All satoshi values MUST be represented as a string and NOT integer values. Make sure you check your integer limits to avoid overflows.

> **Rationale** Plenty of json parsers use a 32bit signed integer for integer parsing. Max safe value is 2,147,483,647; 2,147,483,647sat = BTC21.474,836,47 which is too low.

### Node connection string

Node connection strings like `lsp_connection_string` MUST match one of the following grammars:

* `<node> '@' <ip> `:` <port>`, with the last `:` considered a separator for the port number.
* `<node> '@' '[' <ip> ']' ':' <port>`, with any `:` inside `<ip>` considered part of the IP address.
* `<node> '@' <tor_onion_v3_address> ':' <port>`, with the last `:` after `@` considered a separator for the port number.

Word definition:

* `<node>` MUST be a node id (pubkey). For example: "0200000000a3eff613189ca6c4070c89206ad658e286751eca1f29262948247a5f". See [BOLT08](https://github.com/lightning/bolts/blob/master/08-transport.md?plain=1#L5).
* `<ip>` MUST be a IPv4 OR IPv6 address. See [InetAddressIPv4 in RFC4001](https://datatracker.ietf.org/doc/html/rfc4001) and [RFC 5952 for IPv6](https://datatracker.ietf.org/doc/html/rfc5952).
* `<tor_onion_v3_address>` MUST be a tor onion v3 address. See [prop224](https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt).
* `<port>` MUST be a [port](https://en.wikipedia.org/wiki/Port_(computer_networking)) number.

### Onchain addresses

- All onchain addresses MUST be a [bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki) version 0 ("SegWit") or [bech32m](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki) version 1 ("Taproot") address or null.
- The lsp/user MAY support other address types.

> **Rationale** Onchain addresses MUST be null in case onchain payments are unsupported. This MUST be clearly indicated though. See options dictionary.

### Actors

`LSP` is the API provider. `User` is the user/client of the API.

### Channel sides

`user_balance` are the funds on the user side. `lsp_balance` are the funds on the LSP side.

### Overprovisioning

The LSP is allowed to overprovision channels/onchain-payments/onchain-fees as long as it benefits the user.

> **Rationale** The lsp may need to "bin" UTXOs.

## Flow

### API information

`GET /lsp/channels` is the entrypoint for each client using the api. It lists the versions of the api and all options in a dictionary.

The user MUST pull `GET /lsp/channels` to read

- the `versions` of the api and therefore prove compatibility.
- `options` properties to determine the api boundries.

Example `GET /lsp/channels` response: 

```JSON
{
  "versions": [2],
  "website": "http://example.com/contact",
  "options": {
      "minimum_depth": 0,
      "supports_zero_channel_reserve": true,
      "max_user_balance_satoshi": "0",
      "max_lsp_balance_satoshi": "100000000",
      "min_required_onchain_satoshi": null,
      "max_channel_expiry_blocks": 20160
  }
}
```

#### Api options

The api itself has multiple properties that MUST be defined.

```json
"options": {
        "minimum_depth": 0,
        "supports_zero_channel_reserve": true,
        "max_user_balance_satoshi": "0",
        "max_lsp_balance_satoshi": "100000000",
        "min_required_onchain_satoshi": null,
        "max_channel_expiry_blocks": 20160
}
```

- `minimum_depth` MUST set to the number of blocks it requires for the LSP to send `channel_ready` (previously `funding_locked`).
  - MAY be 0 to allow 0conf channels.
- `supports_zero_channel_reserve` SHOULD set to true if the lsp supports [zeroreserve](https://github.com/ElementsProject/lightning/pull/5315).
- `max_user_balance_satoshi` MUST be the maximum number of satoshi that the LSP is willing to push to the user. MUST be 0 or a positive integer.
- `max_lsp_balance_satoshi` MUST be the maximum number of satoshi that the LSP is willing to contribute to the their balance.  MUST be 1 or greater.
- `min_required_onchain_satoshi` MUST be the number of satoshi (`order_total_satoshi` see below) that are required for the user to pay funds onchain. The LSP MUST allow onchain payments equal or above this value. MAY be null if onchain payments are NOT supported.
- `max_channel_expiry_blocks` MUST be the maximum length in blocks a channel can be leased for. MUST be 1 or greater.

The user MAY abort the flow here.

### 1. Create Order

The user constructs the request body depending on their needs. 

- The user MUST check if [option_support_large_channel](https://bitcoinops.org/en/topics/large-channels/) is enabled before they order a channel larger than BTC0.16777216.
- The user MUST call `POST /lsp/channels` to create an order.

**Example request body**

```json
{
  "api_version": 2,
  "order": {
    "lsp_balance_satoshi": "5000000",
    "user_balance_satoshi": "2000000",
    "confirms_within_blocks": 1,
    "channel_expiry_blocks": 144,
    "coupon_code": "",
    "refund_onchain_address": "bc1qvmsy0f3yyes6z9jvddk8xqwznndmdwapvrc0xrmhd3vqj5rhdrrq6hz49h"
  },
  "open": {
    "announce": true,
    "user_connection_string_or_pubkey": "03d4e028a0d4a90868ec202ab684fb0085779defea9ca7553e06146557631eec20@3.33.236.230:9735"
  }
}
```

- `api_version` MUST be `2`. MUST match one of the versions listed by the API.
- `order` object MUST be provided.
    - `lsp_balance_satoshi` MUST be 1 or greater. MUST be below or equal `base_api.max_lsp_balance_satoshi`.
    - `user_balance_satoshi` MUST be 0 or greater. MUST be below or equal `base_api.max_user_balance_satoshi`. Todo: Rejection error message.
    - `confirms_within_blocks` MUST be 0 or higher.
    - `channel_expiry_blocks` MUST be 1 or greater. MUST be below or equal `base_api.max_channel_expiry_blocks`.
    - `coupon_code` MUST be a string or null.
    - `refund_onchain_address` 
      - MUST be an onchain address or null.
      - If null the LSP MUST disable onchain payments in the order.
- `open` MUST be provided.
    - `announce` If the channel should be announced to the network. MUST be boolean.
    - `user_connection_string_or_pubkey` MUST be a node connection string OR a node id (pubkey).


> **Rationale user_balance_satoshi** User MAY want to have initial spending balance on their wallet or start with a balanced channel.

> **Rationale coupon_code** User MAY provide a coupon_code to claim a discount on the order. 


**Example response body**

HTTP Code: 201 CREATED

```json
{
  "id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c",
  "state": "AWAITING_PAYMENT",
  "lsp_balance_satoshi": "5000000",
  "user_balance_satoshi": "2000000",
  "confirms_within_blocks": 1,
  "channel_expiry_blocks": 12,
  "coupon_code": "",
  "lsp_connection_strings": [
    "03864ef025fde8fb587d989186ce6a4a186895ee44a926bfc370e2c366597a3f8f@3.33.236.230:9735",
    "03864ef025fde8fb587d989186ce6a4a186895ee44a926bfc370e2c366597a3f8f@gwdllz5g7vky2q4gr45zguvoajzf33czreca3a3exosftx72ekppkuqd.onion:9735"
  ],
  "created_at": "2012-04-23T18:25:43.511Z",
  "expires_at": "2015-01-25T19:29:44.612Z",
  "open": {
    "announce": true,
    "client_connection_string_or_pubkey": "03d4e028a0d4a90868ec202ab684fb0085779defea9ca7553e06146557631eec20@3.33.236.230:9735",
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

**LSP**
- SHOULD order the `lsp_connection_strings` on the desirability of a user connecting to it with the top element being the most desirable.

**User**
- SHOULD validate the `fee_total_satoshi` is reasonable.
- SHOULD validate `fee_total_satoshi` + `user_balance_satoshi` = `order_total_satoshi`.
- MAY abort the flow here.

**Errors**

- 400 Bad request - Request body validation error.


### 2. Payment

This section describes the payment object returned by `POST /lsp/channel` and `GET /lsp/channel/{id}`. The user MUST pay the `lightning_invoice` OR the `onchain_address`. Using both methods MAY lead to the loss of funds.

> **Rationale** Onchain Payments are required for payments with higher amounts, especially to push user_balance_satoshi to the user. Onchain payments are also useful to onboard new user to Lightining. Lightning payments are the preferred way to do payments because they are quick and easily refundable.

**Before the payment**
- The user SHOULD already open a peer connection to the LSP. This will allow the LSP to open then channel instantly on payment arrival. See 4.1 Establish Peer Connection.

Example payment object:
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
- `order_total_satoshi` MUST be the fee_total_satoshi plus the user_balance_satoshi requested in satoshi.
- `ln_invoice` 
    - MUST be a Lightning Bolt 11 for order_total_satoshi. 
    - Invoice MUST be a [HOLD invoice](https://bitcoinops.org/en/topics/hold-invoices/).
    - The `ln_invoice` MUST give the same amount as in `order_total_satoshi`.
- `onchain_address` 
    - MUST be an onchain address the user can pay the order_total_satoshi to.
    - MUST be null 
      - if `base_api.min_required_onchain_satoshi` is above or equal order_total.
      - if `base_api.min_required_onchain_satoshi` is null and therefore not supported.
      - if `refund_onchain_address` is null.
- `onchain_address_confirms` 
    - MUST be the number of confirmations that the server will accept before considering the payment as confirmed. If 0, this indicates that the server accepts 0-confirmation payments to this address.
    - MUST be positive or 0.
- `onchain_payments`
    - MUST contain all incoming/confirmed outpoints to onchain_address. 
    - `outpoint` MUST be an outpoint in the form of [txid:vout](https://btcinformation.org/en/glossary/outpoint).
    - `satoshi` MUST contain the received satoshi.
    - `confirmed` MUST contain a boolean if the LSP sees the transaction as confirmed.


#### 2.1 Lightning Payment Flow

**User**

- MUST pay the `lightning_invoice`.
- SHOULD pull `GET /lsp/channel/{id}` to check the success of the payment.
- The user gets refunded automatically in case the channel open failed, the order expired, or the payment timed out.

**LSP**

- MUST change the payment state to `HOLD` when the payment arrived.
- MUST set open.state to `PENDING`.
- If the channel has been opened successfully
    - MUST release the preimage and therefore complete the payment.
    - MUST set the payment state to `PAID`.
- If the payment times out and the channel failed to open or the order expired:
    - MUST reject the payment.
    - MUST set the payment state to `REFUNDED`.



#### 2.2 Onchain Payment Flow

**User**

- MUST pay `order_total_satoshi` to `onchain_address`.
- MAY pull `GET /lsp/channel/{id}` to check the success of the payment.

**LSP**

- MUST monitor the blockchain and update `onchain_payments`.
- MUST set the transaction as confirmed after `onchain_address_confirms` confirmations.
- MUST change the payment state to `PAID` when all the payments for the order are confirmed.
- MUST set open.state to `PENDING`.
- If the order expired and the channel has NOT been opened, OR the channel open failed.
    - MUST refund the user to `refund_onchain_address`.
      - The number of satoshi to refund 
        - MUST be `order_total_satoshi` MINUS transaction size * onchain_fee_rate. The LSP MUST choose a reasonable fee rate.
        - MAY overprovision.
      - The LSP MUST bump the fees in case the transaction doesn't resolve within 6hrs.
    - MUST set the payment state to `REFUNDED`.



### 3 Channel Open

The LSP MUST open the channel under the following conditions:
- The open.state switched to `PENDING`


#### 3.1 Establish Peer Connection

**User**

- MUST open a peer connection with one of the provided node connection strings in `lsp_connection_strings` before the channel is being opened.
- SHOULD choose the best connection string depending on their requirements.

**LSP**
- MAY establish a peer connection to `user_connection_string_or_pubkey`.


#### 3.2 Open attempt

**LSP**
- MUST wait for a peer connection before attempting a channel open.
- MUST attempt a channel open to `user_connection_string_or_pubkey`.
    - MUST respect the `announce` flag.
    - MUST open the channel with a capacity of `lsp_balance_satoshi` + `user_balance_satoshi`.
        - MAY overprovision.
    - MUST push `user_balance_satoshi` to the user.
        - MAY overprovision.
    - MUST use a high enough onchain fee rate to ensure the funding transaction confirms within `confirms_within_blocks` after the user paid the order and established a peer connection.
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
  - In this case, the LSP MUST still ensure that the funding transaction gets confirmed after a maximum of `confirms_within_blocks` after the user payment completed.


### 5 Channel Object

Todo: Describe channel object. Might be simplified or simply unnecessary.


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
    - `user_pubkey` MUST be the node id of the user node.
    - `lsp_pubkey` MUST be the node id if the lsp node.

    
# Open Questions

- Do we allow 0conf for high lsp_balance_satoshi? Do we need something like `max_lsp_balance_satoshi` depending on the `minimum_depth`?
- Error handling
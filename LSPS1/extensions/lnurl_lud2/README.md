# LSPS1.lnurl_lud2

| Name    	| lnurl_lud2                                     	|
|---------	|------------------------------------------------	|
| Version 	| 1                                              	|
| OpenApi 	| [LSPS1.lnurl_lud2.yaml](LSPS1.lnurl_lud2.yaml) 	|



## Base API

The base api consists of 4 endpoints:
- `GET /lsp/channels`: General API information.
- `POST /lsp/channels`: Create channel request.
- `GET /lsp/channels/{id}`: Get request state.
- `POST /lsp/channels/{id}/open`: Synchronously open channel.


### Basic client flow

1. Client pulls `GET /lsp/channels` to determine what the api supports.
2. Client calls `POST /lsp/channels` with the values demanded to create an order. The response includes prices, a lightning invoice and a bitcoin address in case the feature `onchain_payment` is supported.
3. Client pays the order with their method of choice before the order is expired.
4. Client pulls `GET /lsp/channels/{id}` to determine the status of their payment. As soon as `payment.state` switches to `PAID` they can attempt a channel open.
5. Client calls `POST /lsp/channels/{id}/open` with the nessecary parameters for a channel open. The LSP directly attempts a channel open and responds with the result. The client gets direct feedback if the attempt succeeded.
6. The `channel` object in `GET /lsp/channels/{id}` represents the state of the channel. `expiry_ts` shows when the LSP is allowed to close the channel again.

## Features

The base api can be extended by additional features. Feature can extend the base api and add additional endpoints. Features need to stay backward-compatible with the base api.

Each supported feature must to be listed in `GET /lsp/channels`.`features` either as a `string` or as an object.

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

The feature `lnurl_lud2` adds a new endpoint `GET /lsp/channels/{id}/lnurl` that provides an lnurl channel request according to [LUD2](https://github.com/lnurl/luds/blob/luds/02.md).

### local_balance

The feature `local_balance` allows `POST /lsp/channels`.`local_balance` to be greater 0. Otherwise it is statically set to 0.

### refunds

The feature `refunds` adds two endpoints to automatically manage refunds.

- `POST /lsp/channels/{id}/refund` allows a user to claim the refund with a Lightning invoice or an onchain address.
- `GET /lsp/channels/{id}/refund` shows the refund state and available amount.

#### Client flow

1. Pull `GET /lsp/channels/{id}/refund` to receive the information if a refund is available and what amounts. Amounts can differ depending on the payout method.
2. Call `POST /lsp/channels/{id}/refund` to instruct the LSP to issue a refund. The endpoint either takes a lightning invoice or an onchain_address.
3. Client can watch the progress of the payout by pulling `GET /lsp/channels/{id}/refund`.


### jit_channels

The feature `jit_channels` adds the ability to to register a node and receive a route hint to add the channel Just-In-Time.

Todo: More information needed from Breez.

#### Client flow

1. Client creates an order by calling `POST /lsp/channels`.
2. Client registers their node by calling ``POST /lsp/channels/{id}/jit`.
3. Client uses the route hint provided to issue invoices.
4. The LSP opens a channel just in time as soon as their is an incoming channel.
5. The order is paid by high fees on incoming HTLCs.

### onchain_payments

The feature `onchain_payments` indicates if this LSP is willing to receive payments onchain. The feature has 1 required setting:

- `min_satoshi` indicates at what value the LSP is willing to receive payments onchain.

In case the LSP doesn't support onchain payments, all values related to onchain payments will be `null`.



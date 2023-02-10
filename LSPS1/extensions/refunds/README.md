# LSPS1.refunds

| Name    	| refunds                                     	|
|---------	|------------------------------------------------	|
| Version 	| 1                                              	|
| OpenApi 	| [LSPS1.refunds.yaml](./LSPS1.refunds.yaml) 	|



## API

The extension `refunds` adds two endpoints to automatically manage refunds.

### POST /lsp/channels/{id}/refund

Allows a user to claim the refund with a Lightning invoice or an onchain address (if `onchain_payments` is supported).

### GET /lsp/channels/{id}/refund

Shows the refund state and available amount.

## Client flow

1. Client pull `GET /lsp/channels/{id}/refund` to check if a refund is available and how much is available.
2. Client calls `POST /lsp/channels/{id}/refund` with a lightning invoice or an onchain address to claim the refund.
3. Client monitors the state of the payout by pulling `GET /lsp/channels/{id}/refund`.

## Extension

```json
{
    "name": "refunds",
    "version": 1
}
```


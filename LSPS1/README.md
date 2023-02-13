# LSPS1 channel request

| Name    	| base_api                                     	|
|---------	|------------------------------------------------	|
| Version 	| 2                                              	|
| OpenApi 	| [LSPS1.yaml](./LSPS1.yaml) 	|

## Introduction

The API is split between the base api and possible extensions. The base api is required to support this spec. Extensions are defined in the [extensions](./extensions/) folder. It is up to every LSP what extensions they like to implement.

All specs are defined in the [OpenAPI](https://www.openapis.org/about) format. It can be viewed with various editors, most notable the [Swagger Editor](https://editor.swagger.io/) or the [vscode-openapi](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi) extension. All the OpenAPI documents include detailed descriptions for every field.

| Extension                              	    | Version 	|
|----------------------------------------------	|---------	|
| [lnurl_lud2](./extensions/lnurl_lud2/)        | 1       	|
| [refunds](./extensions/refunds/)          	| 1       	|
| [jit_channels](./extensions/jit_channels/) 	| 1       	|

## General API information

`GET /lsp/channels` is the entrypoint for each client using the api. It lists the version of the api and all supported extensions in a dictionary.

An extension in `GET /lsp/channels`.`extensions` consists of the `BaseExtension` properties and can have additional properties based on the individual extensions requirement.

```yaml
BaseExtension:
    type: object
    properties:
        version:
            type: integer
            example: 1
            default: 1
            nullable: true
            description: Version of this extension. Integer starting at 1 counting up.
```

For example, the extension `OnchainPaymentExtension` adds an additional property called `min_satoshi`.

```yaml
OnchainPaymentExtension:
    allOf:
        - $ref: "#/components/schemas/BaseExtension" # Extends BaseExtension
        - type: object
          properties:
              min_satoshi:
              type: string
              description: "Minimal number of satoshi required to be eligable for an onchain payment."
              example: 50000
```


## Base API

The base api is defined in [LSPS1.yaml](./LSPS1.yaml) and consists of 4 endpoints:

- `GET /lsp/channels`: General API information.
- `POST /lsp/channels`: Create channel request.
- `GET /lsp/channels/{id}`: Get request state.
- `POST /lsp/channels/{id}/open`: Synchronously open channel.

### Extensions

The base api itself allows for 2 extensions.

#### local_balance

Indicates if the LSP is willing to push satoshi to the receiver side.

```json
"local_balance": {
    "version": 1
}
```

#### onchain_payments

Indicates if the LSP is willing to receive payments onchain.

Properties:

- `min_satoshi` is the number of satoshi that are required for an onchain payment.

```json
"onchain_payments": {
    "version": 1,
    "min_satoshi": 50000
}
```

### Client flow

1. Client pulls `GET /lsp/channels` to determine what the api supports.
2. Client calls `POST /lsp/channels` with the values demanded to create an order. The response includes prices, a lightning invoice and a bitcoin address in case the extension `onchain_payment` is supported.
3. Client pays the order with their method of choice before the order expires.
4. Client pulls `GET /lsp/channels/{id}` to determine the status of their payment. As soon as `payment.state` switches to `PAID` they can attempt a channel open.
5. Client calls `POST /lsp/channels/{id}/open` with the nessecary parameters for a channel open. The LSP directly attempts a channel open and responds with the result. The client gets direct feedback if the attempt succeeded.
6. The `channel` object in `GET /lsp/channels/{id}` represents the state of the channel. `expiry_ts` shows when the LSP is allowed to close the channel again.


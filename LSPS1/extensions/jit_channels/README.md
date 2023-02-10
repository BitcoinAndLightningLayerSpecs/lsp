# LSPS1.jit_channels

| Name    	| jit_channels                                     	|
|---------	|------------------------------------------------	|
| Version 	| 1                                              	|
| OpenApi 	| [LSPS1.jit_channels.yaml](./LSPS1.jit_channels.yaml) 	|

> **Note** This extension requires some more love.

## API

The extension `jit_channels` adds the ability to to register a node and receive a route hint to add the channel Just-In-Time.

### POST /lsp/channels/{id}/jit

Registers a node. The response contains a route hint that should be added to the invoice.

### GET /lsp/channels/{id}/jit

Gets information about the registered node.

## Extension

```json
{
    "name": "jit_channels",
    "version": 1
}
```
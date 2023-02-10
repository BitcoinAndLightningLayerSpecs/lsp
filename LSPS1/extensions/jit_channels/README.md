# LSPS1.jit_channels

> **Note** This extension requires some more love.

| Name    	| jit_channels                                     	|
|---------	|------------------------------------------------	|
| Version 	| 1                                              	|
| OpenApi 	| [LSPS1.jit_channels.yaml](./LSPS1.jit_channels.yaml) 	|



## API

The extension `jit_channels` adds the ability to to register a node and receive a route hint to add the channel Just-In-Time.

### POST /lsp/channels/{id}/jit

Registers a node. The response contains a route hint that should be added to the invoice.

### GET /lsp/channels/{id}/jit

Contains information about the registered node and route hint.

## Extension

```json
{
    "name": "jit_channels",
    "version": 1
}
```
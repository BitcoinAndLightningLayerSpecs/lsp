# LSPS1.lnurl_lud2

| Name    	| lnurl_lud2                                     	|
|---------	|------------------------------------------------	|
| Version 	| 1                                              	|
| OpenApi 	| [LSPS1.lnurl_lud2.yaml](./LSPS1.lnurl_lud2.yaml) 	|



## API

The extension `lnurl_lud2` adds a new endpoint `GET /lsp/channels/{id}/lnurl` that provides a lnurl channel request according to [LUD2](https://github.com/lnurl/luds/blob/luds/02.md). The endpoint is available after ``GET /lsp/channels/{id}`.`payment.state` is `PAID`.


## Extension

```json
{
    "name": "lnurl_lud2",
    "version": 1
}
```


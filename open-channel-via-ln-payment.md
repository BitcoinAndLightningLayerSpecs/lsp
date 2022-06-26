# Protocol Documentation
<a name="top"></a>

## Table of Contents

- [lspd.proto](#lspd.proto)
    - [ChannelInformationReply](#lspd.ChannelInformationReply)
    - [ChannelInformationRequest](#lspd.ChannelInformationRequest)
    - [OpenChannelReply](#lspd.OpenChannelReply)
    - [OpenChannelRequest](#lspd.OpenChannelRequest)
    - [PaymentInformation](#lspd.PaymentInformation)
    - [RegisterPaymentReply](#lspd.RegisterPaymentReply)
    - [RegisterPaymentRequest](#lspd.RegisterPaymentRequest)
  
  
  
    - [ChannelOpener](#lspd.ChannelOpener)
  

- [Scalar Value Types](#scalar-value-types)



<a name="lspd.proto"></a>
<p align="right"><a href="#top">Top</a></p>

## lspd.proto



<a name="lspd.ChannelInformationReply"></a>

### ChannelInformationReply



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| name | [string](#string) |  | The name of of lsp |
| pubkey | [string](#string) |  | The identity pubkey of the Lightning node |
| host | [string](#string) |  | The network location of the lightning node, e.g. `12.34.56.78:9012` or / `localhost:10011` |
| channel_capacity | [int64](#int64) |  | The channel capacity in satoshis |
| target_conf | [int32](#int32) |  | The target number of blocks that the funding transaction should be / confirmed by. |
| base_fee_msat | [int64](#int64) |  | The base fee charged regardless of the number of milli-satoshis sent. |
| fee_rate | [double](#double) |  | The effective fee rate in milli-satoshis. The precision of this value goes / up to 6 decimal places, so 1e-6. |
| time_lock_delta | [uint32](#uint32) |  | The required timelock delta for HTLCs forwarded over the channel. |
| min_htlc_msat | [int64](#int64) |  | The minimum value in millisatoshi we will require for incoming HTLCs on / the channel. |
| channel_fee_permyriad | [int64](#int64) |  |  |
| lsp_pubkey | [bytes](#bytes) |  |  |






<a name="lspd.ChannelInformationRequest"></a>

### ChannelInformationRequest



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| pubkey | [string](#string) |  | The identity pubkey of the Lightning node |






<a name="lspd.OpenChannelReply"></a>

### OpenChannelReply



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| tx_hash | [string](#string) |  | The transaction hash |
| output_index | [uint32](#uint32) |  | The output index |






<a name="lspd.OpenChannelRequest"></a>

### OpenChannelRequest



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| pubkey | [string](#string) |  | The identity pubkey of the Lightning node |






<a name="lspd.PaymentInformation"></a>

### PaymentInformation



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| payment_hash | [bytes](#bytes) |  |  |
| payment_secret | [bytes](#bytes) |  |  |
| destination | [bytes](#bytes) |  |  |
| incoming_amount_msat | [int64](#int64) |  |  |
| outgoing_amount_msat | [int64](#int64) |  |  |






<a name="lspd.RegisterPaymentReply"></a>

### RegisterPaymentReply







<a name="lspd.RegisterPaymentRequest"></a>

### RegisterPaymentRequest



| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| blob | [bytes](#bytes) |  |  |





 

 

 


<a name="lspd.ChannelOpener"></a>

### ChannelOpener


| Method Name | Request Type | Response Type | Description |
| ----------- | ------------ | ------------- | ------------|
| ChannelInformation | [ChannelInformationRequest](#lspd.ChannelInformationRequest) | [ChannelInformationReply](#lspd.ChannelInformationReply) |  |
| OpenChannel | [OpenChannelRequest](#lspd.OpenChannelRequest) | [OpenChannelReply](#lspd.OpenChannelReply) |  |
| RegisterPayment | [RegisterPaymentRequest](#lspd.RegisterPaymentRequest) | [RegisterPaymentReply](#lspd.RegisterPaymentReply) |  |

 



## Scalar Value Types

| .proto Type | Notes | C++ Type | Java Type | Python Type |
| ----------- | ----- | -------- | --------- | ----------- |
| <a name="double" /> double |  | double | double | float |
| <a name="float" /> float |  | float | float | float |
| <a name="int32" /> int32 | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int32 | int | int |
| <a name="int64" /> int64 | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int64 | long | int/long |
| <a name="uint32" /> uint32 | Uses variable-length encoding. | uint32 | int | int/long |
| <a name="uint64" /> uint64 | Uses variable-length encoding. | uint64 | long | int/long |
| <a name="sint32" /> sint32 | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | int32 | int | int |
| <a name="sint64" /> sint64 | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | int64 | long | int/long |
| <a name="fixed32" /> fixed32 | Always four bytes. More efficient than uint32 if values are often greater than 2^28. | uint32 | int | int |
| <a name="fixed64" /> fixed64 | Always eight bytes. More efficient than uint64 if values are often greater than 2^56. | uint64 | long | int/long |
| <a name="sfixed32" /> sfixed32 | Always four bytes. | int32 | int | int |
| <a name="sfixed64" /> sfixed64 | Always eight bytes. | int64 | long | int/long |
| <a name="bool" /> bool |  | bool | boolean | boolean |
| <a name="string" /> string | A string must always contain UTF-8 encoded or 7-bit ASCII text. | string | String | str/unicode |
| <a name="bytes" /> bytes | May contain any arbitrary sequence of bytes. | string | ByteString | str |


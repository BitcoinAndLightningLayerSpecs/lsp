# Liquidity marketplace spec


## Version: 0.0.1

##### Summary

Creating interoperable liquidty API we can make integrations between LSPs easier and enable consumers to utilize liquidity from wider selection of providers 

##### Description

Request an inbound channel with a specific size and duration.

##### Minimal order 


| Field  |  Description |   Unit |
|---|---|---|
|  min_size  | minimum chan size i'm willing to sell (  | sats |
|  max_size  |  total available capacity  | sats 
|  base_fee |  fixed fee charged for lease  |  sats
|  rate_fee |  example: size*0.002/duration | expressed as formula 
| min_duration   |  promise to keep open for minimum this amount of time / blocks | time
| chan_base_fee   |  promise to not charge greater than this base fee   | sats
| chan_rate_fee   |  promise to not charge greater than this ppm  | ppm
|  pub_key  |  who is selling the liquidity   | 
|  policy  |  additional optional information  | 



##### Policies
Policy field is an  json object that can include various different policies that are not mandatory and serve as an additional information provided. Spec covers a list of predefined policies for common use-cases.

##### Existing policies
1. **closing_policy** - under what conditions will you close the channels or leave it open 
2. **fee_policy** - if you have tiered system for routing fees you're charging 
3. **max_htlc_size**  - what is the max htlc size you'll be allowing on the chan, useful for flow control
4. **min_htlc_size**  - what is the min htlc size you'll be allowing on the chan, if you for example don't want to route 1sat payments over these chans
5. **reputation** - what reputation systems are you using and their links 
6. **push_support** - if you support pushing part of the balance to buyers side 
7. **duration_unit** - what units are you using for duration 
8. **contact** - contact information (tg/twitter/web)
9. **location** - node location


##### Policy structure
Proposed policy format consists of three fields 
1. name -> standardized name in snake case, should be machine readable
2. value ->  machine readable value 
3. info -> human readable field with additional information

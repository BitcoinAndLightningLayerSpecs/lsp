# Liquidity marketplace spec

**This liquidity marketplace specification is OBSOLETE**. The quality of this specification is not sufficient and there is currently no interest to improve it. Therefore, we have made it obsolete. We discourage any usage.

## Version: 0.0.1

### Summary

Creating interoperable liquidty API we can make integrations between LSPs easier and enable consumers to utilize liquidity from wider selection of providers 


### Order


| Field  |  Description |   Unit | Required | Schema
|---|---|---|---|---|
|  min_size  | minimum chan size i'm willing to sell   | sats |yes|integer| 
|  max_size  |  total available capacity  | sats |yes| integer |
|  base_fee |  fixed fee charged for lease  |  sats | yes | integer 
|  rate_fee |  example: size*0.002/duration | expressed as formula |yes|integer|
| min_duration   |  promise to keep open for minimum this amount of time / blocks | time |yes| integer | 
| chan_base_fee   |  promise to not charge greater than this base fee   |  sats|yes| integer|
| chan_rate_fee   |  promise to not charge greater than this ppm  | ppm |yes|integer| 
|  pub_key  |  who is selling the liquidity   | N/A|yes| string
|  policy  |  additional optional information  |N/A|no| string 

#### Order example (with policies)
```
{
	"min_size": "100000",
	"max_size": "999999",
	"base_fee":	"12000",
	"rate_fee": "size*0.002/min_duration",
	"min_duration": "1440",
	"chan_base_fee": "0",
	"chan_rate_fee": "1337",
	"pub_key": "0288037d3f0bdcfb240402b43b80cdc32e41528b3e2ebe05884aff507d71fca71a",
	"policy": [ 
	{
		"name": "duration_unit",
		"value": "blocks", 
		"info": "unit for min_duration on our platform is blocks" 
	},

	{
		"name": "fee_policy", 
		"value": "",
		"info": "we have different fee tiers based on chan sizes, more at https://lsp.com/rates.html" 
	},
	{
		"name": "push_support",
		"value": "true",
		"info": "we support pushing part of the balance on your side, up to 2btc in size"
	}
	]

}
```


### Policies
Policy field is an  json object that can include various different policies that are **not mandatory** and serve as an additional information provided. Spec covers a list of predefined policies for common use-cases. Adding provider specific policies is supported and commonly used policies will be added to future versions of the spec.

##### Communitiy supported policies
1. **closing_policy** - under what conditions will you close the channels or leave it open 
2. **fee_policy** - if you have tiered system for routing fees you're charging 
3. **max_htlc_size**  - what is the max htlc size you'll be allowing on the chan, useful for flow control
4. **min_htlc_size**  - what is the min htlc size you'll be allowing on the chan, if you for example don't want to route 1sat payments over these chans
5. **reputation** - what reputation systems are you using and their links 
6. **push_support** - if you support pushing part of the balance to buyers side 
7. **duration_unit** - what units are you using for `min_duration`
8. **contact** - contact information where node operator/lsp can be reached (tg/twitter/web)
9. **location** - node location information
10. **tos** - where to find terms of use, legal, etc 


##### Policy schema
| Field  |  Description |  Required | Schema
|---|---|---|---|
|  name  | snake case policy name | yes|string| 
|  value  | machine readable value | no|string/number| 
|  info  | additional information | no|string| 


##### Policy example
```
{

"policies": [ 
	{
		"name": "max_htlc_size",
		"value": "123456", 
		"info": "i will enforce this policy for channels smaller than 50million" 
	},
	{
		"name": "closing_policy", 
		"value": "123456", 
		"info": "i will enforce this policy for channels smaller than 50million" 
	},
	{
		"name": "reputation", 
		"value": "https://fulmo.org/bos-score.csv",
		"info": "https://fulmo.org/bos-score.html"
	},
	{
		"name": "rate_fee_formula", 
		"value": "", 
		"info": "size*(100/size)" 
	},		
	{
		"name": "fee_policy", 
		"value": "",
		"info": "we have different fee tiers based on chan sizes, more at https://lsp.com/rates.html" 
	},
	{
		"name": "push_support",
		"value": "true",
		"info": "we support pushing part of the balance on your side, up to 2btc in size"
	}
	]
}
```

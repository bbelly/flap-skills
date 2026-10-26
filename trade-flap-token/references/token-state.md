# Token state

Read the token with `getTokenV8Safe`. Do not use `getTokenV8` or `getTokenV9Safe`.

```bash
cast call 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "getTokenV8Safe(address)((uint8,uint256,uint256,uint256,uint8,uint256,uint256,uint256,uint256,address,bool,bytes32,uint256,uint256,address,uint256,uint8,uint8))" \
  $TOKEN \
  --rpc-url $RPC
```

`cast` prints large integers as `<integer> [<short form>]`. Use only the first integer on each field, never the bracket. `circulatingSupply` compared as a float is wrong.

Decode the tuple in this order:

| Index | Field |
|---|---|
| 0 | `status` |
| 1 | `reserve` |
| 2 | `circulatingSupply` |
| 3 | `price` |
| 4 | `tokenVersion` |
| 5 | `r` |
| 6 | `h` |
| 7 | `k` |
| 8 | `dexSupplyThresh` |
| 9 | `quoteTokenAddress` |
| 10 | `nativeToQuoteSwapEnabled` |
| 11 | `extensionID` |
| 12 | `buyTaxRate` (bps) |
| 13 | `sellTaxRate` (bps) |
| 14 | `pool` |
| 15 | `progress` |
| 16 | `lpFeeProfile` |
| 17 | `dexId` |

An unknown address reverts with `TokenNotFound` (selector `0xde6137d1`). That is not status 0. Stop. Do not quote.

## Status

| Value | Name | Action |
|---|---|---|
| 0 | Invalid | Stop |
| 1 | Tradable | Continue, if `extensionID` is zero |
| 2 | InDuel | Stop |
| 3 | Killed | Stop |
| 4 | DEX | Stop. Do not build a DEX router call |
| 5 | Staged | Stop. The token is not created yet |
| other | Unknown | Stop |

## Route

Continue only when `status` is 1 and `extensionID` is `0x` followed by 64 zero hex characters. A non-zero `extensionID` needs `swapExactInputV3`, which this skill does not build.

**Native quote.** `quoteTokenAddress` is `0x0000000000000000000000000000000000000000`. Buy with BNB in one `swapExactInput`.

**ERC-20 quote.** `quoteTokenAddress` is a token address. Read that token's `decimals()` and `symbol()` from the chain. Do not assume 18 decimals.

- The user already said pay with BNB, and `nativeToQuoteSwapEnabled` is true: one `swapExactInput` paid in BNB. Portal converts BNB to the quote token internally. Do not buy the quote token first, and do not look for a local payment-token list. This skill does not have one.
- The user already said pay with BNB, and `nativeToQuoteSwapEnabled` is false: stop. Do not split the buy.
- The user's amount is in the quote token: pay with the quote token. If its balance is below the order and `nativeToQuoteSwapEnabled` is true, ask once whether to pay a BNB amount instead. A refusal or a failed quote ends the order.

There is no second payment list to sync. The on-chain flag is the Pay-with-BNB switch.

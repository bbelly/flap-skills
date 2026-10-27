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

## Agent ID gate (per token)

```bash
cast call 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "nftGate(address)(address)" \
  $TOKEN \
  --rpc-url $RPC
```

Selector `0x9b95f8dc`. Feature-detect: if the call reverts (Portal older than the gate surface), treat `$NFT_GATE` as the zero address and say the Portal build has no `nftGate`.

- Zero address → this token has **no** Agent ID gate (path A or C).
- Non-zero → `$NFT_GATE` is that token's ERC-721 collection (path B while on the curve). The gate is **per token**, not site-wide. Do not assume Binance Agent ID or any other fixed collection.

## Status

| Value | Name | Action |
|---|---|---|
| 0 | Invalid | Stop |
| 1 | Tradable | On the curve. Continue to Route below |
| 2 | InDuel | Stop |
| 3 | Killed | Stop |
| 4 | DEX | Graduated. Go to `references/graduated.md` (path C). Do not call Portal `swapExactInput` |
| 5 | Staged | Stop. The token is not created yet |
| other | Unknown | Stop |

## Extension ID

Known values that still use Portal `swapExactInput` (not V3):

| `extensionID` | Meaning |
|---|---|
| `0x` + 64 zero hex | No extension |
| `0xc87a159bf445e83477134d6374216c321695c2f3e23ee3351c8d3df6ec740253` | `flap.extensions.v1` (array form; Agent ID gate often uses this) |
| `0x9c312f81d9a1b68ba5472ca93c5447d57f897fbf4a8c468b315a67fe8016d1ab` | `flap.extension.nft_gated.v1` (legacy single-extension form) |

Any other non-zero `extensionID`: stop. It may need `swapExactInputV3`, which this skill does not build. Do **not** refuse the two known IDs above.

## Route

### Path C — graduated

`status` is 4: read `references/graduated.md`. No NFT check. No Portal quote or swap.

### Path A — ordinary inner market

`status` is 1, `$NFT_GATE` is zero, and `extensionID` is allowed above.

Signer: Binance Agentic Wallet first (see `references/preflight.md`). Cast fallback only when BAW is not signed in.

Then pick payment:

**Native quote.** `quoteTokenAddress` is the zero address. Buy with BNB in one `swapExactInput`.

**ERC-20 quote.** Read that token's `decimals()` and `symbol()` from the chain. Do not assume 18 decimals.

- The user already said pay with BNB, and `nativeToQuoteSwapEnabled` is true: one `swapExactInput` paid in BNB. Portal converts BNB to the quote token internally. Do not buy the quote token first.
- The user already said pay with BNB, and `nativeToQuoteSwapEnabled` is false: stop. Do not split the buy.
- The user's amount is in the quote token: pay with the quote token. If its balance is below the order and `nativeToQuoteSwapEnabled` is true, ask once whether to pay a BNB amount instead. A refusal or a failed quote ends the order.

### Path B — Agent ID-gated inner market

`status` is 1, `$NFT_GATE` is non-zero, and `extensionID` is allowed above.

- Signer is **Binance Agentic Wallet only**. If BAW is not signed in, stop and ask the user to sign in. Do not offer a hardware wallet or keystore. If this conversation already chose cast for another order, that choice does not apply here: stop and say this token requires BAW.
- Read `references/agent-id-gate.md` before quoting. Payment / quote-token rules are the same as path A after the NFT check passes.
- Trade ABI is still `swapExactInput`. The gate does not change calldata shape.

After graduation, path B becomes path C: the NFT gate no longer applies.

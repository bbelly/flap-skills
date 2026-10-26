# Tax, audit, and buy quota

## Tax

`buyTaxRate` and `sellTaxRate` from `getTokenV8Safe` are basis points. 100 bps is 1%. The two rates can differ, and either one can be 0.

Show them on two lines. On a buy, still show the sell tax. Do not add the rates together. `quoteExactInput` is already net of tax and Portal fee (a real 3%-tax sell filled exactly at the quote). Do not subtract the tax again, and do not add a protocol fee back onto it to recompute `minOutputAmount`.

| bps | What to say |
|---|---|
| 0–500 | No extra warning beyond the raw bps |
| 501–1000 | Warn. 1000 bps is 10% |
| above 1000 | Say the tax is high, and repeat the raw bps |

A high tax is a warning. It is not a reason to edit the calldata, and it is not a silent skip. A sell tax above 500 bps must appear in the buy confirmation.

## Audit

Follow `references/safety.md`. "This skill has no audit feed" is a stop, not a skip. A buy is not ready to quote, preview, or confirm until the audit step has either returned a usable result or the user has replied a standalone `continue` / `继续` after the no-audit message defined there.

## Buy quota

Read this for a buy that is still on the inner market. Skip it for a sell.

```bash
cast call 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "maxBuyPerOrigin(address)(uint16,uint256)" \
  $TOKEN \
  --rpc-url $RPC

cast call 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "buyQuotaOf(address,address)(uint256,uint256)" \
  $TOKEN $TRADER \
  --rpc-url $RPC
```

`buyQuotaOf` returns `(bought, remaining)` in token base units. `$TRADER` is the address that will send the transaction. Quota is accumulated by `tx.origin`. A sell does not restore it. A Binance Agentic Wallet `contract-call` is sent from the BAW address itself (observed in real fills), so `buyQuotaOf(token, <BAW address>)` is this user's quota.

**No cap wins.** There is no cap when any of these is true: `bps` is 0, `maxBuyAmount` is 0, or `remaining` is `2^256 - 1`. A `remaining` of 0 does not override that. Do not print `2^256 - 1` as tokens left.

Only when a cap exists:

- `remaining` of 0: this trader's inner-market buy quota is used up. Stop. Do not shrink the order or split it.
- The `quoteExactInput` result is larger than `remaining`: the user may not get the full quote. The first simulation in `references/swap.md` uses `remaining` as the floor basis, so a swap that does cap still passes. If that simulation returns more than `remaining`, the quota view did not reduce the swap: the floor moves up to the simulated output, and the swap is simulated again. If it returns no more than `remaining`, the floor stays on `remaining`. Say the website treats unused input as refunded. Check the receipt before calling it a full fill.

Going over the quota does not always revert. That is why the simulation decides the final floor. Do not lower the floor and then skip the simulation. `references/swap.md` "Floor" and "Simulate" are the only source for this order.

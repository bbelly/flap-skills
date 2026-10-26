# Swap

Every buy and sell in this skill is Portal `swapExactInput`. The selector is `0xef7ec2e7`. `permitData` is `0x`. Do not build `swapExactInputV3`. Do not pass a fixed gas limit. The website's old `600000` gas figure is not an argument to copy.

Sections run in this order: approve check, simulation, confirmation, send. Every command that sends a transaction is in the last section, "Send". Do not run one before its own confirmation.

## Fields

| Field | Buy with BNB | Buy with quote token | Sell |
|---|---|---|---|
| `to` | Portal | Portal | Portal |
| `inputToken` | zero address | quote token | Flap token |
| `outputToken` | Flap token | Flap token | quote token, or zero address if the quote is BNB |
| `inputAmount` | BNB wei | quote-token base units | token base units |
| `minOutputAmount` | see "Floor" below | see "Floor" below | see "Floor" below |
| `permitData` | `0x` | `0x` | `0x` |
| `value` | same as `inputAmount` | `0` | `0` |

The zero address is `0x0000000000000000000000000000000000000000`.

Buy with BNB is this one call for both a native-quoted token and an ERC-20-quoted token whose `nativeToQuoteSwapEnabled` is true. Portal converts BNB internally. Do not buy the quote token first, and do not approve it. A later sell still returns that quote token, not BNB. Say that in the confirmation.

## Approve check, only for an ERC-20 input

BNB buys skip this section.

```bash
cast call $INPUT_TOKEN "allowance(address,address)(uint256)" $TRADER 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 --rpc-url $RPC
```

- At least `inputAmount`: no approve. Go to "Simulate".
- Already `type(uint256).max`: say the approval is already unlimited. Do not increase it. Go to "Simulate".
- Below `inputAmount`: the next transaction is only an approve of exactly `inputAmount`. Do not simulate the swap yet, and do not attach it to the same confirmation.

Simulate the approve before asking:

```bash
cast call $INPUT_TOKEN "approve(address,uint256)(bool)" \
  0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 $INPUT_AMOUNT \
  --from $TRADER --rpc-url $RPC
```

If it reverts while the current allowance is above 0, the token may require a reset to 0 first (USDT-style). Then the next transaction is `approve(Portal, 0)` alone, with its own confirmation and its own receipt. After that, simulate the exact approve again. Do not reset an allowance that is already 0, and do not reset without that revert. Any other revert stops the order.

The approve confirmation shows: the token being approved (full address and symbol), the spender (Portal, full address), the amount in base units and in human units, and the sentence "Exact amount, not unlimited." For Binance Agentic Wallet, run the approve preview from "Send" first and show its fields too. Then go to "Confirm". After the approve is sent, wait for a receipt with status 1, read `allowance` again, and confirm it is at least `inputAmount`. Then leave this file: read `references/quote.md`, quote again, come back to "Floor" and "Simulate", and `cast call` the swap on that new quote. Do not open "Confirm" for the swap until that second `cast call` succeeds. A quote is not a simulation. On a sell, reuse the approved amount. Do not pass "all" again: a reflection token's balance can grow past the approval.

## Floor

Read numbers from `cast` as the first integer on the line only. `cast` prints `1710194868288753060294554 [1.71e24]`. The bracket is display. Do the math with integers (shell `$(( ))` overflows; use Python `int` or JavaScript `BigInt`), never floats.

```text
minOutputAmount = basis * (10000 - slippageBps) / 10000   (round down)
```

Pick the first `basis` this way:

| Case | First `basis` |
|---|---|
| No buy cap, or a sell | the `quoteExactInput` result |
| A buy cap exists and the quote is no larger than `remaining` | the `quoteExactInput` result |
| A buy cap exists and the quote is larger than `remaining` | `remaining` |

"Buy cap" is defined in `references/tax-and-quota.md`. If `minOutputAmount` is 0, stop.

## Simulate

Call the same function with `cast call`, from the trader, with the real value. Do not override balance or allowance. The trader must already hold the input and, for an ERC-20 input, the allowance.

```bash
cast call 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "swapExactInput((address,address,uint256,uint256,bytes))(uint256)" \
  "($INPUT_TOKEN,$OUTPUT_TOKEN,$INPUT_AMOUNT,$MIN_OUT,0x)" \
  --value $VALUE \
  --from $TRADER \
  --rpc-url $RPC
```

A revert stops the order. Do not widen the floor, and do not show a send summary. Show the revert text, and wait for the user to change the order or the slippage.

The returned integer is the simulated output. Then:

- First `basis` was `remaining`, and the simulated output is greater than `remaining`: the swap did not apply the quota cap. Set `basis` to the simulated output, recompute `minOutputAmount`, and run the `cast call` once more with that floor. A revert stops the order. Tell the user the quota view did not reduce the swap.
- First `basis` was `remaining`, and the simulated output is no greater than `remaining`: keep the floor. Tell the user the buy is capped by the quota, that they get at most `remaining`, and that unused input may be refunded.
- First `basis` was the quote: keep the floor. If the simulated output is below the quote, show both numbers.

A successful simulation is not permission to send.

## Calldata

Build the calldata from the final fields. The same hex is what the user confirms and what is sent.

```bash
cast calldata "swapExactInput((address,address,uint256,uint256,bytes))" \
  "($INPUT_TOKEN,$OUTPUT_TOKEN,$INPUT_AMOUNT,$MIN_OUT,0x)"

cast calldata "approve(address,uint256)" 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 $INPUT_AMOUNT
```

The swap calldata must start with `0xef7ec2e7`. The approve calldata must start with `0x095ea7b3`, and its amount must not be `type(uint256).max`.

## Confirm

Do not open this section for a swap until "Simulate" has succeeded on the current quote. After an approve, that means the post-approve quote and the second `cast call`, not the pre-approve numbers.

Each transaction gets its own summary and its own confirmation: one for the approve, one for the swap.

The swap summary shows `to`, `value`, every `swapExactInput` argument, the calldata, the quote, the simulated output, `minOutputAmount` and its basis, slippage, both tax rates, the buy quota, `version()`, the quote block, and the audit status. For a 15% default, include the MEV sentence from `references/quote.md`.

For Binance Agentic Wallet, run the preview from "Send" before asking. From a successful preview, add all of these to the summary:

- `data.requestId`
- `data.parsedTx` — `parsedTx.amount` is the smallest unit (wei for BNB). Divide by `decimals` from `data.tokenInfos` before showing a human amount.
- `data.simulationResult.balanceChanges`
- `data.simulationResult.allowanceChanges`
- `data.simulationResult.authorityChanges`
- `data.risks`
- `data.requireConfirmation`
- `data.expiresAt`

Those three change lists live under `simulationResult`, not at the top of `data`. If `simulationResult` is missing, the preview is unusable: stop. If the preview disagrees with your summary (another `to`, another amount, an unexpected allowance or authority change), stop and show both. A preview without `requestId`, or `351803`, stops the order.

Only a reply sent after that summary (and, for BAW, after that preview) counts. The reply must be just `yes`, `confirm`, `go ahead`, `确认`, or `同意`. A question, a condition, or a longer sentence that contains one of those words is not a confirmation. Anything earlier does not count.

When the reply arrives, read the chain time before acting on it:

```bash
cast block latest --field timestamp --rpc-url $RPC
```

If that is more than 60 seconds after `$QUOTE_TS` from `references/quote.md`, or the BAW preview expired: the confirmation is void. Quote again, set the floor again, simulate again, preview again, and ask again. Never send a call that differs from the one the user confirmed.

## Send, only after that confirmation

Own signer. Use a hardware wallet (`--ledger` or `--trezor`) or a Foundry keystore (`--account <name>`). Browser wallets and WalletConnect are out of scope. Do not use `--private-key`, `--mnemonic`, `--interactive`, or a password on the command line. Do not add `--gas-limit`. Print the `cast send` command. Do not run it from the agent shell: the password or device confirm is interactive. The user runs it in their own terminal and pastes the hash.

The printed command has no shell variables. The user's terminal does not have them. Substitute every value as a literal: addresses in full, amounts as base-unit integers, the RPC URL, and the signer flag. The arguments must match the confirmed summary character for character. Do not print a keyed RPC URL that holds a secret; use the public fallback from `references/preflight.md` instead.

Always include `--from <trader>`. Quote, quota, balance, and simulation were all read for that address. If the hardware wallet path or keystore resolves to another address, `cast` refuses to send. For a Ledger or Trezor account that is not the default path, also pass the user's `--mnemonic-derivation-path`; ask for it, do not guess.

Tell the user to run it now. If the command is not run within 60 seconds of `$QUOTE_TS`, do not run it: come back, and the flow restarts at Step 4. The swap has no deadline argument; `minOutputAmount` is the only on-chain guard after that.

```bash
# Approve, after the approve confirmation (literal values):
cast send <input token> \
  "approve(address,uint256)" \
  0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 <input amount> \
  --from <trader> --chain 56 --rpc-url <rpc> <signer flag>

# Swap, after the swap confirmation (literal values):
cast send 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "swapExactInput((address,address,uint256,uint256,bytes))" \
  "(<input token>,<output token>,<input amount>,<min out>,0x)" \
  --value <value> \
  --from <trader> --chain 56 --rpc-url <rpc> <signer flag>
```

Binance Agentic Wallet. Do not use `cast send`. The preview runs before the confirmation. Only `execute` runs after it.

```bash
# Before the confirmation:
baw contract-call preview --binanceChainId 56 --from $TRADER \
  --to <Portal for the swap, the input token for the approve> \
  --value <wei integer> --inputData <calldata from "Calldata"> --json

# After the confirmation:
baw contract-call execute --requestId <requestId from that preview> --json
```

No `--gasLimit` and no gas-price flags. A `requestId` from another chat, another preview, or an older preview is dead. See `references/errors.md`.

Do not change `to`, `value`, or the calldata after confirmation. Then go to `references/receipt.md`.

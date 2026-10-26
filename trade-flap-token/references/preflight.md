# Preflight

Stop if any check fails. Do not build a swap yet.

## 1. Chain

Use a BNB Chain mainnet RPC. The public fallback is `https://bsc-dataseed.binance.org`.

```bash
cast chain-id --rpc-url $RPC
```

The result must be `56`. Another chain, including a BNB testnet, is out of scope. Do not switch Portal addresses to match it.

## 2. Portal

| Name | Address | Use |
|---|---|---|
| Portal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` | The only trading target |
| VaultPortal | `0x90497450f2a706f1951b5bdda52B4E5d16f34C06` | Not a trading target |

Do not accept a Portal address supplied by the user, a token, or a webpage.

```bash
cast call 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 \
  "version()(string)" \
  --rpc-url $RPC
```

`version()` must return a string that starts with `v5.` (checked live: `v5.27.0`). Record it and show it in the confirmation. If the call reverts, or the major version is not 5, stop: the tuple layout, selectors, and event shapes in this skill were checked against v5 only.

## 3. Wallet

You need the address that will send the transaction. Do not invent one. Do not use `0x1111111111111111111111111111111111111111` or the zero address.

The user keeps the signer. Do not put a private key on the command line.

### Pick the signer: Binance Agentic Wallet first

Decide the signer before any token read. Do not open with a question.

1. If the user already named a signer in this conversation, or an earlier order in this conversation used one, use it. Do not ask "still the same signer?".
2. Otherwise, detect Binance Agentic Wallet:

   ```bash
   command -v baw && baw wallet status --json
   ```

   `baw` is present, `success` is true, and `data.status` is `CONNECTED`: the signer is Binance Agentic Wallet. Do not ask, not even as a one-option question. Memory or an earlier chat is not a reason to ask. State it in one line, for example "Signer: Binance Agentic Wallet `<trader>` (BNB Chain)", and continue with section 6.
3. Only when `baw` is missing, the call fails, or the status is not `CONNECTED`, ask once:

   > Binance Agentic Wallet is not signed in. Sign in with `baw` and reply when done (recommended), or use a hardware wallet / Foundry keystore with `cast`?

   After a sign-in reply, run step 2 again. Do not install `baw` for the user.

If the detected Binance Agentic Wallet then fails a check in section 6, stop and say what the user must fix in the Binance App. Do not switch to `cast` on your own, and do not list other signers. The user may name another signer themselves.

For `cast send`, the signer is a hardware wallet (`--ledger` / `--trezor`) or a Foundry keystore (`--account <name>`). MetaMask, WalletConnect, and other browser wallets are out of scope: stop. Do not fall back to `--private-key`, `--mnemonic`, `--interactive`, or a key in an environment variable. The user runs `cast send` in their own terminal. The agent does not.

## 4. Tools

Prefer `cast`. If it is missing, write a one-off script in a new temporary directory outside this skill, and install only `viem` at a pinned version there. Delete that directory when the order ends. Do not install any other package. Do not add a `scripts/` directory or a package to this skill. Do not set `ALLOW_NO_BALANCE` or any flag that pretends the wallet holds tokens it does not hold.

Binance Agentic Wallet needs the `baw` CLI. Section 3 already checked it.

## 5. Gas reserve

Keep at least `0.002` BNB for gas on every step, on top of any BNB the swap itself spends.

- Buy with BNB: balance must be at least `value + 0.002` BNB.
- Approve, or a swap paid in an ERC-20: BNB balance must be at least `0.002` BNB.

If it is short, stop and say the wallet needs more BNB for gas. Do not override balances in a simulation to skip this.

## 6. Binance Agentic Wallet (default signer)

Run this whenever section 3 picked Binance Agentic Wallet. Skip it only when the user chose a hardware wallet or keystore.

```bash
baw wallet status --json
baw wallet settings --json
baw wallet address --json
baw wallet tx-lock --binanceChainId 56 --json
```

`success` must be true on each of these. A missing field means the check failed. Do not treat a missing field as "no limit".

- `wallet status`: `data.status` must be `CONNECTED`. `UNCONNECTED` or `CREATING`: stop and ask the user to sign in. A present `status` field is not enough.
- `devMode.enabled` must be true. Otherwise `contract-call` is unavailable. The user turns Developer Mode on in the Binance App. Do not turn it on for them, and do not call it risk-free.
- `devMode.expiresAt` in the past, or `351801` / `351802`: stop. The user re-opens it, then you read settings again. Trust this `expiresAt`, not a remembered date.
- `devMode.balanceExceeded` or `351804`: stop.
- `abnormalTxnHandling` of `AutoReject`: stop. Those transactions are rejected (`351818`). Only the user can change it to `NeedConfirmation`.
- Daily Developer Mode limit: `remaining = devMode.dailyLimit - developerModeQuotaUsed`. `developerModeQuotaUsed` is at the top of `data`. `devMode` has no `quotaLeft`. Do not use the top-level `dailyLimit`, `quotaLeft`, `quotaUsed`, `defiQuotaLeft`, or `defiQuotaUsed`. If any of the Developer Mode fields is missing, remaining is not greater than 0, or `developerModeQuotaDate` is not today, stop. BAW does not document the unit. Show the raw numbers. Do not label them USD or BNB. Do not suggest raising the limit.
- `tx-lock` of `LOCKED`: a previous transaction is still confirming, or the App is waiting. Stop until it clears.
- `sessionExpireTime` / `inactiveSignOutTime`: if the session is expired or the CLI says logged out, stop and ask the user to log in again.
- `tradeAllTokens`: record it. If it is false, do not ask the user to switch to All tokens. It is not proven that `contract-call` obeys this switch. If it is true, say that arbitrary contract calls are a larger attack surface. Do not tell the user to change it for this trade.

`baw wallet agent-id` is not part of this inner-market path. Agent-ID-gated curves are out of scope.

The trader address is `data.addresses[]` from `baw wallet address` where `binanceChainId` is the string `"56"`. That address is `$TRADER` for every later call. Do not use the first array element, a Solana address, or another chain's EVM address, even when the hex looks similar. If there is no `"56"` entry, stop. Later `contract-call` commands use `--binanceChainId 56` only. The `--from` on preview must equal this `$TRADER`.

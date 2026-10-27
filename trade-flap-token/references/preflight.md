# Preflight

Stop if any check fails. Do not build a swap yet.

## 1. Chain

Use a BNB Chain mainnet RPC. The public fallback is `https://bsc-dataseed.binance.org`.

Read the chain ID from that RPC (`eth_chainId`, or your library's `getChainId()`). The result must be `56` (`0x38`). Another chain, including a BNB testnet, is out of scope. Do not switch Portal addresses to match it.

Every read and simulation in this skill is an `eth_call` against that chain-56 endpoint. Any transaction this skill hands to a signer carries `chainId: 56`. Do not point the RPC at another chain.

## 2. Portal

| Name | Address | Use |
|---|---|---|
| Portal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` | The only trading target |
| VaultPortal | `0x90497450f2a706f1951b5bdda52B4E5d16f34C06` | Not a trading target |

Do not accept a Portal address supplied by the user, a token, or a webpage.

Call `version()` (selector `0x54fd4d50`, returns `string`) on Portal. See `references/interface.md`.

`version()` must return a string that starts with `v5.` (checked live through `v5.27.2`). Record it and show it in the confirmation. If the call reverts, or the major version is not 5, stop: the tuple layout, selectors, and event shapes in this skill were checked against v5 only. Agent ID `nftGate` needs a Portal that exposes that selector (v5.27.0+); feature-detect in `references/token-state.md`.

## 3. Wallet

You need the address that will send the transaction. Do not invent one. Do not use `0x1111111111111111111111111111111111111111` or the zero address.

The user keeps the signer. Do not put a private key on the command line.

### Pick the signer: Binance Agentic Wallet first

Detect BAW before any token read. Do not open with a question. **Do not finalize a non-BAW signer** until Step 2 has chosen path A. Paths B (Agent ID-gated) and C (graduated) are BAW-only — see `references/token-state.md`.

1. Detect Binance Agentic Wallet:

   ```bash
   command -v baw && baw wallet status --json
   ```

   `baw` is present, `success` is true, and `data.status` is `CONNECTED`: provisional signer is BAW. Do not ask. State it in one line, for example "Signer: Binance Agentic Wallet (pending route)", and continue with section 6. After Step 2, confirm the same line with the route (A/B/C).
2. If the user already named BAW, or an earlier order in this conversation used BAW, keep BAW. Do not ask "still BAW?".
3. If the user already named a hardware wallet or keystore signer in this conversation: remember it only as a **path-A fallback**. If Step 2 later chooses B or C, ignore that choice, stop, and say this route requires Binance Agentic Wallet.
4. Only when BAW is missing or not `CONNECTED`, **and** Step 2 has already shown path A, ask once:

   > Binance Agentic Wallet is not signed in. Sign in with `baw` and reply when done (recommended), or sign the transaction yourself with a hardware wallet or keystore?

   If Step 2 is not done yet and BAW is down: continue reading token state. If the route is B or C, stop with a sign-in-only message (no other signer offered). If the route is A, then ask the question above.

After a sign-in reply, run the BAW detect again. Do not install `baw` for the user.

If BAW fails a check in section 6 on a path that requires BAW (always on B/C; on A when BAW was selected), stop and say what to fix in the Binance App. Do not switch to another signer on your own.

Non-BAW signer, path A only: a hardware wallet (Ledger / Trezor) or an encrypted keystore the user controls. MetaMask, WalletConnect, and other browser wallets are out of scope. The agent never holds the key: do not accept a private key, mnemonic, or key in an environment variable. The agent hands over the exact transaction (`to`, `value`, `data`, chain `56`); the user signs and broadcasts it with their own tool and pastes the hash back. See `references/swap.md` "Send".

## 4. Tools

This skill needs two capabilities. It does not require any particular program for either.

1. **EVM reads and simulations** — an `eth_call` against the chain-56 RPC, with `from`, `value`, and an optional pinned block, plus reading blocks, receipts, and transactions. Any of these works: `viem` / `ethers` / `web3.py`, Foundry `cast`, or raw JSON-RPC. `references/interface.md` has the ABIs and selectors.
2. **ABI encoding** — build `swapExactInput` / `approve` calldata from the confirmed fields. Any ABI encoder works; the selectors in `references/interface.md` are the check.

Use what is already available. If nothing is, write a one-off script in a new temporary directory **outside** this skill, installing a single EVM library at a pinned version (`viem` is what `launch-bnb-token-on-flap` uses). Delete that directory when the order ends. Do not add a `scripts/` directory or a package to this skill. Do not set `ALLOW_NO_BALANCE` or any flag that pretends the wallet holds tokens it does not hold.

Do **not**:

- Grep / scrape agent logs, chat history, or random files looking for past call output or ABIs
- Pretend on-chain reads succeeded without actually making the `eth_call`
- Decode return data by guessing — decode against the types in `references/interface.md`

If no eth_call capability can be set up at all, say so and stop the order.

```bash
command -v baw   # required for paths B/C; strongly preferred for A
```

### `baw` missing (paths B/C, or A when BAW is the chosen signer)

Stop and tell them to install / open Binance Agentic Wallet CLI so `baw` is on `PATH`. Do not offer another signer as a substitute on B/C. Section 3 already covers sign-in.

Binance Agentic Wallet needs the `baw` CLI. Section 3 already checked connectivity when present.

## 5. Gas reserve

Keep at least `0.002` BNB for gas on every step, on top of any BNB the swap itself spends.

- Buy with BNB: balance must be at least `value + 0.002` BNB.
- Approve, or a swap paid in an ERC-20: BNB balance must be at least `0.002` BNB.

If it is short, stop and say the wallet needs more BNB for gas. Do not override balances in a simulation to skip this.

## 6. Binance Agentic Wallet (default signer)

Run this whenever the signer is Binance Agentic Wallet (provisional or final). Skip it only for a finalized path-A non-BAW signer.

```bash
baw wallet status --json
baw wallet settings --json
baw wallet address --json
baw wallet tx-lock --binanceChainId 56 --json
```

`success` must be true on each of these. A missing field means the check failed. Do not treat a missing field as "no limit".

- `wallet status`: `data.status` must be `CONNECTED`. `UNCONNECTED` or `CREATING`: stop and ask the user to sign in. A present `status` field is not enough.
- **Before Step 2 chooses a route:** read settings and address, but do **not** fail the order solely because Developer Mode is off or expired. Path C does not need it.
- **After Step 2, paths A/B only** (Portal `contract-call`): `devMode.enabled` must be true. Otherwise stop. The user turns Developer Mode on in the Binance App. Do not turn it on for them. Path C (`market-order`) never requires Developer Mode.
- Paths A/B after Step 2: `devMode.expiresAt` in the past, or `351801` / `351802`: stop. Re-read settings after the user re-opens it. Trust this `expiresAt`, not a remembered date.
- Paths A/B after Step 2: `devMode.balanceExceeded` or `351804`: stop.
- `abnormalTxnHandling` of `AutoReject`: stop. Those transactions are rejected (`351818`). Only the user can change it to `NeedConfirmation`.
- Paths A/B daily Developer Mode limit: `remaining = devMode.dailyLimit - developerModeQuotaUsed`. `developerModeQuotaUsed` is at the top of `data`. `devMode` has no `quotaLeft`. Do not use the top-level `dailyLimit`, `quotaLeft`, `quotaUsed`, `defiQuotaLeft`, or `defiQuotaUsed`. If any of those Developer Mode fields is missing, remaining is not greater than 0, or `developerModeQuotaDate` is not today, stop. Show raw numbers. Do not label them USD or BNB. Do not suggest raising the limit.
- `tx-lock` of `LOCKED`: stop until it clears.
- `sessionExpireTime` / `inactiveSignOutTime`: if expired or logged out, stop and ask the user to log in again.
- `tradeAllTokens`: record it. If false, do not ask the user to switch to All tokens. Whether `contract-call` or `market-order` obeys this switch is not proven. If true, say arbitrary contract calls are a larger attack surface. Do not tell the user to change it for this trade.

`baw wallet agent-id` is not used to detect path B. Path B uses on-chain `nftGate(token)` and `balanceOf` on that collection — see `references/agent-id-gate.md`.

The trader address is `data.addresses[]` from `baw wallet address` where `binanceChainId` is the string `"56"`. That address is `$TRADER` for every later call. Do not use the first array element, a Solana address, or another chain's EVM address. If there is no `"56"` entry, stop. Portal `contract-call` uses `--binanceChainId 56` only; `--from` on preview must equal this `$TRADER`. Market-order commands also use `--binanceChainId 56`.

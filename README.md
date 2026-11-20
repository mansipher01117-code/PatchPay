# PatchPay — bounties that are already paid for

> **BCH BLAZE 2026 · Mainnet Track** · Deck: `<add Widget Factory link>` · Live demo: `<add Vercel URL>` · EMBER2026 votes → `<add token-aware bitcoincash:z… address>` · All code written during BLAZE (from 14 Nov 2026).

**Problem:** open-source bounties are full of rewards that never get paid — maintainers ghost, "symbolic" prizes, no proof the money exists — and now of low-effort AI pull requests. **Solution:** a maintainer locks Bitcoin Cash into a per-issue escrow covenant, so the funds are provably there *before* anyone starts work. When the PR is merged, CI is green and the maintainer approves, the contract pays the exact contributor and stamps a **PatchProof** hash on-chain; if nobody delivers by the deadline, anyone can push the money back to the maintainer.

## The full flow

1. **Maintainer posts** — pastes a GitHub issue on `/new`. The Issue Scout (Gemini) summarises it and drafts testable acceptance criteria.
2. **Funds via WizardConnect** — PatchPay derives a one-off `PatchEscrow` address for that issue; the maintainer funds it from Paytaca / Cashonize / Selene. The balance is public on the docket.
3. **Contributor opens a PR** with `Fixes #N` and a `PatchPay-Address:` line (add `PatchPay-Agent: true` if you're an AI). PatchPay comments the live bounty on the PR.
4. **Merge → PatchProof** — on merge PatchPay reads GitHub directly (merge commit, head SHA, diff SHA-256, every check run) plus a sceptical AI review, and hashes it all into a canonical PatchProof posted on the PR. (CI finished late? The `check_suite` webhook or the **Refresh proof** button rebuilds it.)
5. **Merge = approval → contract pays** — when the merged PR has green CI, a valid payout address, an AI review that finds the criteria met, and was merged by a maintainer who is not its author, the oracle signs `(bountyId, recipientLock, proofHash)` automatically; the covenant verifies the signature, pays that exact address and writes the proof hash into an `OP_RETURN`. If any condition fails, the PR comment says why and a maintainer can approve manually. (`AUTO_PAYOUT_ON_MERGE=0` turns auto-pay off.)
6. **Refund after deadline** — nobody delivered? Anyone can trigger the refund; the covenant only lets funds return to the maintainer.

## Try it in 2 minutes (local, no keys)

```bash
cp .env.example .env.local      # leave it blank to run in demo mode
npm install
npm run dev                     # http://localhost:3000
```

With no Supabase/GitHub keys the app runs on a seeded in-memory store and a `demo-maintainer` login:

1. Open **`/`** — the ledger shows three seeded bounties.
2. Click any docket (e.g. **PP-A1B2**) → watch the escrow receipt read the chain and render the top-up **QR**.
3. Click a contributor like **@bch-builder** → their earnings profile at `/u/bch-builder`.
4. Open a PatchProof (`/proof/<hash>`) → click **Verify in my browser** to re-hash it with WebCrypto.
5. Visit **`/dev/log`** to tail the event audit trail.

## The contract, line by line — `contracts/PatchEscrow.cash`

Constructor params: `bytes32 bountyId` (sha256 of the bounty UUID), `pubkey oraclePk`, `bytes maintainerLock` (refund target), `int deadline`.

`release(datasig oracleSig, bytes recipientLock, bytes32 proofHash)`
- `require(tx.inputs.length == 1)` — one UTXO per tx, so each payout is atomic.
- `require(checkDataSig(oracleSig, bountyId + recipientLock + proofHash, oraclePk))` — only the PatchPay oracle can authorise a release, and only for this exact bounty + recipient + proof.
- `require(tx.outputs[0].lockingBytecode == recipientLock)` — pays the signed recipient and nobody else.
- `require(tx.outputs[0].tokenCategory == 0x)` — the payout is plain BCH.
- `require(tx.outputs[0].value >= tx.inputs[...].value - 3000)` — caps the fee at 3000 sats so the contributor is not shortchanged.
- `require(tx.outputs[1].lockingBytecode == new LockingBytecodeNullData([0x50505046, proofHash]))` — output 1 **must** be an `OP_RETURN` publishing `PPPF` + the proof hash on-chain.
- `require(tx.outputs.length == 2)` — exactly payout + proof, nothing hidden.

`refund()`
- `require(tx.inputs.length == 1)` and `require(tx.time >= deadline)` — only after the deadline.
- `require(tx.outputs[0].lockingBytecode == maintainerLock)` — refunds can only ever go to the maintainer.
- `require(tx.outputs[0].value >= tx.inputs[...].value - 3000)` and `require(tx.outputs.length == 1)`.

No user signature is required inside the contract, which keeps funding compatible with WizardConnect.

## Trust model

- The contract **cannot pay anyone the oracle did not sign for**, and refunds **only** reach the maintainer.
- The oracle is a server-only key. It signs **only after**: PR merged (re-checked live at approval), all checks green, maintainer approval, and approver ≠ PR author.
- The AI never holds keys and cannot approve; it only fills the review section of the proof.
- Everything verifiable is public: escrow address, oracle pubkey, every PatchProof (re-hash it in your own browser), and the `OP_RETURN` in each payout.
- Roadmap: 2-of-3 oracle (maintainer, PatchPay, independent verifier); PatchProof CashToken NFT receipts (see `scripts/create-receipt-category.mjs`).

## Setup (production)

```bash
cp .env.example .env.local
npm install
npm run oracle:key        # prints ORACLE_PRIVATE_KEY_HEX — paste into .env.local
# Supabase: run supabase/schema.sql in the SQL editor
npm run test:contract     # expect: 5 passed, 0 failed
npm run dev
```

Every environment variable:

| Var | Where | Notes |
|---|---|---|
| `NEXT_PUBLIC_BCH_NETWORK` | public | `chipnet` while building, `mainnet` for the demo |
| `ORACLE_PRIVATE_KEY_HEX` | **server** | from `npm run oracle:key`; never commit |
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | public | Supabase project API |
| `SUPABASE_SERVICE_ROLE_KEY` | **server** | writes rows; bypasses RLS |
| `GITHUB_APP_ID` / `GITHUB_APP_PRIVATE_KEY` / `GITHUB_WEBHOOK_SECRET` | **server** | GitHub App; PEM uses `\n` for newlines |
| `GEMINI_API_KEY` / `GEMINI_MODEL` | **server** | Google AI Studio; defaults to `gemini-2.5-flash` |
| `NEXT_PUBLIC_SITE_URL` | public | used in PR comment links |
| `NEXT_PUBLIC_REPO_URL` / `NEXT_PUBLIC_BLAZE_DECK_URL` / `NEXT_PUBLIC_EMBER_ADDRESS` | public | BLAZE banner |
| `NEXT_PUBLIC_SHOW_DEV_LOG` | public | set `1` to expose `/dev/log` in production |
| `RECEIPT_WALLET_WIF` / `RECEIPT_CATEGORY` | **server** | optional: PatchProof NFT receipts (stretch) |

Without the server keys the app falls back to demo mode (in-memory DB, simulated scout/release) so judges can click through everything offline.

## Build on chipnet, demo on mainnet

Develop on **chipnet** (`NEXT_PUBLIC_BCH_NETWORK=chipnet`, `bchtest:` addresses, Paytaca chipnet mode, faucet: tbch.googol.cash). Flip the single env var to `mainnet` for the demo with a small real bounty (≈0.001 BCH).

## Mainnet transactions

| What | TXID |
|---|---|
| Escrow funding | `<add mainnet txid>` |
| Payout + PatchProof `OP_RETURN` | `<add mainnet txid>` |
| Refund (if shown) | `<add mainnet txid>` |

Contract tests: `scripts/escrow.test.mjs` (forged signature, wrong recipient, early refund all revert) via `npm run test:contract`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for PatchPay claim instructions, including the PR body format and payout requirements.

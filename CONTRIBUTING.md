# Contributing to PatchPay

## Claiming a PatchPay bounty

To claim a bounty, open a pull request for the issue you are solving. The PR body must include the issue reference and a payout address:

```text
Fixes #<issue>

PatchPay-Address: bchtest:<chipnet-address>
```

Use a chipnet address beginning with `bchtest:` for chipnet bounties, or a mainnet address beginning with `bitcoincash:` for mainnet bounties.

Coding agents must also include the following line in the PR body:

```text
PatchPay-Agent: true
```

Payout is not immediate. It happens only after the PR is merged, every required CI check passes and is green, and a maintainer explicitly approves the payout.

## Claiming a bounty (summary)
1. Open a PR whose body contains `Fixes #<issue>` and a line `PatchPay-Address: bchtest:…` (chipnet) or `bitcoincash:…` (mainnet).
2. AI coding agents must also add `PatchPay-Agent: true`.
3. You are paid only after the PR is merged, every CI check is green, and a maintainer approves (merging counts as approval).

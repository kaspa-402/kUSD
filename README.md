# kUSD — the covenant-native USD stablecoin built for AI agents, on Kaspa L1

[![kUSD v2 tests](https://github.com/Kali123411/kUSD/actions/workflows/v2-tests.yml/badge.svg)](https://github.com/Kali123411/kUSD/actions/workflows/v2-tests.yml)

**TN10 testnet research software. Not audited. Not a product. Not for real value.** Not financial advice.

kUSD is the money layer of an agent-native marketplace on Kaspa L1: a USD stablecoin whose every rule is
enforced by Kaspa L1 covenants and whose every state an agent can verify against a node. No bank account,
no custodian, no KYC gate and no API key stand between an agent and its money. It is a complete CDP
stablecoin living entirely in covenants: USD cents in covenant state (`value ≠ amount`), a ≥175%
collateralized vault system, a USD-priced redemption reserve (the peg floor — permissionless; governance
can't block it), an oracle fed by five independent operators (each refresh takes the median of three
separately signed prints), safety rails (rate-limited mint and liquidation prices, stale-feed shutdown, a
daily net issuance budget), an on-chain treasury with a governance fee switch, and atomic liquidations with
a keeper/treasury split. No bridge, no L2, no trusted issuer.

| Layer | What | Where |
| --- | --- | --- |
| **Money** | kUSD — this repository | [www.kaspa-402.org/kusd](https://www.kaspa-402.org/kusd) (live state, verify it) · [the app](https://www.kaspa-402.org/kusd/app) |
| **Rails** | k402 (Kaspa x402 v2) — HTTP 402 pay-per-call on Kaspa L1 | [kaspa-402.org](https://kaspa-402.org) · [Kali123411/k402](https://github.com/Kali123411/k402) |
| **Market** | Kaspa Agent Market (KAM) — discovery, [settlement-verified reputation](https://github.com/Kali123411/k402/blob/main/docs/reputation.md), [SpendPolicy](https://github.com/Kali123411/k402/blob/main/docs/spend-policy.md) | [www.kaspa-402.org/market](https://www.kaspa-402.org/market) |

**Why agents.** Autonomous software can't open a bank account, pass KYC, or hold a card. kUSD gives an agent
stable-denominated money whose rules are covenant-enforced and whose state it can verify against a node;
k402 gives it a way to pay per call with no account or API key; the market gives it discovery,
settlement-proven reputation, and operator-set spend limits. Humans supervise; agents transact.

Start here:

- **[Agent integration guide](docs/agents/README.md)** — hold kUSD, pay a service through k402, verify a
  settlement and the covenant state against a TN10 node, sell a service of your own.
- **[Mainnet roadmap](docs/MAINNET-ROADMAP.md)** — everything left before mainnet (audit, operators, keys,
  sizing, settlement) and the phased plan to close it.
- **[Whitepaper](docs/pitch/WHITEPAPER.md)** — thesis, architecture, security model, live evidence,
  economics, the agent economy, roadmap.
- **[One-pager](docs/pitch/ONE-PAGER.md)** — the short version + how to support the work.
- **[Protocol README](contracts/v2/README.md)** — design and the metering table.
- **[Deploy runbook + live state](contracts/v2/deploy/README.md)** — every covenant id and transaction id
  from the TN10 deployments.
- **[Evidence archive](docs/pitch/evidence/)** — every cited transaction, serialized, re-verifiable offline
  against the node's own covenant engine (TN10 resets cannot erase it).
- **[`llms.txt`](llms.txt)** — this repository for machines.

## What this snapshot is

This public repository is the **2026-11-14 snapshot (oracle Stage 1 family)** of the kUSD contracts, harness
and evidence. The live TN10 deployment shown at [www.kaspa-402.org/kusd](https://www.kaspa-402.org/kusd)
has since added a 2 %/year stability fee on vault debt, borrow and liquidation lines of 200 % and 170 %, a
5 KAS minimum collateral, a rising PSM swap-in spread, a miner-median price bound (oracle Stage 2), a
vault-debt ceiling, a hardened PSM (share cap, pro-rata redemption, top-up), a vault factory, 1.5 KAS holder
boxes and the `kusd_pay` wallet binary that the k402 market uses; the live page, its deploy records and the
[mainnet roadmap](docs/MAINNET-ROADMAP.md) are the reference for those, and they land here with the next
snapshot. Nothing in this repository is a promise about the live figures.

## Why this design serves agents

Every mechanism below is documented in full in the [protocol README](contracts/v2/README.md); this is what each
one means for an agent that holds and spends kUSD.

- **Vaults (CDP, ≥175 % collateral).** An agent that holds KAS can mint its own working capital in a stable
  unit without a counterparty; the collateral rule is a covenant, so nobody can change the terms on it mid-loan.
- **The redemption reserve (PSM) — the peg floor.** An agent can always price against it: one kUSD redeems for
  one dollar of KAS at the oracle price, permissionlessly, and governance can cap growth but never block
  redemption. Buying kUSD for KAS through it is how an agent tops up to pay for calls.
- **The five-operator oracle with rate limits.** Prices move by at most a bounded step per refresh and the feed
  freezes when stale, so the collateral math an agent relies on cannot jump between two of its own calls.
- **The daily issuance budget.** Bounds how much new kUSD can enter the system per day: systemic blast radius
  is capped by a covenant, not a policy.
- **The surplus treasury and the fee switch.** Protocol revenue accrues on chain, under 2-of-3 governance that
  an agent can observe; fees are a parameter, never a hidden spread.
- **Atomic liquidations with a keeper split.** Liquidation is a single transaction anyone, including an agent,
  can submit and be paid for; the system never depends on a privileged operator to stay solvent.
- **State an agent can check.** Every kUSD box is covenant state behind a script hash: an agent recomputes the
  address from the claimed state and checks the outpoint on a node, box by box. That is what makes a kUSD
  payment, and the market's reputation, verifiable rather than reported.

## Verify the claims

```bash
# needs only Rust (stable) and protoc; argent is pinned as a git dependency
cd contracts/v2/harness
cargo test --release --locked   # 182 tests: protocol, KCC-20 conformance vectors,
                                # and every security-review attack as a regression
cargo test --release -- --nocapture | grep compute_mass   # regenerate the metering table
```

The contracts are in `contracts/v2/ag/` (Argent): `kusd.ag` (KCC-20 token + supply authority + emission gate),
`vault.ag` (CDP), `psm.ag` (redemption reserve + fee switch), `oracle.ag` (five-operator signed-print feed with
rate-limited prices and freeze/settle), `budget.ag` (daily net issuance budget), `surplus.ag` (treasury). The
vendored KCC-20 reference and conformance fixture are under `contracts/v2/vendor/kcc20/` (ISC, provenance
documented).

## History

Built in phases during 2026: the KCC-20 port proving `value ≠ amount`, the redemption PSM and emission-gate
hardening, the first live TN10 deployment, and the Phase 4 completion — KCC-20 interface conformance, fee
switch, penalty split, vault close, four adversarial security reviews with attack-transaction regression
tests, and a full re-deployment under real consensus. Phase 5 added the safety rails, proven live through a
frozen feed and a 75 % price crash absorbed by the rate limit before an atomic liquidation. Oracle Stage 1
replaced the 2-of-3 committee with five independent operators signing prices offline, live with relayed
refreshes and a key rotation. Every entry has run on TN10 except the 30-day settlement, which is covered by
tests. CI runs the full suite on every change.

## License

MIT — see [LICENSE](LICENSE). The vendored KCC-20 reference under `contracts/v2/vendor/kcc20/` remains under
its own ISC license.

## Safety

This repository exists so claims can be verified, not so the system can be used. Nothing here is audited;
TN10 covenants and keys are worthless by design; do not deploy this with real value. Today all five oracle
operator keys are held by the deployer; independent operators are a pre-mainnet gate. The full list of
pre-mainnet gates is in the [mainnet roadmap](docs/MAINNET-ROADMAP.md).

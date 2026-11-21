# kUSD: what is left before mainnet, and the roadmap to get there

**Status (2026-11-21): TN10 testnet only. Not audited. Not for real value.** This page is the single list of
everything that stands between the live TN10 deployment and a mainnet launch, and the order in which we intend
to close it. It is updated as items close; the live state is always at
[www.kaspa-402.org/kusd](https://www.kaspa-402.org/kusd).

## Where it stands

The protocol is complete and exercised end to end on TN10: vaults, the token authority with its emission gate,
the redemption reserve (PSM), the five-operator oracle, the daily issuance budget and the surplus treasury, all
in Kaspa L1 covenants. Since this repository's 2026-11-14 snapshot the live deployment has added the following;
each was re-deployed on TN10 and checked against the node's own covenant engine:

| Added | What it closes |
|---|---|
| Stability fee, 2 %/year on vault debt (accrued from the oracle's rate index) | vault debt is no longer free; the fee goes to the treasury |
| Separate borrow and liquidation lines: borrow at **200 %**, liquidate below **170 %** | a vault at its maximum is not liquidated on the next dip; both lines were set from a replay of the 10 Oct 2025 crash with measured order-book depth and liquidation impact |
| 5 KAS minimum collateral | every liquidation can pay the keeper in full |
| PSM swap-in spread rising from 0.50 % to 2.50 % as the daily budget empties (bound 3 %) | one party can't drain the day's issuance at the flat fee |
| Miner-median price bound (oracle Stage 2, proven by a zero-knowledge proof of coinbase votes) | the operators' price is clamped to ±5 % of the miner median when one is proven; fail-open when none is |
| Vault-debt ceiling ($330,000 total vault debt on TN10) | caps how much debt can ever need liquidating at once |
| Hardened PSM: share cap (a floor of $1,000 plus 25 % of vault debt), pro-rata redemption when the reserve is short, permissionless reserve top-up | the PSM can't become the dominant issuer, and a short reserve shares losses evenly instead of first-come-first-served |
| Vault factory: every vault is opened by a factory with zero debt, and the authority accepts vaults only from that factory's covenant group | closes the last known way to count vault debt that was never minted |

The harness for the current contracts runs 465 tests, including every attack found in the internal adversarial
reviews as a regression test.

## What is left

### Hard blockers: no mainnet until each is done

| # | Item | Why it blocks | Done when |
|---|---|---|---|
| B1 | **Independent third-party audit** of the contracts, the oracle proof path and the client that builds transactions | every review so far was internal; a review is not an audit | a reputable firm's report is published, every finding is fixed with a regression test, and the fixed code is re-deployed and re-checked on TN10 |
| B2 | **Written invariant argument**: a token box's KAS always matches its kUSD amount, and kUSD is created only through the vault and PSM paths under the budget | today only tests show it; auditors need the argument they are checking against | a document maps each covenant path to the invariants it preserves, with the tests that pin each one |
| B3 | **Publish the code that will ship** in this repository, plus a public security register | this repository is the 2026-11-14 snapshot; anyone verifying mainnet must be able to rebuild the exact audited contracts | the audited contracts, harness, deploy records and a current security register (known issues, accepted risks, review history) are here, with reproducible template hashes |
| B4 | **Five independent oracle operators** | on TN10 one deployer runs all five operator keys; mainnet needs three of five independent signers every 5 minutes | five named operators, each with its own machine, key and price source, run the signing service for at least 14 days on TN10 without a missed refresh window |
| B5 | **Mainnet governance keys** | the TN10 2-of-3 keys must never carry over; governance sets the PSM, budget and fee parameters and wires the factory once | new 2-of-3 keys on hardware signers, holders named publicly, a written key-loss and rotation procedure |
| B6 | **Genesis sizing and funding** | every covenant is created fresh on mainnet; caps set on TN10 values would be wrong | PSM ceiling, daily budget and vault-debt ceiling sized from a fresh measurement of KAS order-book depth at launch; deploy and operator fees funded (about 29 KAS/day at a 5-minute refresh) |

### Should close before launch

| # | Item | Done when |
|---|---|---|
| S1 | **Full settlement after a permanent oracle failure.** Today a feed silent for 30 days settles; the PSM still redeems at the last price (pro-rata if short), but vault collateral has no redemption path for holders | a settlement path lets holders redeem pro-rata against vault collateral, or the gap is accepted in the security register with its limits |
| S2 | **Reliable proving hardware** for the miner-median proof | a prover produces every window for 14 days with host verification on, without thermal faults |
| S3 | **Open-source keeper bot** (liquidations already pay the keeper 110 % of the debt burned) | a reference bot anyone can run, tested against a live liquidation on TN10 |
| S4 | **Monitoring and incident runbook** | alerts on a missed refresh, a frozen feed, budget exhaustion and PSM share-cap pressure; written responses for each, rehearsed on TN10 |
| S5 | **Mainnet fee and mass recalibration** | every transaction kind is re-metered against mainnet compute-mass and storage-mass rules |
| S6 | **Miner-vote bootstrap (Stage 2 on mainnet)** | the vote format is published to pools; a payout to voting miners is funded; the bound is switched on by governance once mainnet participation holds at least half of blocks for 7 days |

### Decisions still open

- **Legal structuring**: jurisdiction and a governance entity for a USD stablecoin. Unsolved; funded work.
- **Launch caps**: how small the first mainnet ceilings are, and the rule for raising them.
- **Operator set**: who the five operators are, and whether operators post a bond.
- **Stage 2 switch-on threshold**: the proposal is at least half of blocks voting for 7 days.

## Roadmap

Each phase starts only when the previous one's exit criteria are met.

### Phase 0 — Freeze and prepare for audit
- Freeze the contract set (no feature changes after this point except audit fixes).
- Write the invariant argument (B2) and the public security register; publish the current code here (B3).
- Write the audit scope: the six contracts and the factory, the oracle proof path, the transaction-building
  client, and the parameter rationale (stress replay, depth measurement).
- **Exit:** scope document sent to auditors; this repository matches the frozen code.

### Phase 1 — Audit
- Engage the auditor (B1); fix every finding with a test that failed first.
- Re-deploy the fixed contracts on TN10 and re-run every live flow.
- Close S1 (settlement) here if it is in scope, so it is audited with everything else.
- **Exit:** report published; all findings fixed or accepted in the register; TN10 runs the audited code.

### Phase 2 — Mainnet dress rehearsal on TN10
- Real operator set on their own machines (B4); mainnet governance keys on hardware (B5) used against TN10.
- 5-minute refresh for 14 days; prover (S2), keeper (S3) and monitoring (S4) running throughout.
- Drills: one and three operators offline, a stale-feed freeze and revival, a crash replay with a liquidation,
  budget exhaustion, a short PSM paying pro-rata, a governance key rotation.
- Re-meter everything for mainnet (S5).
- **Exit:** 14 clean days and every drill passed with the outcome written up.

### Phase 3 — Capped mainnet launch
- Measure depth, size the caps (B6), fund the deploy and operators.
- Create the covenants on mainnet; publish every genesis transaction and its verification against the source,
  as for each TN10 deployment.
- Switch the site, app, k402 and the market to mainnet.
- **Exit:** live with small caps and every genesis independently verified.

### Phase 4 — Grow
- Raise ceilings step by step, each step backed by a fresh depth measurement and a period of clean operation.
- Bootstrap miner votes (S6); switch the Stage 2 bound on when participation holds.
- **Exit:** caps at their target sizes; the miner bound on.

### Phase 5 — Beyond
- Oracle Stage 3: the miner median becomes the primary price, operators the fallback; or adopt a shared
  ecosystem miner oracle by changing the covenant id the vaults pin.
- Legal entity and longer-term governance.

## How to follow progress

- Live state and every covenant id: [www.kaspa-402.org/kusd](https://www.kaspa-402.org/kusd).
- Deployment records: [`contracts/v2/deploy/README.md`](../contracts/v2/deploy/README.md) (updated with each
  snapshot).
- This page: items move from the tables above to "Where it stands" as they close.

# kUSD — a covenant-native USD stablecoin on Kaspa L1

**Technical whitepaper & funding prospectus — DRAFT (Phase 4)**
Status: TN10 testnet research. **Not audited. Not a product. Not for real value.**
Every claim in this document is traceable to code, tests, or an on-chain
transaction; sources are cited inline. Repo: https://github.com/Kali123411/kUSD.

---

## 1. Thesis

Every stablecoin on every UTXO L1 to date has faced the same wall: the chain's
native value unit is the coin itself, so a "dollar token" ends up either bridged
(trust the bridge), on an L2 (trust the operator), or wrapped around a trusted
issuer (trust the signer). kUSD removes the wall instead of climbing it: **the
dollar amount lives in covenant state, decoupled from the box's KAS value**
(`value ≠ amount`), and every rule that moves those dollars — minting against
collateral, redemption at the oracle price, liquidation, fees — is enforced by
Kaspa consensus itself. No bridge, no L2, no multisig issuer, no trusted
executor anywhere in the protocol.

Three things make this possible now, and only on Kaspa:

1. **Toccata covenants** (mainnet since 2026-06-30): genesis-derived covenant
   identity, continuation bindings, and cross-covenant introspection — enough
   expressive power for a full CDP system on L1.
2. **O(1) cross-covenant validation** (SilverScript 1.0): a covenant validates a
   counterpart's complete state transition by sourcing the counterpart's
   template from a co-spent input — constant script size regardless of the
   counterpart's complexity. This single mechanism is why the whole protocol
   fits in ~5–10% of a block's compute budget (§6).
3. **10 blocks per second, sub-second confirmation**: liquidations and
   redemptions — the operations that defend a peg — confirm at interactive
   speed on L1, not on a rollup's exit schedule.

## 2. Why Kaspa

- **Throughput where it matters.** Peg defense is latency-sensitive: an
  arbitrageur who can redeem at $1.00 within a second of a $0.98 print is the
  peg. Kaspa's blockDAG gives L1 finality at speeds other L1 stablecoins only
  get from centralized sequencers.
- **UTXO parallelism.** Each vault is its own covenant lineage: collateral
  adjustments and everything else that moves no supply run in parallel. The
  serialized objects are the supply authority, the issuance budget, the
  redemption reserve and the oracle — every supply change passes through the
  authority and the budget, by design, because those ARE the monetary
  invariants.
- **Auditability without an indexer.** Covenant identity is genesis-derived;
  anyone can walk the kUSD lineage from the genesis outpoint and audit total
  supply against the UTXO set (we do exactly this in §5's conservation audit).
- **No bridged collateral anywhere.** The reserve is KAS; the debt unit is
  USD cents in state. The peg references an oracle price, not a
  wrapped dollar.

## 3. Architecture

Six covenants, one frozen configuration:

| Covenant | Role | Singleton? |
|---|---|---|
| **kUSD token** (`kusd.ag`) | KCC-20 fungible token; USD cents in state; supply authority box (marker `0x10`) holds un-issued supply | authority: yes; holders: no |
| **Vault** (`vault.ag`) | CDP: mint kUSD against KAS collateral at ≥175% ratio; repay, adjust, liquidate (110%/3% keeper/surplus split), close | no — permissionless, one per borrower |
| **PSM** (`psm.ag`) | USD-priced redemption reserve: the peg floor. `swap_in` mints at $1 of KAS/cent (governance-capped); `swap_out` redeems at $1 (permissionless, NEVER capped) | yes |
| **Oracle** (`oracle.ag`) | Price feed from five independent operators: each refresh takes the median of three separately signed prints (1e-4 USD/KAS precision), publishing two rate-limited prices — a low-biased *mint price* and a high-biased *liquidation price* — with permissionless stale-feed freeze and settlement | yes |
| **Surplus** (`surplus.ag`) | Protocol revenue in KAS; permissionless `receive`, 2-of-3 governance `spend` | yes |
| **Budget** (`budget.ag`) | Global issuance velocity cap: a token bucket every mint and PSM swap-in must pay from (refills fully in 24h, governance-set) | yes |

**The emission gate.** kUSD supply can only change in a transaction that
co-spends a *genuine, correctly-wired* emitter — a vault (by template: vaults
are permissionless) or THE PSM instance (by covenant id: its risk parameters
are governance-set). The authority's state carries
`blake3(vault_template ‖ psm_template ‖ oracle_covid ‖ psm_covid ‖ surplus_covid ‖ budget_covid)`, frozen at
genesis; the gate re-derives it from witness data on every emission. A forged
oracle, a hostile parallel PSM, or a mis-wired vault simply cannot gate an
emission — these are regression-tested as attack transactions, not argued in
prose (§5).

**Covenant identity, used correctly.** Counterparties are referenced by
*template hash* (stable, circularity-free) where permissionless, and by
*covenant id* (instance-pinned) where governance parameters matter. The deploy
order — oracle → unwired PSM → surplus → unwired budget → authority → wire → vaults — exists
precisely so every pin can be set without circular dependencies.

**KCC-20 conformance.** kUSD's state layout, transfer ABI (witness-blob
dispatch), all five KCC-2 owner schemes and all four borrow schemes conform to
the KCC-20 Final convention, verified vector-by-vector against the official
conformance fixture — including byte-exact state encoding and the exact
dispatch tag, so KCC-20 wallets and indexers recognize and construct kUSD
transfers with no custom integration. Declared deviations (all four visible
in the conformance table): the authority marker 0x10 (out of KCC-2's range, so
standard wallets correctly treat the supply box as nonstandard), a per-output
100M-sompi value floor (Kaspa's KIP-9 storage-mass reality), and the
supply-changing entries (`emission`, `rotate`) which sit outside the transfer
interface by design, and the rejection of owner scheme 0x04 on outputs (D4, §3's
conformance results).
**Conformance results** (`harness/tests/kcc20_conformance.rs`, driven by the official
reference fixture): state record layout & byte-exact encoding ✓; dispatch tags identical
to the reference (`79c71c23`/`fd3ef14a` — structural signatures) ✓; transfer argument
encoding byte-exact in the live sigScript ✓; all 6 standard-transfer cases ✓; all 5
owner-witness schemes live-executed (including a real P2SH co-input and ECDSA) ✓; all 16
borrowed-receive cases ✓; hash-chain vectors ✓. Declared deviations: **D1** per-output
value ≥ 100M sompi (KIP-9); **D2** authority marker 0x10; **D3** `emission`/`rotate`
outside the transfer interface; **D4** owner scheme 0x04 (covenant-id,
presence-as-consent) rejected on kUSD outputs — unsafe where every protocol covenant has
permissionless entries (our security review's finding I3, which we judged a hazard of
the scheme itself; wallets interoperate on schemes 0x00–0x03).

## 4. Security model

- **Adversarial reviews with attack regression tests.** Each development phase
  ended with a fresh-model adversarial review; every Critical/Important finding
  became an attack-shaped transaction in the test suite that failed before the
  fix and passes after (examples: the hostile-parallel-PSM ceiling bypass
  (Phase 2 C-1), the self-attested oracle wiring (Phase 1 C2), a
  delegate-custody hole closed by removing the check-free `join` delegate in
  Phase 4). The Phase 4 review alone produced two Critical findings — a burn double-credit across
co-spent emitter entries and an attacker-authored surplus pointer that would have made
vaults unliquidatable — both now closed by consensus-enforced pins and preserved as
attack regression tests (`harness/tests/phase4_review.rs`).
- **Oracle: no single operator moves the price.** Five independent operators
  each sign their own price print offline, stamped with the DAA score they
  signed at. Anyone can relay three of them into a refresh; the covenant takes
  the **median**, and refuses the set if the prints disagree by more than 3%. One
  rogue operator cannot move the price at all; two colluders can shift it at
  most within that band of an honest print; free movement needs three of five,
  and the rate limits below bound even that. The feed's time is the median
  signed-at score, so a relayer can neither pick a convenient clock nor revive a
  frozen feed with prints held back from before it went silent. Three operators
  (never the one being replaced) can rotate a key, so even two compromised
  operators can always be removed.
- **Oracle safety rails.** The operators post a raw price, but nothing prices
  off it directly. Minting and withdrawals use a *mint price* that follows the
  market down instantly and up at most 10% per hourly refresh; liquidations and
  redemptions use a *liquidation price* that follows it up instantly and down at
  most 25% per refresh — however long the feed was silent. A manipulated or flash print can neither inflate borrowing
  power nor trigger a liquidation cascade before the market confirms it. Time is
  proven inside the covenant from the confirmation DAA score of a co-spent
  input — a consensus fact that can only understate the clock, so no one can
  game the limiter by lying about time.
- **Stale-feed shutdown.** If the operators go silent for an hour, anyone may
  freeze the feed: minting, withdrawals, liquidations and PSM swap-in stop,
  while repayment, vault close, deposits and PSM **redemption keep working** —
  the peg floor stays open. Thirty days of silence lets anyone settle the
  system permanently.
- **Issuance velocity cap.** Net new supply — vault mints and PSM swap-ins
  minus repayments, liquidations and redemptions — draws from one global token
  bucket (default $50k/day, governance-set, refills fully in 24h). Metering is
  bound to the supply authority's actual change, so neither a holder transfer
  nor a mint-and-repay round trip can drain it into a minting freeze (both were
  attack transactions in our Phase 5 review, now regression tests). The total supply ceiling remains the authority's genesis supply,
  which governance cannot raise.
- **Frozen-family invariant.** Every vault/PSM entry that doesn't change supply
  co-spends the kUSD authority FROZEN — so the emission gate can never hide
  under an unrelated entry. This includes the terminal `close`.
- **Supply conservation, demonstrated on-chain.** After the full Phase 3 cycle
  (mint, transfer, swaps, repay, liquidation), the authority's on-chain amount
  returned to exactly the genesis supply: zero unaccounted cents across 14 live
  transactions. (§5; and repeated on the hardened Phase 4 family: supply − outstanding matched to the cent).
- **Bounded arithmetic.** Every product stays under 2^62 by parameter bounds
  (collateral ≤ 40k KAS, debt ≤ $400M, price ≤ $10k/KAS at 1e-4 USD precision,
  spread ≤ 200 bps, daily budget ≤ $400M).
- **Known limits, stated plainly.** Redemption is paid from the reserve's own
  KAS, which was deposited at the price of each swap-in. If KAS falls far enough
  that the reserve is worth less than the kUSD it issued, redemptions are paid
  first-come until the reserve reaches its floor, and later ones fail (the reserve
  never goes negative, and governance still cannot block it). Pro-rata redemption is
  a design option on the §9 list. During a genuine crash the redemption floor
  pays at the lagging liquidation price, i.e. somewhat less than $1 of KAS per
  kUSD until the price catches up (about $0.67 one hour into a 50% crash) — the
  deliberate price of manipulation resistance. "Settlement" today is an
  emergency halt that keeps redemption, repayment and close open; a full
  settlement mechanism (pro-rata claims on vault collateral after a permanent
  oracle failure) is designed-not-built and listed in §9.
- **What is NOT claimed:** no external audit yet (§9 — it is the first thing
  funding buys), a five-operator oracle is still a trust assumption (three colluding operators can move the price within the rate limits; §9), and
  TN10 evidence is testnet evidence.

## 5. Live evidence (TN10)

The full tables live in `contracts/v2/deploy/README.md` ("Live state"); every
transaction is archived under `docs/pitch/evidence/`. Four deployment campaigns
so far. The latest (oracle Stage 1, 2026-11-14) re-created every covenant around
the five-operator oracle (all five genesis covenant ids reproduced by `argentc
genesis verify --require-argent`) and ran it live:

- **No operator signed the transaction.** Operators 0, 2 and 4 each signed a
  print offline (41.00¢, 41.20¢, 40.90¢); a relayer submitted them and the
  covenant took the median, 41.00¢ (`a20b5c40…`). A print set spread more than 3%
  apart was refused by the covenant.
- **Operator rotation:** slot 4's key was replaced by three other operators
  (`9629c0e1…`); the next refresh included the new key (`2fdae10b…`). Minutes
  after the previous refresh, the rate limit let the mint price rise only 41.00¢
  → 41.08¢, and a mint priced at exactly that (`54cfc73b…`).
- **Governance cut the daily issuance limit** from $50,000 to $40,000
  (`ad01eaf9…`), clamping the available budget in the same step.

The Phase 5 campaign (same day) put the safety rails through real events:

- **A stale feed shut minting down:** after an hour of oracle silence anyone could
  freeze the feed (`82f8ffef…`), and a mint against it was refused. The next
  refresh revived it, with borrowing power allowed to rise only 10% in one step
  (`9ce3b8cf…`).
- **A crash the rate limit absorbed:** the raw price fell 75% (80¢ → 20¢) and stayed
  there. Borrowing power dropped at once, but the liquidation price stepped down
  25% per hourly refresh (80 → 60 → 45 → 33.75 → 25.32¢). The vault stayed safe for
  three hours, then was liquidated at the rate-limited price (`0486065a…`): keeper
  868,870,000 sompi, surplus +23,690,000.
- **Issuance metered and refunded:** every mint and reserve swap drew on the daily
  budget, and every burn credited it back. At the end the budget was full again, and
  the authority's on-chain amount again equals the genesis supply minus exactly the
  450 cents outstanding.

The Phase 4 campaign (2026-11-11) highlights:

- **Genesis, independently verified:** every covenant's on-chain covenant id is
  reproduced by `argentc genesis verify --require-argent` from the published
  state (oracle `347c14f8…`, PSM `93ee96b6…`, surplus `00b9bcbf…`, authority
  `a2af8c3b…`).
- **The fee demo:** governance set the spread to 25 bps (`91cc22ca…`), a mint-side
  swap routed **2,500,000 sompi of protocol revenue into the surplus covenant**
  (`727025a8…`), a redemption-side swap routed another 2.5M (`466cae76…`), and
  governance switched fees back OFF (`6421cf41…`). Fees are off today; the
  mechanism is proven.
- **Borrowed receive:** 50 cents paid INTO a third party's kUSD box with no
  signature from its owner, under its declared KCC-20 borrow policy (`32869de3…`).
- **The full CDP lifecycle:** mint at 40¢/KAS (`f30b0a25…`), collateral adjust,
  a second vault minted at the moved 45¢ price, full repay releasing 11.9 KAS
  (`4135dcc1…`), terminal `close` ending the lineage (`a67542c0…`).
- **Split liquidation:** after a committee price crash to 30¢, the keeper seized
  733,333,333 sompi and the **surplus covenant took 20,000,000** — the 110/3
  split, live (`a3670d1c…`).
- **Governance revenue withdrawal:** 20M sompi spent from the surplus by 2-of-3
  with a consensus-pinned destination (`6edb962b…`).
- **The audit:** at the end, the authority's on-chain amount is 99,999,550 cents
  — the genesis supply minus EXACTLY the 450 cents still outstanding; the surplus
  balance is 105,000,000 sompi, every sompi of it accounted for (genesis + two
  2.5M fees + the 20M split − the 20M withdrawal). Nothing minted unaccounted,
  nothing lost. Across four deployment campaigns and more than 100 live
  transactions, the supply has reconciled to the cent every time.

## 6. Performance & cost

Full table: `contracts/v2/README.md`. The worst path (fee-routed redemption,
6 covenant inputs including the treasury and the issuance budget) uses **50,466
compute mass — 10.1% of the 500k cap** — and 15.5% of the transient cap; the atomic
split liquidation is 46,092 (9.2%). A refresh relaying three signed operator prints
costs 6,665 (1.3%). For contrast, v1's non-split liquidation was 498k
(99.6% of the cap): O(1) cross-covenant validation bought an ~11x improvement while
ADDING the penalty split, the fee switch, the rate-limited oracle and the net
issuance budget. There is roughly 10x headroom left for future mechanism work on L1.

## 7. Economics

**The fee switch exists, is OFF, and has been demonstrated live.** Protocol
revenue mechanisms are built into the covenants but ship at zero:

- **PSM spread** (`spread_bps`, 0–200, governance 2-of-3): charged on the KAS
  leg of swaps — on top for swap_in, out of the payout for swap_out, so reserve
  backing per outstanding cent is never diluted. Routed trustlessly to the
  surplus covenant (covid-pinned; a hostile surplus box cannot capture fees).
- **Liquidation split**: of the 13% penalty, the keeper takes 10 points, the
  surplus takes 3 — keepers stay profitable, the protocol accrues reserves.
  (Constants in this slice; governance parameters for production.)
- **Roadmap: stability fees** on CDP debt (DAA-based accrual) — honest status:
  designed, not built (§9).

Turning fees on is a governance act we commit to taking only after an external
audit — "we could extract, and haven't" is verifiable on-chain rather than a
promise. The demo cycle in §5 is the receipt: spread on (`91cc22ca…`), fees routed
(`727025a8…`, `466cae76…`), spread off (`6421cf41…`), governance withdrawal
(`6edb962b…`).

**Illustrative revenue scenarios** (clearly labeled illustration, not a
projection: PSM volume is the one number nobody can know pre-launch):

| PSM annual volume | 10 bps | 25 bps | 50 bps |
|---|---|---|---|
| $1M | $1k/yr | $2.5k/yr | $5k/yr |
| $10M | $10k/yr | $25k/yr | $50k/yr |
| $100M | $100k/yr | $250k/yr | $500k/yr |

(Liquidation revenue scales with CDP risk appetite and is deliberately not
projected. Stability fees, once built, are the dominant revenue line in
comparable CDP systems.)

## 8. Competitive landscape

Two other stablecoins are announced for Kaspa. They make different trade-offs,
and an honest comparison is the best argument for kUSD's specific one.

| | **kUSD** | **Kaspa Dollar (KSD)** | **Trestly kUSDT** |
|---|---|---|---|
| Backing | KAS collateral, ≥175% | KAS collateral, 250% open / 200% liquidation | USDT locked 1:1 on Ethereum / BNB Chain |
| Trust model | L1 covenants; median of 5 independent operators' signed prints | L1 covenants; miner-voted price, ZK-proven | L1 covenants gated by a 3-of-5 signer federation |
| Peg floor | Redemption reserve (PSM): $1 of KAS per kUSD, permissionless, governance can't block it; paid from reserve KAS while it lasts | Liquity-style redemption against any vault ≥100% backed | Burn → federation releases USDT |
| Liquidation | Atomic; keeper 110% of debt, treasury 3% | Anyone; 5% penalty; partial to 200% | n/a |
| Rate limits / breakers | Rate-limited mint & liquidation prices; daily issuance budget; stale freeze + settlement | Rate-limited mint & liquidation prices; daily net-mint budget; freeze + settlement | Daily mint allowance; pool sell breaker; guardian pause |
| Revenue | Fee switch (built, off) + 3% liquidation share | 0.1% oracle fee to miners/provers/keepers | Bridge fees 0.5–2.5%; pool fee |
| Status (Nov 2026) | **Every entry but one (the 30-day settle, test-covered) executed live on testnet; supply audited to the cent** | Whitepaper; testnet planned mid-2027 | Public testnet with wallet extension |
| Audit | Not yet | Not yet | Not yet |

*Sources: KSD whitepaper (kaspadollar.com/whitepaper, a working draft); Trestly
documentation (testnet.trestly.io/docs). Figures are as those projects state them.*

**Where kUSD leads.** It is the only one of the three that exists as a working,
fully exercised protocol — every mechanism except the 30-day settlement has run
under real consensus, and the
supply reconciles on-chain to the cent. It is the only one with a dedicated
redemption reserve, a demonstrated on-chain revenue mechanism, and interface-level
KCC-20 conformance proven against the official fixture.

**Where kUSD trails.** KSD's miner-voted, ZK-proven oracle removes trusted price
operators altogether; kUSD's five-operator median is its largest remaining trust
assumption (§9 sets out the path off it). Trestly's dollar is backed 1:1 by a fiat stable —
simpler for users to reason about, at the cost of a federation that can mint or
release if three of five keys collude.

**Complementary, not zero-sum.** A KAS-collateralized dollar and a bridged one
serve different users. And kUSD's vaults pin their price feed by covenant id, so
if KSD's miner-voted oracle ships as a readable price output, kUSD can adopt it
by re-pointing that one pin — the two designs can strengthen each other.

## 9. Road to production & the ask

What exists is a complete, live-exercised protocol. What stands between it and
mainnet is exactly what funding buys, in order. The detailed list, with a "done when" for each item and the
phased plan, is [`docs/MAINNET-ROADMAP.md`](../MAINNET-ROADMAP.md).

| Stage | What it closes | Status |
|---|---|---|
| **External audit** (2 firms, covenant + economic) | the single biggest gap: nobody but us and our adversarial-review process has attacked this code | `[OWNER: ask amount]` |
| **Oracle decentralization** | Stage 1 (five-operator signed-print median) and Stage 2 (operators bounded to ±5 % of a proven miner median) **live on TN10**; Stage 3 makes the miner price primary (see below) | Stage 2 needs mainnet miner participation |
| **Mainnet beta under caps** | tiny governance ceiling, fee switch off, real KAS, capped exposure | after audit |
| **Keeper ecosystem** | open-source keeper bots (the liquidation split already pays them 10%) | reference bot exists as deploy tooling |
| **Stability fees** | DAA-based accrual covenant | **live on TN10** (2 %/year) |
| **Full settlement** | pro-rata redemption against vault collateral after a permanent oracle failure (today: an emergency halt) | designed, not built |
| **Liquidity bootstrap** | PSM reserve seeding, market-maker partnerships | `[OWNER: ask amount]` |
| **Legal structuring** | jurisdiction, governance entity | explicitly unsolved; funded work |

### The oracle roadmap

Trusted operators are a bootstrap, not the destination. The roadmap runs in
stages, each building on the same covenant-id pin the vaults already use:

- **Stage 1 — independent signed prints (done, live on TN10 2026-11-14).** Five
  operators sign prints offline; refreshes take the median of three with a 3%
  spread check; prints carry their signing time; three operators rotate a key.
  Live evidence: refreshes relayed from separately signed prints (`a20b5c40…`,
  `2fdae10b…`), an out-of-band print set refused, and a key rotation
  (`9629c0e1…`) after which the new key printed and the old one cannot.
- **Stage 2 — the miner median bounds the operators.** Kaspa miners already
  publish data in every block's coinbase, and the covenant toolchain can verify
  chain-derived data with zero-knowledge proofs. A miner-voted median, proven to
  the covenant, bounds the operators' print: neither party alone can move it.
- **Stage 3 — the miner median is primary.** Operators become the fallback.

Three ways to get to Stages 2–3, not mutually exclusive:

1. **Consume a shared miner oracle.** If the Kaspa ecosystem converges on one
   miner-voted price output (KSD's design is one), kUSD adopts it by changing
   the covenant id its vaults pin. One oracle serving several protocols is the
   best outcome for everyone: miners integrate once, and its security grows
   with the value depending on it.
2. **Hybrid bound.** Keep the operators, but accept their median only inside a
   band around the miner median (Stage 2).
3. **Operators with stake.** Bond the operator keys, as a fallback while miner
   adoption grows.

The rate-limited prices and stale-feed shutdown already in place apply to any
of these sources, and they bound what any single compromised feed can do.

**Funding ask:** `[OWNER: amounts mapped to the stages above]`
**Team:** `[OWNER: bio]`
**Contact:** `[OWNER: contact handles]`

### Community funding

The Kaspa community can fund this work directly:
**`kaspa:qre88nev5r8y3knz4jja6g08tpstqaukumump6q2xh0da5zy3j5sulvmlwy5c`**
(held by the project founder in a Kaspa wallet; spending is reported against
the roadmap stages above; this address funds development, it is not an
investment contract and confers no token or claim).

## 9b. The agent economy: money, rails, market

kUSD's first native demand is not people paying people; it is **agents paying agents**. Autonomous
software can't open a bank account, pass KYC, or hold a card. kUSD gives an agent stable-denominated
money whose rules are covenant-enforced and whose state it can verify against a node; k402 (Kaspa x402
v2) gives it a way to pay per call with no account or API key; the market gives it discovery,
settlement-proven reputation, and operator-set spend limits. Humans supervise; agents transact.

The product is three layers, and this document is the first of them:

| Layer | What | Where |
| --- | --- | --- |
| Money | kUSD — this system | www.kaspa-402.org/kusd |
| Rails | k402 — HTTP 402 pay-per-call on Kaspa L1; kUSD is an x402 `exact` asset (binding `kusd-holder-v1`); KCC-0402 covenant payment channels are the streaming roadmap | kaspa-402.org · github.com/Kali123411/k402 |
| Market | the Kaspa Agent Market (KAM) — services priced in kUSD, a settlements feed the registry verifies on its own node, reputation that is settled kUSD rather than ratings, and `SpendPolicy`: the bounds a human operator sets on an autonomous buyer (per-call ceiling, budgets, call caps, a stop after unserved payments, allowed capabilities, a kill switch) | www.kaspa-402.org/market |

**The market is the live proof.** On testnet-10, a provider lists `summarize` at 0.05 kUSD a call and
answers unpaid requests with an x402 402; an agent — the k402 Python client, the `k402-mcp` tool
`pay_kusd`, or a browser wallet on the market page — pays with a covenant box of exactly 5 cents to the
offer's own derived key, retries with the proof, and is served; the provider verifies the box on its node,
claims it, and reports it to the registry, which verifies it again before it counts toward reputation.
Each step is a transaction a node can check, and the operator's policy refuses before any box is built when
a bound would be crossed. The safe claim, and the only one made: *an agent-native marketplace on Kaspa L1,
settled in a covenant-native stablecoin built for agents* — the first agent-native marketplace on Kaspa.

Why the design serves agents, mechanism by mechanism, is in the repository README; the integration guide
for agent developers is `docs/agents/README.md`.

## 10. Verify everything

- The contracts, tests, and this document: `[MIRROR-URL]`
- Every cited txid: `docs/pitch/evidence/` (serialized transactions,
  re-verifiable locally against the node-equivalent covenant engine — TN10
  resets cannot erase the evidence).
- The conformance suite: `cargo test --release --test kcc20_conformance`
- The full suite: `cargo test --release` (116 tests, including every review attack as a regression)
- The metering table regenerates with `cargo test --release -- --nocapture`

---

*Safety: kUSD is TN10 research software. It is not audited, not deployed to
mainnet, not a product, and must not hold real value. This document is not
investment advice and offers no security or token.*

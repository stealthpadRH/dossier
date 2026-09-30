# Stealthpad: Creator Fees Held by Code

**A thesis on keyless fee vaults, time-weighted snapshots, and private landings for launchpad coins on Robinhood Chain**

Version 0.1 · September 2026 · Pre-launch

---

## Abstract

A launchpad coin produces one reliable cash flow: the creator tax paid on every trade. On most venues that flow ends in a single wallet. Stealthpad routes it into a vault contract deployed per coin, with no owner, no withdraw path, no upgrade, and no pause. Any wallet may convert the accumulated fees into USDG once a threshold and a cooldown are met, and the conversion pays the caller a fixed tip in the same transaction. Holder shares are computed from balances averaged over twelve blocks drawn at random from the window since the previous round, committed as a cumulative Merkle root, and delayed two hours before claims may use it. A holder claims once for every round to date and chooses the landing: USDG in the same wallet, Zcash at the holder's transparent address, or dollars on Aleo. This paper states the mechanism, its arithmetic, the exact limits of its privacy, the role of the $STEALTH token, the revenue it produces, and the risks it carries.

---

## 1. The Problem

### 1.1 One wallet collects

A standard launchpad pays the creator tax to the launcher for the life of the coin. The holders carry the price risk. One address collects the cash. The arrangement is legible and it is lopsided: the party with the least exposure after launch holds the only income the coin generates.

### 1.2 Fee sharing with a key in the middle

Fee-sharing launchpads changed the recipient without changing the custodian. A keeper wallet run by the platform holds the fees, often with every vault key derived from one server secret, and distributes them on its own schedule. Two failure modes follow directly. When the keeper stalls, fees accumulate and nobody can move them. When the keeper is compromised, every vault on the platform drains in the same hour. Holders are asked to trust a server they cannot inspect with money they cannot reach.

### 1.3 Snapshots that reward timing

A snapshot read at a single block pays whoever holds at that block. A trader who buys one block before and sells one block after collects a full share of a period he did not hold through. The honest holder subsidises the timer.

### 1.4 Payouts nobody claims

Distributions that arrive as dust in one asset, behind a claim that demands a new wallet type, go unclaimed. The payout exists on paper and the holder never sees it.

Stealthpad addresses each of these with a specific construction. The sections below describe them in order.

---

## 2. The Keyless Vault

### 2.1 Deployment

A launcher calls `StealthFactory.launch(input, taxBps, minTokensOut)` with a name, ticker, image, links, and a creator tax of 1, 2, 3, 4 or 5 percent (`taxBps` in {100, 200, 300, 400, 500}). In that single transaction the factory deploys the coin's `StealthVault`, launches the coin on the Pons V2 bonding curve with the vault as creator fee recipient, and optionally executes the launcher's first buy. The tax is fixed for the life of the coin.

### 2.2 Inflow

Two streams enter each vault, both in ETH:

1. the creator tax on every buy and every sell, and
2. the creator's share of the Pons V2 trading fee (70% of 1% under the current Pons policy, fixed per coin at launch), credited through the Pons fee escrow.

### 2.3 Powers and their absence

| The vault can | The vault cannot |
|---|---|
| Pull fees the Pons escrow credited to it | Be upgraded, paused or migrated by anyone |
| Swap ETH to USDG with a minimum-out guard | Change its split after deployment |
| Pay the round tip and two protocol addresses fixed at deployment | Change the coin's creator tax |
| Pay claims that verify against its active root | Send funds to any other destination |
| | Pay out more than the rounds it has recorded |

Every constant lives in bytecode with no setter:

| Constant | Value | Meaning |
|---|---|---|
| `HOLDERS_BPS` | 9200 | 92% of each round to holders |
| `CALLER_BPS` | 300 | 3% to the wallet that ran the round |
| `STEALTH_BPS` | 250 | 2.5% to the $STEALTH vault |
| `TREASURY_BPS` | 250 | 2.5% to the treasury |
| `MIN_ROUND` | 0.02 ETH | smallest round that may run |
| `COOLDOWN` | 3600 s | minimum spacing between rounds |
| `ROOT_DELAY` | 7200 s | wait between a root's posting and its use |
| `MAX_ROUTE_FEE_BPS` | 20 | ceiling on the route fee of a claim |
| `IMPACT_CAP_BPS` | 100 | one round swaps at most 1% of the pool's USDG |
| `SLIPPAGE_BPS` | 150 | minimum-out sits 150 bps below the 30-minute TWAP |

There is no launcher share. A launcher is paid only as a holder of the coin, under the rule that binds every other holder.

---

## 3. Rounds

A round converts accumulated ETH into a holder allocation. Any address may trigger it.

### 3.1 Preconditions

Let $V$ be the vault's ETH balance and $E$ the fees waiting in the Pons escrow. A round may run when

```math
V + E \ge 0.02\ \text{ETH} \quad \text{and} \quad t_{\text{now}} \ge t_{\text{last}} + 3600\ \text{s}.
```

### 3.2 Execution

`runRound()` pulls the escrowed fees, fixes the budget $B$, and applies the split:

```math
\text{tip} = \tfrac{300}{10^4}B, \qquad \text{stealth} = \tfrac{250}{10^4}B, \qquad \text{treasury} = \tfrac{250}{10^4}B, \qquad H = B - \text{tip} - \text{stealth} - \text{treasury}.
```

The holder portion $H$ is swapped to USDG on the WETH/USDG pool with a minimum output of

```math
\text{minOut} = H \cdot P_{\text{TWAP}}^{30\text{m}} \cdot \left(1 - \tfrac{150}{10^4}\right),
```

and the budget is capped so that one round never takes more than 1% of the pool's USDG:

```math
B \le \frac{0.01 \cdot U_{\text{pool}}}{P_{\text{TWAP}}^{30\text{m}}} \cdot \frac{10^4}{9200}.
```

Anything above the cap waits for the next round. The tip is paid to the caller in the same transaction. The round is recorded with its identifier, timestamp, ETH budget, holder USDG amount $U_r$, and the running total $\sum_{i \le r} U_i$, and a `RoundRun` event is emitted.

The $STEALTH vault forwards no share to itself. Its own 2.5% stays with its holders, so a $STEALTH round splits 94.5% holders, 3% caller, 2.5% treasury.

### 3.3 Why the tip is a share

A fixed tip would pay keepers to trigger empty rounds. A proportional tip pays nothing when there is nothing to split, and the 0.02 ETH minimum makes a hollow round impossible to call at all. Keepers are paid for work that moves money to holders, in proportion to the money moved.

---

## 4. The Time-Weighted Random Snapshot

### 4.1 Construction

Let round $r$ execute in block $b_r$ with block hash $h_r$, and let the previous round have executed in block $b_{r-1}$ (for the first round, the block before launch). The sampling window is

```math
W_r = (b_{r-1},\ b_r], \qquad N = |W_r| = b_r - b_{r-1}.
```

With $k = 12$, if $N \le k$ the whole window is sampled. Otherwise the attester draws candidates

```math
s_i = b_{r-1} + 1 + \big(\operatorname{keccak256}(h_r \,\|\, i) \bmod N\big), \qquad i = 0, 1, 2, \dots
```

and keeps the first twelve distinct values as the sample set $S_r$. The seed $h_r$ is the hash of the round's own block, so the draw is fixed by history that no participant could observe while trading inside the window.

### 4.2 Weight

For holder $a$, let $\beta_a(s)$ be the balance at the end of block $s$. The weight is the mean over the sample:

```math
\bar{\beta}_a = \frac{1}{|S_r|} \sum_{s \in S_r} \beta_a(s).
```

### 4.3 Eligibility and exclusion

Let $X$ be the excluded set: the bonding curve, the pool manager and its hook, the vault, the token contract, the factories, the zero address, and the burn address. Let $F = 100{,}000$ tokens, which is 0.01% of supply. The eligible set is

```math
\mathcal{E}_r = \{\, a \notin X \;:\; \bar{\beta}_a \ge F \,\}.
```

### 4.4 Allocation and carry

Let $C_{r-1}$ be the amount carried from the previous round. The distributable amount is $D_r = U_r + C_{r-1}$, and each eligible holder receives

```math
q_a = \left\lfloor D_r \cdot \frac{\bar{\beta}_a}{\sum_{j \in \mathcal{E}_r} \bar{\beta}_j} \right\rfloor,
\qquad
C_r = D_r - \sum_{a \in \mathcal{E}_r} q_a.
```

Shares forfeited by holders below the floor, and every rounding remainder, roll into the next round. Nothing is lost to the protocol.

### 4.5 Cumulative commitment

Each holder's leaf carries the total assigned to date, $L_a^{(r)} = L_a^{(r-1)} + q_a$. The attester builds a standard Merkle tree over leaves

```math
\ell_a = \operatorname{keccak256}\!\big(\operatorname{keccak256}(\operatorname{abi.encode}(a,\ L_a^{(r)}))\big),
```

publishes the full leaf file, and signs an EIP-712 message `PostRoot(roundId, root, total, seed)` under the domain `{name: "Stealthpad", version: "1", chainId: 4663, verifyingContract: vault}`. The vault accepts the root only if the signer is the factory's attester, the round exists and is newer than the active root, and the total $T$ sits between the active root's total and the cumulative holder amount the vault itself recorded through that round:

```math
T_{\text{active}} \;\le\; T \;\le\; \sum_{i \le r} U_i .
```

The bound is an inequality by design: carry and rounding remainders keep $T$ at or slightly below the recorded USDG, and a root can never assign more than the vault received.

### 4.6 What the snapshot defeats

**Block timing.** A position opened one block before $b_r$ and closed one block after appears in few or none of the twelve samples. Its weight approaches $\beta / 12$ at best and zero in the expected case. Holding through the window is the only strategy that scales.

**Sybil splitting.** Allocation is linear in $\bar{\beta}$. For any partition of a balance across wallets $a_1, \dots, a_m$,

```math
\sum_{i=1}^{m} \frac{\bar{\beta}_{a_i}}{\sum_j \bar{\beta}_j} = \frac{\bar{\beta}_{a}}{\sum_j \bar{\beta}_j},
```

so splitting gains nothing. The eligibility floor makes it strictly worse: fragments below $F$ forfeit their share to the carry.

### 4.7 Verifiability

The arithmetic above is pure. The open verifier replays every `Transfer` of the coin and every `RoundRun` of the vault from public chain data, recomputes each root, and compares it with the root the vault accepted. No Stealthpad server is involved in that check.

---

## 5. Roots, Delay, and Claims

### 5.1 The two-hour window

A posted root becomes claimable 7200 seconds after posting. During that window one address, the guardian named in the factory, may call `veto()`. A vetoed round is attested again. The guardian cannot move funds, change a split or a tax, pause a vault, stop a round, or post a root. It can propose a new attester, and that change takes effect only after `ATTESTER_DELAY` of two days, announced on chain by an `AttesterProposed` event that watchers alert on. A vetoed root is not retired by the veto alone: the operator pauses the attester, corrects the cause, and re-enables it, so the corrected root is the one posted next.

### 5.2 The claim

```solidity
claim(uint256 cumulative, bytes32[] proof, address to, uint16 feeBps)
```

The vault verifies the leaf for `msg.sender` against the active root, pays `cumulative - claimed[msg.sender]` in USDG to `to`, and sends `feeBps` of it to the treasury. The contract rejects any `feeBps` above 20. Because leaves are cumulative, one claim collects every round since the holder's previous claim, and nothing expires.

### 5.3 Invariant

For every vault, at every block:

```math
\text{USDG balance} \;\ge\; \text{total}_{\text{active}} - \text{totalClaimed}.
```

The root total is bounded by the USDG the vault actually received from its swaps, and each claim reduces the outstanding amount by exactly what it pays.

---

## 6. Payout Routes

Every route begins from USDG in the vault. Zcash and Aleo deliveries are performed by NEAR Intents solvers.

| Route | Lands at | Route cost (dry quotes, 2026-09-28) | Time | Stealthpad fee |
|---|---|---|---|---|
| USDG | the same wallet on Robinhood Chain | none | one transaction | none |
| Zcash (ZEC) | the holder's transparent address (`t1...`) | 2.5% at $25, 0.96% at $100, 0.49% at $1,000 | about 2 minutes | up to 0.20% |
| USDCx on Aleo | an `aleo1...` address | 6.4% at $5, 0.64% at $100 | under a minute | up to 0.20% |
| USAD on Aleo | an `aleo1...` address | similar to USDCx | under a minute | up to 0.20% |
| Monero | added the day NEAR Intents lists it | | | |

Zcash and Aleo claims carry a $5 minimum. USDG has none. Route cost has a fixed component, so larger and later claims pay proportionally less; the cumulative claim makes waiting costless.

### 6.1 The route sequence

1. The holder's own browser requests a quote from NEAR Intents for the destination address the holder typed. The address is validated first: base58check for a Zcash transparent address, bech32m for an Aleo address.
2. The quote returns a deposit address, an amount out, and a deadline.
3. The holder signs `claim()` with `to` set to that deposit address.
4. Solvers deliver ZEC to the transparent address or dollars to the Aleo address.
5. If the quote expires unfilled, NEAR Intents refunds the USDG to the holder's wallet on Robinhood Chain.

The destination address never passes through a Stealthpad server.

### 6.2 The exact scope of privacy

The route delivers Zcash to transparent addresses only. It rejects shielded and unified addresses. Zcash lands in the open and becomes private when the holder shields it in the wallet; Zashi and most Zcash wallets perform that in one step.

| Private | Public |
|---|---|
| Zcash after the holder shields it: its balance and every later move | The Robinhood Chain wallet that held the coin |
| Aleo dollars moved with a private transfer | Its share of each round, since leaves are published |
| | The claim transaction and the deposit address it paid |
| | The landing address until the payout is shielded or privately moved |

The route's own records link a deposit to its delivery. Stealthpad offers no batching, no fixed denominations, and no delays designed to obscure who claimed. It is a fee-sharing launchpad with a private landing, and it is not a mixer. The full statement is in [PRIVACY_SCOPE.md](./PRIVACY_SCOPE.md).

---

## 7. $STEALTH and Its Tiers

$STEALTH is the launchpad's own coin. It launches first on mainnet day, through the same StealthFactory, onto the same Pons V2 curve, under the same vault rule, with a 3% creator tax paid to its own StealthVault.

### 7.1 What holding it gives

1. **Exposure to the whole board.** 2.5% of every round on every Stealthpad coin flows into the $STEALTH vault and is shared among $STEALTH holders by the same time-weighted snapshot.
2. **Route fee relief** on Zcash and Aleo claims.
3. **Early access** to new payout routes.
4. **A signal vote** on the next route for the Core tier. The vote is advisory; no admin key exists for it to steer.

### 7.2 Tiers

Tiers are measured as a share of supply using the same time-weighted snapshot as rounds.

| Tier | Share of supply | Route fee | Extra |
|---|---|---|---|
| Visitor | any | 0.20% | |
| Holder | at least 0.05% | 0.10% | holder badge on the claim page |
| Keeper-plus | at least 0.2% | 0% | first access to new routes |
| Core | at least 1% | 0% | signal vote on the next route |

The contract enforces only the ceiling of 20 bps; the interface applies the tier.

### 7.3 What $STEALTH may never do

It cannot buy a larger share of any coin's rounds, alter the snapshot or its seed, pause, upgrade or drain a vault, mint new supply, or override a coin's creator tax. Badges are neither tradeable nor multipliers.

---

## 8. Revenue

### 8.1 Sources

The protocol earns from two places and charges no launch fee.

1. **Treasury share.** 2.5% of every round on every coin.
2. **Route fee.** Up to 0.20% on Zcash and Aleo claims, reduced or waived by $STEALTH tier. USDG claims carry none.

Launchers pay only the Pons V2 launch fee of 0.0005 ETH and the network fee.

### 8.2 Treasury allocation

| Share | Use |
|---|---|
| 40% | Operations: attester, indexer, read API, monitoring |
| 30% | Security: audits, bug bounty, fork rehearsals |
| 20% | Growth: receipts, launch campaigns, partner coins |
| 10% | Reserve |

Totals and the treasury address are published monthly.

### 8.3 Scenarios

Assumptions: ETH at about $2,690, an average creator tax of 3%, daily figures. The table counts creator tax only; the Pons creator fee share would add roughly 0.7% of volume on top, so the figures are conservative.

| Scenario | Coins x volume/day | Tax into vaults | To holders (92%) | To keepers (3%) | To $STEALTH vault (2.5%) | To treasury (2.5%) |
|---|---|---|---|---|---|---|
| Low | 10 x 2 ETH | 0.6 ETH ($1,614) | 0.552 ETH ($1,485) | 0.018 ETH ($48) | 0.015 ETH ($40) | 0.015 ETH ($40) |
| Mid | 30 x 5 ETH | 4.5 ETH ($12,105) | 4.14 ETH ($11,137) | 0.135 ETH ($363) | 0.1125 ETH ($303) | 0.1125 ETH ($303) |
| High | 100 x 10 ETH | 30 ETH ($80,700) | 27.6 ETH ($74,244) | 0.9 ETH ($2,421) | 0.75 ETH ($2,018) | 0.75 ETH ($2,018) |

If 40% of holder payouts leave through Zcash or Aleo at the full 0.20%, the route fee adds about $1, $9 and $59 per day in the three cases. The treasury is paid by volume. The route fee covers route operations and is not the business.

These are illustrations of the split under stated assumptions. Volume can fall to zero, and every figure falls with it.

---

## 9. Tokenomics

| Parameter | Value |
|---|---|
| Name / ticker | Stealthpad / $STEALTH |
| Total supply | 1,000,000,000, fixed, no mint function |
| Distribution | 100% on the Pons V2 bonding curve |
| Team allocation | none |
| Presale or private round | none |
| Launch path | StealthFactory on Pons V2, first launch on mainnet day |
| Snipe window | first 3 seconds taxed at 99% (a Pons V2 setting) |
| Graduation | the curve completes at 4.2 ETH and liquidity moves to the pool |
| Creator tax | 3%, paid to the $STEALTH StealthVault |
| $STEALTH round split | 94.5% holders, 3% caller, 2.5% treasury |
| Burn | none in Phase 1 |

A buyback share, if ever introduced, could only apply to new vault versions. Existing vaults keep their split for as long as they exist.

### 9.1 Why Pons V2 first

Every Stealthpad coin appears in the Pons V2 launch feed that Robinhood Chain traders already watch. The curve, the snipe window and graduation are familiar, so a trader buys without learning a new venue. Every coin starts under identical conditions with a public graduation target. The vault only has to be the creator fee recipient, so the venue exists on day one. A native Uniswap mode, with the vault as the pool's fee recipient and no curve, follows if launchers ask for it. The vault design does not change.

---

## 10. Risks

1. **Pons V2 recipient override.** The Pons V2 owner can queue a change of any coin's creator fee recipient behind a 3-day timelock. Each vault exposes `pendingOverride()` and `feeRouteIntact()`, and a watcher raises a public alert on every queued change. Funds already in a vault cannot be reached by it; future fees could be redirected.
2. **Graduated fee conversion.** After graduation some fee conversions depend on the Pons operator. The fork rehearsal proves the path before mainnet.
3. **Single attester.** One attester posts every root. The guardian window stops a wrong root; a missing root delays claims and loses nothing. Phase 2 adds a backup attester set.
4. **Guardian concentration.** The guardian is one address until Phase 2 moves it to a multisig. A compromised guardian cannot move funds, change splits or pause a vault, but it can propose a new attester. That proposal is public for two days before it takes effect, which is the window in which watchers alert and holders can react.
5. **Route liquidity.** A destination can lose solver liquidity. Quotes are live, and USDG remains available at all times.
6. **Traceability.** Claims are public on Robinhood Chain. Zcash lands transparent and is private only after shielding.
7. **Demand.** Launch demand on Robinhood Chain is thin without a campaign. Payouts depend on volume.
8. **Software.** Contracts can contain defects despite 23 unit tests, 4 fork tests against live Pons V2, and an external audit that precedes the $STEALTH launch.

The component-level analysis is in [THREAT_MODEL.md](./THREAT_MODEL.md).

---

## 11. Roadmap

| Phase | Window | Scope |
|---|---|---|
| 0 | done | Concept locked, payout routes measured, name and brand locked |
| 1 | weeks 1 to 4 (now) | StealthFactory and StealthVault, fork rehearsal against live Pons V2, indexer, attester and verifier, the app (board, launch, coin, claim, keeper, receipts), landing page and docs, external audit, mainnet day with $STEALTH first |
| 2 | months 1 to 2 | Guardian to multisig, backup attester set, Adopt mode for existing Pons coins, shielded Zcash delivery when the route supports it, automated receipts, keeper leaderboard, analytics page |
| 3 | months 2 to 4 | Claim relayer so claims need no ETH, native Uniswap mode if asked for, Monero when listed, claim widget for wallets |
| 4 | months 6 to 12 | Vaults on other EVM chains, SDK for launchpads, partner routes |

Status at the time of writing: the landing page and docs are live; the contracts, worker and app are built and proven end to end on a fork of Robinhood Chain (connect, buy, sell, launch with logo, run a round with the tip paid, claim USDG, live Zcash quote). Mainnet deployment, the audit and the $STEALTH launch are pending. No contract address exists for $STEALTH.

---

## 12. Disclaimer

Stealthpad is experimental software on public blockchains. $STEALTH and every coin launched through Stealthpad are digital tokens with no promise of value, profit or return. Payouts depend on trading volume, which can fall to zero. Smart contracts can contain bugs despite audits and tests. Nothing in this paper is investment, legal or tax advice. Use only funds you can afford to lose. The $STEALTH contract address will be published on launch day through official channels only. Stealthpad is independent and is not affiliated with Robinhood, Pons, NEAR, or the Zcash and Aleo foundations.

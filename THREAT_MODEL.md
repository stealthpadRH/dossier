# Stealthpad Threat Model

Version 0.1 · September 2026 · Pre-audit

This document models the threats against Stealthpad component by component using STRIDE (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege), then treats the abuse patterns specific to a fee-sharing launchpad. Every mitigation is tied to a constant, a check, or a published process that exists in the code today. Residual risk is stated where it remains.

Status: contracts, worker and app are built and proven on a fork of Robinhood Chain. Mainnet deployment and the external audit are pending. No $STEALTH contract address exists.

---

## 1. Scope and Assets

### 1.1 Components

| Component | Role | Holds funds | Holds a key |
|---|---|---|---|
| `StealthFactory` | Deploys vaults, launches coins on Pons V2, stores guardian and attester | No | No |
| `StealthVault` (one per coin) | Receives fees, runs rounds, verifies roots, pays claims | Yes: ETH before a round, USDG after | No |
| Attester | Computes snapshots, publishes leaf files, signs EIP-712 roots | No | Yes: signing key only |
| Guardian | Vetoes pending roots, proposes attester and guardian changes, opens or closes new launches | No | Yes: one address (multisig in Phase 2) |
| Keeper | Runs ready rounds, relays signed roots | Earns tips | Yes: a gas wallet |
| Indexer and read API | Follows chain events, serves board, coins, leaves, claimables, proofs | No | Database credential (server only) |
| App (`app.stealthpad.online`) | Board, launch, coin, claim, keeper, receipts | No | None in the browser |
| Route quoting | Holder's browser calls NEAR Intents 1Click directly | No | None |
| Pons V2 (external) | Bonding curve, fee escrow, creator fee recipient registry | Escrowed fees | Owner key (not ours) |
| NEAR Intents (external) | Solvers deliver ZEC and Aleo dollars | In flight | Not ours |

### 1.2 Assets, ranked

1. USDG held by vaults for unclaimed holder shares.
2. ETH in vaults and in the Pons fee escrow before a round.
3. Integrity of holder allocations (the Merkle roots).
4. Holder funds at the moment of claim (the `to` address).
5. Holders' trust in the official interface and contract addresses.
6. Availability of rounds, roots and claims.

### 1.3 Trust boundaries

```
 Holder browser ──(TLS)──► App + read API ──► Postgres (server only)
       │                         │
       │ wallet tx               │ reads chain
       ▼                         ▼
 ┌───────────────────── Robinhood Chain 4663 ─────────────────────┐
 │ StealthFactory ─► StealthVault ◄─ postRoot (attester signature) │
 │        │               │  ▲                                      │
 │        ▼               │  └── veto() guardian only              │
 │   Pons V2 curve + fee escrow (external owner, 3-day timelock)   │
 └────────────────────────┼─────────────────────────────────────────┘
       │                  │ claim(to = deposit address)
       │ quote request    ▼
       └──────────► NEAR Intents 1Click ──► solvers ──► ZEC t-addr / Aleo addr
```

Money moves only on chain. Stealthpad servers read and sign; none of them can send a vault's funds anywhere.

---

## 2. STRIDE by Component

Severity uses Critical, High, Medium, Low against the assets above, before mitigation.

### 2.1 StealthFactory

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Launch with an out-of-range creator tax | Tampering | Medium | `taxBps` must be one of 100 to 500 in steps of 100, else `BadTax` | None |
| A third party front-runs a launch to squat the predicted vault address | Spoofing | Low | `predict(launcher, salt, taxBps)` binds the launcher address and a random 32-byte salt into the CREATE2 address | None |
| Pointing the $STEALTH share at an arbitrary address | Elevation | High | `setStealthVault` may run once (`AlreadySet`) and only to a vault this factory deployed (`NotOurVault`) | None |
| Factory used to alter existing vaults | Elevation | Critical | The factory has no function that writes to a deployed vault; vault constants have no setter | None |
| Guardian closes new launches | Denial of service | Low | `setLaunchesOpen` affects new launches only; rounds and claims on existing vaults continue | Launches pause until reopened |

### 2.2 StealthVault

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Reentrancy during `claim` or `runRound` | Tampering | Critical | `nonReentrant` on both; `claimed[holder]` updated before transfer | None known; audit pending |
| Double claim | Tampering | Critical | Cumulative accounting: pays `cumulative - claimed[holder]`, reverts `NothingToClaim` otherwise | None |
| Claiming someone else's leaf | Spoofing | Critical | Leaf is `keccak256(keccak256(abi.encode(msg.sender, cumulative)))`; a proof for another address fails `BadProof` | None |
| Paying to the zero address or the vault itself | Tampering | Low | `BadRecipient` | None |
| Excess route fee | Tampering | Medium | `feeBps > MAX_ROUTE_FEE_BPS (20)` reverts `FeeTooHigh` | None |
| Root that over-allocates | Tampering | Critical | `total <= rounds[roundId].cumulativeHolders` (`TotalTooHigh`), `total >= active.total` (`TotalDecreased`), round must be newer (`StaleRound`) | See 3.5 |
| Swap manipulation during a round | Tampering | High | Min-out at `SLIPPAGE_BPS` 150 below the pool's 30-minute TWAP; per-round cap `IMPACT_CAP_BPS` 100 (1% of pool USDG) | Up to about 1.5% per round under adversarial conditions |
| Caller refuses its own tip | Denial of service | Low | `TipFailed` reverts that caller's transaction only; any other caller can run the round | None |
| $STEALTH vault or treasury refuses ETH | Denial of service | Low | A refused $STEALTH share falls back to the treasury; a refused treasury share stays for the next round | None |
| Upgrade, pause or admin withdraw | Elevation | Critical | No proxy, no owner, no pause, no withdraw function in the bytecode | None |

### 2.3 Attester

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Forged root from a non-attester | Spoofing | Critical | EIP-712 signature over `PostRoot(roundId, root, total, seed)` with domain `{name: "Stealthpad", version: "1", chainId, verifyingContract: vault}` must recover to `factory.attester()` (`BadSignature`) | None |
| Attester key compromise signs a false allocation | Tampering | Critical | Bounded by root checks in 2.2; `ROOT_DELAY` 7200 s before use; guardian veto; public verifier recomputes every root from chain data; watchers compare | Requires guardian response inside 2 hours |
| Replay of a signature on another vault or chain | Spoofing | High | Domain separator binds the vault address and chain id | None |
| Attester denies a published allocation | Repudiation | Medium | The signed root, seed and total are on chain; the leaf file is published with its hash (`x-leaf-hash`) and served immutable | None |
| Vetoed root posted again automatically | Tampering | Medium | After a veto the worker would re-post the same root on its next tick. Runbook: set `ATTESTER_ENABLED=0` first, fix the cause, then re-enable; the guardian can veto each repeat within `ROOT_DELAY` | Operational discipline; covered in the warden runbook |
| Attester goes silent | Denial of service | Medium | Funds stay in the vault; rounds keep running; roots resume when posted; cumulative leaves mean no round is skipped | Claims for new rounds wait; Phase 2 backup attester set |
| Attester key rotated to an attacker | Elevation | Critical | `proposeAttester` is guardian-only and takes effect after `ATTESTER_DELAY` (2 days); the proposal emits `AttesterProposed` | Detection window of 2 days plus 2 hours per root |
| Attester holds funds | Elevation | n/a | By design it never sends a transaction; the keeper relays its signature | None |

### 2.4 Guardian

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Guardian moves funds | Elevation | Critical | No such function exists; guardian powers are `veto`, `proposeAttester`, `proposeGuardian`, `setLaunchesOpen`, one-time `setStealthVault` | None |
| Guardian vetoes every root | Denial of service | High | Already active roots stay claimable; vetoes are public events (`RootVetoed`) | New rounds unclaimable until resolved; multisig in Phase 2 |
| Compromised guardian proposes a malicious attester | Elevation | Critical | `proposeAttester` takes effect only after `ATTESTER_DELAY` (2 days); the `AttesterProposed` event is public and watchers alert on it; anyone can see `pendingAttester` and `pendingAttesterAt` on the factory | A 2-day public window to react; the guardian moves to a multisig in Phase 2 |
| Guardian hands the role to an attacker | Elevation | High | Two-step transfer: `proposeGuardian` then `acceptGuardian` by the new address; events on both | Single key until Phase 2 |
| Guardian and attester compromised together | Elevation | Critical | 2-day attester delay makes the rotation visible; public verifier and watchers alert on any root that does not recompute | Unclaimed USDG in affected vaults is at risk; the main reason the guardian moves to a multisig in Phase 2 |

### 2.5 Indexer and read API

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Index loss or corruption | Denial of service | Medium | The index is derived entirely from chain data and can be rebuilt from the factory's deployment block | A rebuild needs an archive RPC: the public RPC keeps only about 15 minutes of state |
| Database tampering serves wrong proofs | Tampering | Medium | A wrong proof fails on chain (`BadProof`); the leaf binds `msg.sender`, so no proof can redirect a holder's funds | Misleading display until corrected |
| Database tampering shows false totals | Tampering | Low | Every figure is recomputable from vault views (`totalClaimed`, `rounds`, `unclaimed`) | Display only |
| Leaking claim destinations | Information disclosure | High | The server never receives a route destination; quotes go browser to NEAR Intents | None |
| Credential leak | Information disclosure | High | `DATABASE_URL` is runtime server env, never a build argument, never `NEXT_PUBLIC_*`; the database holds only public chain data and uploaded logos | Read access to public data |
| Scraping or flooding RPC fan-out | Denial of service | Medium | `/api/claimable` limited to 60 requests per minute per IP; wallet parameter validated as `0x` plus 40 hex; `no-store` on dynamic data, immutable caching on leaf files and images | Soft, per-process limit; the edge may add more |
| Logo upload abuse | Denial of service, Tampering | Medium | PNG, JPEG, WebP, GIF only; 4 MB cap; `limitInputPixels` 40,000,000; decoded, squared to 512 px and re-encoded to WebP; content-addressed by SHA-256; 30 uploads per hour per IP; accepted only once the factory is live; served with `x-content-type-options: nosniff` | None known |

### 2.6 App

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Script injection through coin name, description or links | Tampering | High | React escapes all text; website links must match `https://` at launch and render only when they do; logos served from the app's own re-encoded store | None known |
| Secrets in the browser bundle | Information disclosure | High | Only public configuration uses `NEXT_PUBLIC_*` (chain id, RPC URL, factory address, factory block, app URL); no key of any kind is shipped | None |
| Compromised container | Elevation | High | Standalone output in a non-root container (`USER node`); no Docker socket mounted; no credentials beyond the read database | Read access to public data |
| Dependency compromise | Tampering | High | Exact pins on `next`, `react`, `viem`; lockfile installs; automated dependency update alerts | Supply-chain risk common to all web apps |
| Rehearsal tooling reachable in production | Elevation | High | Fork-only routes require `DEV_TOOLS=1`, a chain id other than 4663, and a local RPC, else 404; they are excluded from the public console repo | None |
| Wrong network signing | Tampering | Medium | The wallet layer checks chain id 4663 before any write and prompts a switch | None |

### 2.7 Route quoting

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Typo in a destination address | Tampering | High | Zcash: base58check, version bytes `0x1CB8` (t1) or `0x1CBD` (t3), 4-byte double-SHA-256 checksum. Aleo: 63 characters, `aleo1` prefix, bech32m checksum constant `0x2bc830a3` | A valid address of the wrong owner passes |
| Shielded or unified Zcash address submitted | Tampering | Medium | Addresses starting `u1`, `zs1`, `zc` are refused before quoting with a plain explanation | None |
| Clipboard hijack substitutes a valid address | Spoofing | High | The address field shows the full destination; holders should compare its first and last characters with their wallet before signing | Depends on the holder's device |
| Server learns the destination | Information disclosure | High | Browser calls 1Click directly (`POST /v0/quote`, `GET /v0/status`); the app's servers never proxy it | None |
| Quote expires unfilled | Denial of service | Medium | `refundTo` is the holder's own Robinhood Chain wallet with `refundType: ORIGIN_CHAIN`; 30-minute deadline | Funds return as USDG after expiry |
| Short deposit | Tampering | Low | The quote is requested for the exact net amount (`claimable - fee`) the vault will send | None |

### 2.8 Pons V2 dependency

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Pons owner redirects a coin's creator fee recipient | Tampering | High | Pons V2 enforces a 3-day timelock; each vault exposes `pendingOverride()` and `feeRouteIntact()`; a watcher raises a public alert on every queued change | Future fees of the affected coin; funds already in the vault are untouchable |
| Fee escrow fails to credit | Denial of service | Medium | `sweepAndClaim()` is callable by anyone; rounds count vault balance plus escrow; fork tests against live Pons V2 | Delay only |
| Post-graduation fee conversion stalls | Denial of service | Medium | Fork rehearsal covers graduation before mainnet | Depends on the Pons operator |
| Snipers at launch | Tampering | Medium | Pons V2 taxes the first 3 seconds at 99% | None beyond Pons policy |

### 2.9 NEAR Intents

| Threat | Category | Severity | Mitigation | Residual |
|---|---|---|---|---|
| Solver failure or delay | Denial of service | Medium | Status polling shows each stage; expired quotes refund to the holder | Time |
| Route delisted or illiquid | Denial of service | Medium | Quotes are live and dry-run first; USDG route always available | Chosen route unavailable |
| Deposit linked to delivery | Information disclosure | Medium | Stated plainly in the interface and in [PRIVACY_SCOPE.md](./PRIVACY_SCOPE.md) | By design of the route |
| Zcash delivered to a transparent address | Information disclosure | Medium | Stated before signing; holder shields it on arrival | Until shielded |

---

## 3. Launchpad-Specific Abuse

### 3.1 Snapshot sniping

**Attack.** Buy one block before a round, sell one block after, collect a full share.

**Mitigation.** The seed is the round's own block hash; twelve blocks are drawn from the window `(previous round block, round block]`; weight is the mean balance over those twelve. A position held for $n$ of $N$ window blocks has expected weight $\beta \cdot n / N$. `COOLDOWN` of 3600 s keeps windows long. The draw is fixed by history, so it cannot be predicted while trading.

**Residual.** A trader who controls the round's block hash could bias the draw. On an L2 with a single sequencer this power is not available to ordinary participants, and the gain is capped at the trader's actual balance.

### 3.2 Sybil splitting

**Attack.** Split one bag across many wallets for a larger combined share.

**Mitigation.** Allocation is linear in average balance, so the split sum equals the whole. The eligibility floor of 100,000 tokens (0.01% of supply) makes fragments below it forfeit to the carry. Splitting is strictly worse than holding in one wallet.

### 3.3 Sandwiching rounds

**Attack.** Move the WETH/USDG price before `runRound()` and reverse after, extracting from the vault's swap.

**Mitigation.** `SLIPPAGE_BPS` 150 below the 30-minute TWAP bounds the worst accepted price; a single-block manipulation moves the TWAP little. `IMPACT_CAP_BPS` 100 limits any one round to 1% of the pool's USDG. The excess waits for the next round.

**Residual.** Up to 150 bps of one round's holder portion under worst-case conditions.

### 3.4 Empty-round griefing

**Attack.** Trigger rounds that cost gas and attester work while moving nothing.

**Mitigation.** `MIN_ROUND` 0.02 ETH makes a hollow round revert (`BelowMinimum`). The tip is 3% of the round, so a small round pays a small tip. `COOLDOWN` caps any vault at 24 rounds per day. Running a ready round is the intended behavior and is harmless.

### 3.5 Malicious root

**Attack.** A compromised attester signs a root that assigns holder USDG to attacker addresses.

**Mitigation.** Five layers: the vault enforces `active total <= total <= cumulative holder amount recorded through that round`, so a root can neither roll back nor over-allocate; `ROOT_DELAY` 7200 s before any claim may use it; guardian `veto()` during that window; the public verifier recomputes the root from chain data alone; a vetoed round is attested again. Rotating the attester needs `ATTESTER_DELAY` of 2 days.

**Residual.** Loss requires the guardian to miss a 2-hour window, or the guardian key to be compromised as well. A compromised guardian cannot move funds itself, but it can propose a new attester that takes effect after 2 days; that public window is when watchers alert. After any veto, operators set `ATTESTER_ENABLED=0` before fixing the cause, or the worker posts the same root again on its next tick. Phase 2 moves the guardian to a multisig and adds a backup attester set.

### 3.6 Phishing with fake contract addresses

**Attack.** Scammers post a "$STEALTH" contract address before launch, or clone a coin's name and ticker.

**Mitigation.** No $STEALTH contract address exists before launch day. It will be published on launch day through official channels only: `stealthpad.online`, `app.stealthpad.online`, and @stealthpadRH on X. There is no presale and no private round. The board lists only coins the factory deployed; anyone can confirm a coin with `StealthFactory.vaultOf(token)` returning a non-zero vault. The team never sends the first direct message.

### 3.7 Fake claim sites

**Attack.** A look-alike site asks for a token approval, a signature, or a seed phrase, or substitutes its own deposit address into a claim.

**Mitigation.** A real claim is one transaction to the coin's vault: `claim(cumulative, proof, to, feeBps)`. It needs no token approval, no off-chain signature, no permit, and never a seed phrase. For USDG, `to` is the holder's own wallet. For a route, `to` is the deposit address from a quote the holder's own browser requested. The only claim page is `app.stealthpad.online/claim`. Holders can confirm the vault address with `vaultOf(token)` before signing.

### 3.8 Keeper races

**Attack.** Bots front-run each other to collect the 3% tip.

**Assessment.** Accepted. Rounds run sooner, which serves holders. The tip is identical whoever calls.

### 3.9 Route fee bypass

**Observation.** `feeBps` is supplied by the claimant. A holder calling the vault directly can pass 0 on a route claim.

**Assessment.** Accepted. The contract enforces only the 20 bps ceiling that protects holders from the interface. The fee is a service charge of the official interface and never a condition of reaching one's own share.

---

## 4. Mitigations Mapped to Constants

| Constant or check | Value | Defends against |
|---|---|---|
| `MIN_ROUND` | 0.02 ETH | Empty-round griefing |
| `COOLDOWN` | 3600 s | Round spam, short sampling windows |
| Sample count | 12 blocks | Snapshot sniping |
| Seed | Round block hash | Predictable sampling |
| Eligibility floor | 100,000 tokens (0.01% supply) | Sybil dust |
| Exclusions | Curve, pool manager, hook, vault, token, factories, zero, burn | Infrastructure addresses taking shares |
| `SLIPPAGE_BPS` | 150 vs 30-minute TWAP | Swap sandwiching |
| `IMPACT_CAP_BPS` | 100 (1% of pool USDG) | Large-round price impact |
| `ROOT_DELAY` | 7200 s | Malicious or faulty roots |
| `ATTESTER_DELAY` | 2 days | Silent attester rotation |
| `MAX_ROUTE_FEE_BPS` | 20 | Excess fees from any interface |
| `TotalTooHigh`, `TotalDecreased`, `StaleRound` | Root checks | Over-allocation, rollback |
| `nonReentrant` | claim, runRound, postRoot | Reentrancy |
| Pons fee recipient timelock | 3 days (Pons V2) | Silent fee redirection |
| Claimable rate limit | 60 requests per minute per IP | API flooding |
| Upload limits | 4 MB, 40 MP, 30 per hour per IP | Image abuse |

---

## 5. Assurance Plan

| Activity | Status |
|---|---|
| Unit tests (Foundry) | 23 passing |
| Fork tests against live Pons V2 | 4 passing: launch, fee accrual, sweep, round, claim, graduation path |
| End-to-end fork rehearsal | Done: connect, buy, sell, launch with logo, round with tip, USDG claim, live Zcash quote |
| Public verifier | Built; recomputes roots from chain data alone |
| External audit | Before $STEALTH launches |
| Bug bounty | Opens with mainnet |
| Guardian multisig, backup attester set | Phase 2 |

---

## 6. Reporting a Vulnerability

Report privately through the security contact published on `stealthpad.online` once the bug bounty opens. Do not disclose publicly before a fix is deployed. Never share a seed phrase with anyone, including someone claiming to be from Stealthpad.

<sub>Stealthpad is experimental software. This model reflects the pre-audit design and will change with the audit's findings. Stealthpad is independent and is not affiliated with Robinhood, Pons, NEAR, or the Zcash and Aleo foundations.</sub>

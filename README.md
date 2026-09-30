<div align="center">

# Stealthpad Dossier

**The papers behind Stealthpad: the thesis, the threat model, and a plain statement of what private means.**

*Launch a coin. Holders get paid in Zcash. Nobody holds the keys.*

[![Docs](https://img.shields.io/badge/docs-stealthpad.online%2Fdocs-34B27B?style=flat-square)](https://stealthpad.online/docs)
[![Chain](https://img.shields.io/badge/chain-Robinhood%20Chain%204663-05110F?style=flat-square)](https://stealthpad.online)
[![Status](https://img.shields.io/badge/status-pre--launch-77D6A8?style=flat-square)](https://app.stealthpad.online)
[![Audit](https://img.shields.io/badge/audit-pending-299867?style=flat-square)](./THREAT_MODEL.md)

</div>

---

## Contents

| Document | What it answers | Read it if you are |
|---|---|---|
| [THESIS.md](./THESIS.md) | Why creator fees belong in a keyless vault, how rounds and the time-weighted random snapshot work (with the math), payout routes, $STEALTH and its tiers, revenue scenarios, tokenomics, risks, roadmap | A holder, launcher, researcher or partner |
| [THREAT_MODEL.md](./THREAT_MODEL.md) | STRIDE per component (factory, vault, attester, guardian, read API, app, route quoting, Pons V2, NEAR Intents) and launchpad-specific abuse, mapped to real constants | An auditor, integrator or security reviewer |
| [PRIVACY_SCOPE.md](./PRIVACY_SCOPE.md) | Exactly what is private, what is public, and what is not offered | Anyone choosing a payout route |

## The mechanism on one page

```
 trade on Pons V2 ──► creator tax (1 to 5%) + creator share of the 1% Pons fee
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  StealthVault (per coin)     │   no owner · no withdraw
                    │  ETH accrues                 │   no upgrade · no pause
                    └──────────────┬───────────────┘
      vault + escrow ≥ 0.02 ETH    │   and 1 h since last round
      anyone calls runRound()      ▼
         ┌──────────┬──────────────┼──────────────┬─────────────┐
         ▼          ▼              ▼              ▼             │
     3% caller  2.5% $STEALTH  2.5% treasury  92% holders ──► swap to USDG
                  vault                                         │
                                                                ▼
          attester: seed = round block hash · 12 random past blocks
          average balance · floor 100,000 tokens · cumulative Merkle root
                                                                │
                                   2 h delay, guardian may veto ▼
                              claim(cumulative, proof, to, feeBps)
                     ┌──────────────────┼─────────────────────┐
                     ▼                  ▼                     ▼
               USDG, same wallet   Zcash t-addr,         USDCx / USAD
                                   holder shields it     on Aleo
```

## Key numbers

| Parameter | Value |
|---|---|
| Split per round | 92% holders · 3% caller · 2.5% $STEALTH vault · 2.5% treasury |
| Round threshold | 0.02 ETH, at most one round per hour per coin |
| Snapshot | 12 blocks drawn from the window since the last round, seeded by the round's block hash |
| Eligibility floor | 100,000 tokens (0.01% of supply) |
| Root delay | 2 hours, guardian veto only |
| Route fee ceiling | 0.20%, enforced by the vault |
| $STEALTH supply | 1,000,000,000, 100% on the Pons V2 curve, no team allocation, no presale |

## Sibling repositories

| Repository | Contents |
|---|---|
| [stealthpad-foundry](../stealthpad-foundry) | Solidity contracts and Foundry tests |
| [stealthpad-conduit](../stealthpad-conduit) | Read API reference and TypeScript SDK |
| [stealthpad-console](../stealthpad-console) | The app at app.stealthpad.online |
| [stealthpad-warden](../stealthpad-warden) | Indexer, attester, keeper, public verifier |

## Links

| | |
|---|---|
| Site | https://stealthpad.online |
| Docs | https://stealthpad.online/docs |
| App | https://app.stealthpad.online |
| X | [@stealthpadRH](https://x.com/stealthpadRH) (reserved, opening soon) |
| GitHub | [github.com/stealthpadRH](https://github.com/stealthpadRH) (reserved, opening soon) |

> **No contract address exists for $STEALTH yet.** It will be published on launch day through official channels only. There is no presale. The team never sends the first direct message.

---

<div align="center">
<sub>Stealthpad is experimental software on public blockchains. $STEALTH and every coin launched through Stealthpad are digital tokens with no promise of value, profit or return. Payouts depend on trading volume, which can fall to zero. Nothing here is investment, legal or tax advice. Stealthpad is independent and is not affiliated with Robinhood, Pons, NEAR, or the Zcash and Aleo foundations.</sub>
</div>

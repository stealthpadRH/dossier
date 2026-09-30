# What Private Means Here

Stealthpad shares a coin's creator fees with its holders and lets each holder choose where the payout lands. Some landings can be private. Everything before the landing is public. This page states the line exactly, so nobody has to guess.

---

## Private

| What | When it becomes private |
|---|---|
| Zcash (ZEC) | After you shield it in your Zcash wallet. From then on its balance and every later move are private. Zashi and most Zcash wallets shield in one step. |
| USDCx or USAD on Aleo | When you move it with a private transfer from your Aleo wallet. |

## Public

| What | Where it can be seen |
|---|---|
| The Robinhood Chain wallet that held the coin | Robinhood Chain |
| That wallet's share of every round | The published leaf file of each round, and the vault's root |
| The claim transaction and its amount | Robinhood Chain (`Claimed` event) |
| The route deposit address the claim paid | Robinhood Chain (the claim's `to`) |
| The landing address, until you shield or privately move the payout | The destination chain, and the route's own records, which link a deposit to its delivery |

Zcash lands at a **transparent** address (`t1...`). The route rejects shielded (`zs1...`) and unified (`u1...`) addresses today. Shielded delivery is on the Phase 2 roadmap, for the day the route supports it.

## Not offered, on purpose

- **No mixing.** Stealthpad is not a mixer. It does no batching, uses no fixed denominations, and adds no delays designed to obscure who claimed.
- **No custody of payouts.** Your own browser requests the route quote, and the vault pays the deposit address that quote returns. Stealthpad never holds a payout in transit.
- **No destination on our servers.** The address you type goes from your browser to NEAR Intents directly. It is validated in your browser (base58check for Zcash, bech32m for Aleo) and never sent to or stored by Stealthpad.

## What Stealthpad does keep

The read API and its database hold public chain data only: coins, trades, balances, rounds, roots, leaves and claims, plus the coin logos launchers upload. They hold no route destinations, no names, no emails, and no account of any kind. Wallet connection happens in your browser.

## Plain answers

**Is my claim private?** No. Your Robinhood Chain wallet, your share and your claim are public, and the route's records link a claim to the address it lands at. What can be private is what happens next: shield the Zcash, or move Aleo dollars with a private transfer.

**Is the USDG route private?** No. USDG lands in the same wallet on Robinhood Chain, in full public view. It has no route cost and no fee.

**Can Stealthpad see where my Zcash went?** Stealthpad's servers never receive the destination. The address is public on the Zcash chain until you shield the funds, and the route operator can link your deposit to its delivery.

---

<sub>Stealthpad is independent and is not affiliated with Robinhood, Pons, NEAR, or the Zcash and Aleo foundations. Privacy properties of Zcash and Aleo are those of their own protocols and wallets.</sub>

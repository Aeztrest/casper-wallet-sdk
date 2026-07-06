# casper-wallet-sdk

A programmable, on-chain spending-limit vault for [Casper Network](https://casper.network).
Casper has no native account-abstraction feature for scoped spending
permissions, so this fills that gap as a standalone contract: `PaymentGuard`.

## What it does

An owner deposits a CEP-18 token into the vault and grants each merchant:

- a **per-transaction cap**, the largest single payment that merchant can receive, and
- a **rolling 24-hour cap**, the cumulative amount that merchant can receive per day.

The owner can then delegate day-to-day spending to an **agent** key (a hot
wallet, an automated script, an AI agent, anything) via `set_agent`. The
agent calls `pay(merchant, amount)` to settle payments without the owner
signing each one. The on-chain caps are the entire authorization model. A
payment above a cap, to an unregistered merchant, or to a paused/revoked
merchant reverts on-chain, regardless of who calls `pay`.

This is useful anywhere you want "let this key spend on my behalf, but only
within limits I set once": recurring payments, agentic/autonomous spending,
subscription-style billing, or any dApp that wants programmable spend
authority without requiring a human signature on every transaction.

## Entry points

| Entry point | Caller | What it does |
|---|---|---|
| `init(owner, token)` | anyone (constructor) | One-time setup: records the owning account and the CEP-18 asset. |
| `set_allowance(merchant, cap_per_tx, cap_per_day)` | owner | Grants/updates a merchant's caps, resets to `Active`. |
| `set_agent(agent)` | owner | Delegates `pay` to `agent`. Pass the owner's own address to revoke delegation. |
| `pause(merchant)` / `resume(merchant)` / `revoke(merchant)` | owner | Changes a merchant's status. |
| `deposit(amount)` | anyone (typically owner) | Pulls `amount` into the vault. Caller must `approve` the vault first. |
| `pay(merchant, amount)` | owner or the designated agent | Settles a payment from the vault to `merchant`, enforcing both caps. |
| `withdraw(amount)` | owner | Pulls funds back out of the vault. |
| `get_allowance(merchant)` / `available_today(merchant)` | anyone (view) | Inspect a merchant's current caps / remaining daily budget. |

## Build

Requires [Odra](https://odra.dev) and the Odra CLI:

```sh
cargo odra build
```

Produces `wasm/PaymentGuard.wasm`, ready to install on a Casper node.

## Test

```sh
cargo test --lib
```

Runs against Odra's MockVM. No live network required.

## Design notes

- **One agent slot.** Delegating to a new agent overwrites the previous one.
  There's no multi-agent support yet.
- **Caps are per-merchant**, not per-(merchant, asset) or per-origin. A
  merchant address is trusted as a single unit once approved.
- **The rolling 24h window** resets based on Casper block time
  (`get_block_time`), not wall-clock time as observed by any particular client.
- Only the owner or the designated agent may call `pay`. An arbitrary third
  party can't force the vault to pay an already-approved merchant ahead of
  schedule, which would otherwise let anyone grief the daily cap before the
  real agent needs it.

## Origin

Extracted from [Baret](https://github.com/Aeztrest/CasperBaret), a Casper
wallet-level transaction firewall, where `PaymentGuard` backs its x402
agentic-payment spending caps. Built for a Casper Network hackathon
submission alongside two companion projects: [`casper-usdc`](https://github.com/Aeztrest/casper-usdc)
(a stablecoin with gasless meta-transfers) and [`x402-casper`](https://github.com/Aeztrest/x402-casper)
(the x402 HTTP micropayment protocol on Casper).

## License

MIT

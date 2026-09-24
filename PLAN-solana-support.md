# Plan — Solana chain support

Status: **PLAN ONLY, nothing built.** Drafted 2026-08-19 after live testing against
`gmgn-cli market trending --chain sol`. Do not implement until told.

**Answer to the question: yes, straightforwardly.** gmgn-cli already supports
`sol` as a first-class chain, and `config.mjs` was built with multi-chain
validation from day one (`VALID_CHAINS`, `EVM_CHAINS`, `VALID_FILTER_TAGS.sol`
already exist and are unused so far). But Solana is not "Robinhood with a
different chain string" — three assumptions baked into the code are
Robinhood/EVM-specific and were verified live to not carry over.

---

## 1. What transfers unchanged

- `market trending` response shape — same fields, same envelope, same
  `price_change_percent1h` semantics, same PERCENT units (verified live: sol
  sample median matched the same order of magnitude as Robinhood's).
- All 38 `trending.range` params, `min_created` duration strings, `order_by`
  values — chain-agnostic on the API side.
- `market kline` — same shape, same MILLISECOND timestamps, same string
  numbers. The freshness/AGE/OFF-HI logic needs no changes.
- Telegram delivery, the AI `--explain`/`--listen` layer, config validation
  machinery, `--survey`, `--print-config` — none of this touches chain
  specifics.
- `--min-price-change-percent` as a genuine percent (not ratio) — would need
  reconfirming on sol data, but the mechanism is the same field.

## 2. What does NOT transfer — verified live, not assumed

### 2a. Address format
Robinhood is EVM (`0x` + 40 hex). Solana is **base58**, no `0x` prefix, no fixed
length (typically 32–44 chars). Verified: `7g2xyrq9Fk1TvFiqqcmLJAQ9jAtfHobmrdzzM9Dp2SpV`.

**Three call sites hard-code the EVM regex** (`screener.mjs:260, 365, 751`):
```js
/^0x[a-fA-F0-9]{40}$/
```
These gate the kline enrichment call and the Telegram bare-address detector.
On Solana they'd silently reject every valid address — kline enrichment would
never run, and `--listen` would never recognize a pasted token address. This
is not a hidden bug to discover later; it's the first thing that would break.

### 2b. No tokenized-equity problem on Solana (as far as tested)
Robinhood's `isTokenizedStock()` filter (name suffix `• Robinhood Token` /
`pool_robinhood_stock_amm`) exists because Robinhood Chain mixes real-world
equities into the trending feed. On a 100-token Solana sample, ordered by
market cap, the launchpad platforms were `Pump.fun` (91), `letsbonk` (2),
`stonkfun` (6), `moonshot_app` (1) — `stonkfun` turned out to be a themed
**memecoin** launchpad (STONK, GTA6, MANLET, BUTTHOLE), not tokenized stocks.
No RWA-suffix names appeared.

**Caveat, stated honestly:** Solana does have a real tokenized-equity product
— `xstocks` (Backed Finance/Kraken-backed) is a known platform value in
gmgn-cli's own sol vocabulary — it just didn't appear in this one 100-token
market-cap-ordered sample. Before shipping, run `--survey` against a live sol
pull and grep for `launchpad_platform === "xstocks"`; if it shows up, it needs
the same filter `isTokenizedStock()` already has, generalized to check both
chains' markers instead of only Robinhood's.

### 2c. Risk fields mean something different — and rug_ratio actually works here
On Robinhood Chain, `rug_ratio` was **flat 0.0000 for every single token** —
verified across multiple samples — making it dead weight in the risk gate.
**On Solana it is a live, informative signal**: sampled distribution was
`min=0 median=0.2 p90=0.93 max=1`. The risk gate that was cosmetic on
Robinhood does real work here.

The inverse also holds: `is_open_source` / `is_renounced` were the working
EVM contract-verification signals on Robinhood. On Solana, SPL tokens have no
bytecode to verify — both fields read `0`/`false` for every token (verified).
The Solana-native equivalents are `renounced_mint` / `renounced_freeze_account`
(booleans on whether mint/freeze authority was given up) — already present in
the response and already the basis of `VALID_FILTER_TAGS.sol` in `config.mjs`.

### 2d. Filter tags — different vocabulary, different defaults
`config.mjs` already has this coded (`VALID_FILTER_TAGS.sol` vs `.evm`) but
it's never been exercised. Sol tags: `renounced`, `frozen`, `burn`,
`token_burnt`, `has_social`, `not_wash_trading`, … — no `not_honeypot`,
`verified`, or `locked` (EVM-only). The current `config.json` sets
`"filters": ["not_honeypot"]`, which is an **EVM tag and would fail Solana
validation outright** (or silently no-op server-side, which the validator is
specifically designed to catch and reject).

Verified live: `renounced`+`frozen` explicitly vs. omitted returned
**byte-identical address sets** — confirming the docs' claim that sol has
server-side default filters (`renounced frozen`) applied even when you pass
nothing. Same "your filters may be no-ops" trap as Robinhood, different cause.

### 2e. Platform vocabulary
Sol platforms are a completely different list — `Pump.fun`, `letsbonk`,
`bags`, `moonshot_app`, `pump_agent`, `heaven`, `boop`, `xstocks`,
`pool_ray`/`pool_meteora`/`pool_orca`, etc. (documented in `config.mjs`
comments already, unused). `trending.platforms` in the current config is
empty so this doesn't block anything, but any sol-specific filtering by
platform needs this list, not Robinhood's.

---

## 3. Scope decision (yours, 2026-08-19): one chain running at a time

Simplifies §3 substantially — no concurrent `--listen`/`--serve` conflict, no
"which chat gets which chain's alerts" question. One process, one active
chain, switchable.

### 3.1 Config split: shared vs per-chain

Two chain-specific files, everything else shared:

| File | Contents |
|---|---|
| `config.json` | Robinhood: `chain`, `trending.*`, `screen.*`, `risk.*` |
| `config.sol.json` | Solana: same shape, sol-calibrated values (filter tags, `rug_ratio` gate, `renounced_mint`/`renounced_freeze_account`, market-cap/volume bands from a real `--survey`) |
| `.env` | Unchanged — Telegram + LLM creds, shared across chains |

`telegram.*` and `ai.*` blocks stay duplicated (not shared/imported) —
simpler to reason about, and both chains alert into the same bot/chat by
construction since only one runs at a time anyway.

### 3.2 A small state file remembers the active chain across restarts

`active-chain.json`: `{"chain": "robinhood"}` (or `"sol"`). Read at startup to
pick which config file loads by default; written whenever the chain is
switched, so a crash/restart resumes on whichever chain you last selected —
not silently back on Robinhood.

---

## 4. Switching chains — from Telegram, and from the CLI

**Yes, both are straightforward,** given §3's one-chain-at-a-time scope. The
mechanism is the same either way: reload the chain-specific config and swap
what the running `watchLoop` reads, without restarting the process (so the
Telegram listener — the very thing receiving the switch command — never drops).

### 4.1 From Telegram

```
you:  /chain sol
bot:  Switched to sol. Re-screening now...
bot:  [next alert uses Solana config]

you:  /chain
bot:  Currently: robinhood
```

Recognized as a new branch in `listenLoop`'s message router, checked before
the existing address-detection / free-prompt logic (so `/chain` itself is
never mistaken for a raw LLM prompt). Validates the name against the known
chain configs before switching — an unrecognized chain name replies with the
valid list instead of silently doing nothing.

### 4.2 From the CLI

```bash
node screener.mjs --chain sol --serve      # start directly on sol
node screener.mjs --chain robinhood --once # one-off pass on robinhood
```

`--chain <name>` is sugar for `--config config.<name>.json` (with
`robinhood` mapping to the existing bare `config.json`) — resolved before
`loadConfig` runs, so it composes with every existing flag.

### 4.3 The mechanism that makes a live switch possible

`watchLoop` currently closes over one `cfg` object for its entire lifetime —
switching chains means it needs to pick up a *different* `cfg` mid-loop
without restarting the process (restarting would drop the Telegram listener
that received the switch command in the first place).

Fix: wrap the active config in a mutable holder both loops read from —

```js
const active = { cfg: loadConfig(initialConfigPath) };
```

`watchLoop` reads `active.cfg` at the top of every iteration instead of
capturing `cfg` once; the `/chain` handler in `listenLoop` calls
`loadConfig()` on the new file, writes `active-chain.json`, replaces
`active.cfg`, and (optionally) fires an immediate out-of-schedule pass so the
switch is visible right away rather than waiting out the rest of the old
5-minute interval.

### 4.4 One nuance worth flagging

A `/chain` switch that lands **mid-pass** (a screen already running against
the old chain when the command arrives) should let that in-flight pass finish
against the chain it started on, then apply the switch on the *next*
iteration — not abort mid-request. Simple to get right (check `active.cfg`
only at loop-top, not mid-function), easy to get wrong if rushed, so it's
called out explicitly here rather than left implicit.

## 5. Build order

1. **Fix the three EVM-regex call sites** to accept both formats — the address
   format actually determines which chain a pasted address belongs to, so
   this is chain-detection infrastructure, not cosmetic:
   ```js
   const isEvmAddr = (a) => /^0x[a-fA-F0-9]{40}$/.test(a);
   const isSolAddr = (a) => /^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(a); // base58, no 0/O/I/l
   ```
2. **Generalize `isTokenizedStock()`** to a per-chain marker table instead of
   a single Robinhood-only regex, even if Solana's entry starts empty —
   confirmed clean data shouldn't calcify into hard-coded assumptions.
3. **Run `--survey` against live sol data** to calibrate `trending.range`
   (marketcap/volume/liquidity floors) the same way Robinhood's were derived
   from real percentiles, not guessed.
4. **`config.sol.json`**: `chain: "sol"`, sol filter tags (`renounced`,
   `frozen`, …), risk gate using `rug_ratio` for real (it's live data here) +
   `renounced_mint`/`renounced_freeze_account` in place of
   `is_open_source`/`is_renounced`, sol-appropriate `trending.range` from
   step 3, `ai.promptTemplate` still works as-is (chain-agnostic wording).
5. **`active-chain.json`** state file + the `active` mutable-config holder
   (§4.3) that both loops read from.
6. **`/chain <name>` in `listenLoop`** (§4.1) + **`--chain <name>` CLI sugar**
   (§4.2).
7. **package.json scripts**: `sol:once` / `sol:watch` / `sol:serve` as a
   convenience on top of `--chain sol`, mirroring the existing ones.

## 6. Open questions

1. Confirm the `xstocks` caveat in §2b with a live `--survey` before
   assuming Solana needs no equity filter at all.
2. Should `/chain` restrict who can invoke it? It already inherits the
   existing Telegram allowlist (only your `TELEGRAM_USER_IDS` can message the
   bot at all), so this is likely a non-issue — flagged only because a chain
   switch is more consequential than an `--explain` query.
3. On switch, fire an immediate re-screen pass (§4.3) or wait for the next
   scheduled tick? Recommend immediate — otherwise `/chain sol` can sit
   silent for up to `watch.intervalMs` before you see anything change.

# Deferred work — rh-pump-screener

Things considered and consciously postponed. Each entry says what it is, why it
was raised, and enough detail to pick it up without rediscovering the problem.

Last updated: 2026-09-20

---

## 1. Telegram alert cooldown (per token)  — DONE 2026-09-20

**Was:** `--watch` sent a Telegram message every pass. The same tokens kept
matching pass after pass, so the same three symbols arrived ~12 times an hour
and you stopped reading them.

**Now:** `telegram.cooldownMin` (config.json: 60) suppresses a token that has
already been alerted, keyed per chat. `telegram.cooldownStateFile`
(`./data/screener-alerts.json`) persists it across restarts. `0` disables.

Decisions made, so they are not relitigated:

- **Watch-only.** `runOnce(cfg, { useCooldown })` — only `watchLoop` passes
  true. A single manual pass always shows everything and never advances the
  clock, so running `node screener.mjs` by hand cannot silence the watch loop.
- **The terminal is never suppressed.** `render()` always prints every match;
  only the Telegram alert is filtered. The suppressed count appears in the
  message's `filtered:` line as `N in cooldown`.
- **Time-based only.** A token that drops off the list and comes back still
  waits out its window — the clock is keyed on last-alerted time and nothing
  clears it early. This is the opposite of `/root/robinhoodscreener`, which
  re-alerts on return. If you want that behaviour you must track per-pass
  presence and clear the entry on absence; the store has no hook for it.
- **Per chat, like the new-token loop.** Key is `chatId:chain:address`, so two
  Telegram users have independent clocks and a new recipient is not born
  already silenced.
- **The clock only advances on confirmed delivery.** A failed send leaves the
  token eligible rather than silently swallowing it for the window.

Reuses `AlertCooldownStore` from `new-token-alerts.mjs` unchanged — it was
already generic. The validator hard-errors if `telegram.cooldownStateFile`
equals `newTokenAlerts.stateFile`, since a shared file would let each alert
path suppress the other.

---

## 2. Supertrend / TA indicators — scratched 2026-08-18

**Status.** Explicitly dropped ("scratch that for now"), not blocked.

**Established fact:** gmgn-cli has NO indicator support. The whole package was
grepped for supertrend / rsi / macd / bollinger / ema / atr / stochastic —
zero hits. All 31 API routes and ~130 CLI flags were enumerated; there is no
indicator endpoint and no indicator flag. `market kline` (raw candles) is the
only price-series data. Indicators must be computed locally.

The `indicatorInterval` / `requireBullishSupertrend` style params are NOT gmgn
fields — they come from `/root/robinhoodscreener/user-config.example.json`.

**Cheap path if resumed:** the freshness stage ALREADY fetches 1m candles per
surviving token, so indicators cost **no extra API calls** — just widen
`freshness.lookbackMin`, since supertrend at `period: 10` needs well over 70
bars to stabilise.

`/root/robinhoodscreener/indicators.js` already implements `computeRSI`,
`computeBollingerBands`, `computeATR`, `computeSupertrend` and presets
(`supertrend_break`, `supertrend_or_rsi`, `rsi_plus_supertrend`). Port it rather
than rewrite — but note it imports that project's `config.js` and `gmgn.js`, so
it needs decoupling first.

---

## 3. Hidden columns — off by choice 2026-08-18

`output.columns` currently omits `rvol`, `age`, `offHigh` ("for now I dont need
those"). Add the strings back to the array to restore them; order in the array
is the order in the table.

Never displayed but available: `swaps`, `holders`, `recentShare`.

**Important:** hiding a column does NOT save API calls. `age` / `offHigh` /
`when` all come from the same single kline call per token. Only
`freshness.enabled: false` actually skips that call — and that removes `when`
too, which is the last remaining stale-pump defence.

---

## 4. `total_fee` filtering — not available on `market trending`

`min_total_fee` / `max_total_fee` sit in `trending.range` as `null` placeholders
and are inert. They are **`market trenches`** parameters; `market signal` has its
own `total_fee_min` / `total_fee_max` (note the flipped word order). There is no
total-fee filter on `market trending` at all.

The validator warns while they are null and hard-errors the moment either is
given a real number — because the API silently ignores unknown range metrics
rather than rejecting them, so a value there would be an invisible no-op.

To make it real, a separate `market trenches` query has to be added alongside
the trending one (weight 3 vs 1).

---

## 5. LP execution — out of scope for gmgn-cli

gmgn-cli has no liquidity-provision surface (no add/remove_liquidity,
mint_position, pool_create; `token pool` is read-only). This screener finds
candidates; it cannot open a position.

`/root/unicrit` is the LP bot — "Telegram bot for Uniswap v3/v4 single-sided LP
on Robinhood Chain and BSC", chain id 4663, using grammy + viem/ethers +
Uniswap SDK. That is where an execution leg belongs, not here.

Caution already noted: providing liquidity into a token that just pumped means
taking the other side as it mean-reverts. On a low-liquidity memecoin,
impermanent loss dominates fee income. A `cooling` verdict after a completed
move is likely a friendlier LP entry than a `live` one.

---

## 6. AI `--explain` fact block — deferred 2026-08-18

**Built and working:** `node screener.mjs --explain <address>` sends
`ai.promptTemplate` (in `config.json`, `{address}` substituted) to the LLM and
posts the reply to Telegram. Uses meridian's DeepSeek key, no new deps.

**Known limitation, accepted for now.** The prompt contains only the address. An
LLM cannot resolve a hex address, so it answers from nothing. Observed live: for
`0x298348d5b2e45c774e3ee4f1a0924071dfbdc8c7` (swappy, Robinhood Chain) it
confidently replied *"Token: Hood, Symbol: HOOD, Network: Ethereum"* — wrong
token, wrong chain, stated with no hedging.

**The fix, if picked up:** paste a fact block into the prompt, the way meridian
does at `index.js:599`. Meridian never hands the model a bare address — it builds
~15 lines of metrics/audit/tags/smart-wallets, of which the Jupiter narrative is
only ONE line, sanitised via `sanitizeUntrustedPromptText(text, 500)`.

gmgn already returns the equivalent signals for free in the trending response.
For swappy they were genuinely informative: `twitter_create_token_count: 14` (X
account tied to 14 launches), `image_dup: 4` (recycled logo), `creator_close:
true` (dev sold out — meridian's prompt calls this bullish), `cto_flag: 1`,
`smart_degen_count: 31`, `renowned_count: 10`, plus top-10 %, dev hold rate,
bundler rate, sniper count, holders, liquidity, age.

Cost: one function. No extra API calls for tokens already in a pass.

### Narrative sources — investigated, all dead ends for Robinhood Chain

Do not re-derive this:
- **Meridian's LLM has no web search.** 44 tools, none browse. `search_pools` is
  Meteora pools; `get_token_info` is Jupiter's asset index.
- **Jupiter ChainInsight** (`datapi.jup.ag/v1/chaininsight/narrative/{mint}`) —
  meridian's only prose narrative source. Solana-only, base58 mints.
- **OKX Web3** — risk signals, not narrative. Tested directly 2026-08-18 for
  chainIndex 4663 AND 56: both return an **x402 payment-required** response now,
  not data.
- **`api.agentmeridian.xyz`** — Solana/Meteora only.

Conclusion: no narrative *source* is reusable. Only the *pattern* (fact block +
GOOD/BAD criteria + untrusted-data rule) transfers.

### Also worth knowing
`deepseek-v4-flash` is a reasoning model: it spent 5,836 hidden reasoning tokens
before writing a visible character. `ai.maxTokens` is set to 12000 for headroom —
at 900 the reply came back completely empty with `finish_reason: "length"`.

---

## 7. Solana chain support — deferred 2026-08-19

**Full plan lives at `/root/rh-pump-screener/PLAN-solana-support.md`.** Do
not redo the investigation before reading it — the plan is grounded in live
gmgn-cli calls, not extrapolation. Answer to "is it possible": yes. Scope
decided with the user: one chain runs at a time, switchable via `/chain sol`
in Telegram or `--chain sol` on the CLI.

### Non-obvious findings from the plan (so a future session doesn't re-derive)

- **Address regex is chain-specific and gates real behavior.** Three call
  sites in `screener.mjs` (lines 260, 365, 751) hard-code `/^0x[a-fA-F0-9]{40}$/`
  for the kline enrichment and the Telegram bare-address detector. On Solana,
  valid base58 addresses (32-44 chars, no `0`/`O`/`I`/`l`) get silently
  rejected there — kline enrichment never runs. Fix these first; they're
  chain-detection infrastructure, not cosmetic.
- **`rug_ratio` role flips between chains.** On Robinhood it was flat 0.0000
  for every token — cosmetic filter, no work done. On Solana it's a live
  signal (sampled `min=0 median=0.2 p90=0.93 max=1`) and the risk gate that
  was dead weight actually filters here.
- **`is_open_source`/`is_renounced` are EVM-only.** SPL tokens have no
  bytecode; both fields read 0/false for every Solana token. The Solana
  equivalents are `renounced_mint` and `renounced_freeze_account` (both
  already present in the response), keyed by the `renounced`/`frozen` filter
  tags in `VALID_FILTER_TAGS.sol`.
- **Filter-tag vocabulary is per-chain.** Current `"filters": ["not_honeypot"]`
  is EVM-only and would fail validation on `chain: "sol"`. `config.mjs`
  already has `VALID_FILTER_TAGS.sol` coded and unused.
- **Same "your filter is a no-op" trap as Robinhood, different cause.** Sol
  server-side default filters (`renounced frozen`) apply even when you pass
  nothing — verified: sol trending with and without those tags returned
  byte-identical address sets.
- **Solana tokenized-stock caveat: `xstocks` exists** as a real gmgn-cli
  platform value (Backed Finance/Kraken-backed) but didn't appear in the
  100-token market-cap-ordered sample I tested. Before shipping, `--survey`
  live sol data and check whether the `isTokenizedStock()` filter needs a
  sol entry too. Do NOT assume clean.

### The one design decision worth remembering

**Both loops must read the active config from a shared mutable holder**, not
capture it at startup. Otherwise a `/chain` switch can't take effect without
process restart, and restart kills the Telegram listener that just received
the command. `watchLoop`'s in-flight pass must finish on the chain it started
on and pick up the new config at the next iteration — check `active.cfg` at
loop-top only, not mid-function.

See PLAN-solana-support.md for the full build order and three still-open
questions (all documented, none blocking).

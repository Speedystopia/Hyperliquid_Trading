# HYPE-USDC short grid — Hyperliquid

`hype-short-grid.html` is a self-contained execution dashboard for an adverse-averaging short grid on the Hyperliquid HYPE perpetual. No dependencies, no build step, no server: open the file in a browser and the market-data feed starts.

| | |
|---|---|
| Venue | Hyperliquid |
| Default instrument | HYPE perpetual |
| Settlement currency | USDC |
| Market data | `POST api.hyperliquid.xyz/info` with `{"type":"metaAndAssetCtxs"}` |
| Fields consumed | `markPx`, `midPx`, `oraclePx`, `funding` |
| Assumed taker rate | 0.045% |
| Live-order signing | EIP-712 |

The info endpoint is public, requires no key, and returns `access-control-allow-origin: *`. Nothing needs to be proxied to read the market.

---

## The loop

1. **Poll the mark price**, every 2 seconds by default.
2. **Sell 5 HYPE** short at the mark.
3. **Sell 5 more** whenever the mark reaches the last fill price × 1.005. The reference is always the last fill, never the break-even or the price the cycle opened at.
4. **Recompute the break-even** after every fill.
5. **Cover the entire position** as soon as the mark trades through BE × 0.995.
6. **Record the cycle**, clear the state, wait 60 seconds, then reopen at whatever the mark happens to be.

**Exposure cap.** At 35 HYPE — seven clips of five — further adds are rejected. The position stays at full size until step 5 triggers. There is no stop-loss and no automatic exit at a loss: the only way out is mean reversion, or a manual cover.

---

## Break-even, net of fees

The displayed break-even is not the plain size-weighted average of the entries. It carries both the taker fee already paid on the way in and the one that will be paid on the way out.

For a short of size `sz` at average entry `P`, with taker rate `f`, the flat cover price `X` solves:

```
sz·(P − X) − f·sz·P − f·sz·X = 0
        ⇒   X = P · (1 − f) / (1 + f)
```

The cover level shown on the dashboard is then `X × 0.995`.

The rate is **hardcoded at the most punitive point of the schedule**: tier 0, base rate, 0.045% taker. No 14-day volume tier, no HYPE staking discount and no referral discount are assumed. If your effective rate is better, the displayed break-even is simply conservative and the realized exit will be marginally more favourable than the level shown.

---

## Parameters

| Field | Purpose |
|---|---|
| Perp instrument | The coin queried in the perp universe, `HYPE` by default |
| Clip size | Quantity per order |
| Exposure cap | Maximum position size; the guardrail on step 3 |
| Add on +% | Distance above the last fill that triggers an add |
| Cover on −% | Distance below the break-even that triggers the full cover |
| Cooldown after cover | Idle time between one cycle closing and the next opening |
| Polling interval | Market-data request frequency |
| Execution venue | Simulated fills, or live orders through a local signer |

Two requests per tick at weight 20 each — 600 weight per minute at the default interval, against a budget of 1,200 per minute per IP.

Indicative USDC notionals sit beside every quantity. Under the net position and in the exposure gauge they are marked at the current mark price and move with it; in the tables they are frozen at the execution price of each fill.

---

## Controls

- **Start engine** — starts the state machine. Nothing is sent before this click; on load, only the price feed runs.
- **Halt engine** — suspends the logic. The position stays open, the feed keeps running, no orders are sent.
- **Cover now** — immediate cover at the mark, recorded as `manual cover`.
- **Stop and flatten state** — clears the internal state. Note that it sends no order: any position open at the venue stays open and the dashboard simply forgets about it.

---

## Live execution

Live mode never sends an order directly. The page posts to a signer service you run locally, and that service holds the key and signs. A private key inside a web page is readable by any browser extension or injected script, which is why none of that logic lives in the HTML file.

The page sends:

```json
{ "coin": "HYPE", "action": "short" | "close", "sz": 5,
  "isBuy": false, "reduceOnly": false, "refPx": 84.3410 }
```

The signer must reply with `{"fillPx": <execution price>}` — or `{"error": "…"}`. Without an execution price the page falls back to the mark, which corrupts the break-even from that fill onward.

Use `hyperliquid-python-sdk` or the equivalent Node client: both handle the EIP-712 phantom-agent signing and the `order` action against `/exchange`. Any signer error halts the engine rather than letting it run blind.

---

## Recorded data

Every closed cycle produces one row, exportable as CSV:

`cycle, opened_utc, closed_utc, duration_s, fills, size, notional_usdc, avg_entry, break_even, cover_px, gross_pnl, fees, net_pnl, cycle_high, reason, venue`

`cycle_high` is the highest mark price reached during the cycle — the measure of how far the position went against the book before reverting.

---

## Known limitations

- **History is held in memory.** A page reload clears it. Export the CSV before closing.
- **Simulated fills execute at the mark, with no spread and no slippage.** A real taker crosses the spread, so live results are systematically worse than simulated ones, more so when the book is thin.
- **Funding is not carried into P&L.** A short pinned at the cap for hours pays or receives funding every period. Over a long cycle the gap against the displayed figure can be material.
- **No reconciliation with the venue.** The dashboard keeps its own position accounting. A rejected order, a partial fill, a liquidation or an ADL leaves its state silently diverged from reality.
- **One tab only.** Two live instances would double every order without seeing each other.
- **No margin management.** Nothing verifies that the account can absorb all seven clips.

---

## Strategy risk

This is an adverse-averaging martingale: every add increases exposure while the market moves against the book. The cap bounds size, not loss. A sustained rally fills all 35 HYPE within roughly 3% of upside, and the position then sits there — no stop — for as long as it takes the mark to revert 0.5% through the break-even, paying funding throughout.

The payoff is asymmetric in the wrong direction: many small winning cycles, and rare losing cycles of an entirely different magnitude. Measuring how often the book reaches the cap, over several days of simulated running and through the exported CSV, is the only way to see what that tail actually looks like on this instrument.

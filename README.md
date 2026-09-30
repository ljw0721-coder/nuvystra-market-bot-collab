# Market bot code review

Snapshot: 2026-10-01 KST. Completed code commit: 28015da29ebbfd6df48793c2d60c7b00fa75580f.
Branch: instinct/market-data-bot-2026-10-01.

This is a selected code review package, not a full repository or runnable checkout. Source excerpts preserve their original text and line numbers. Unrelated files, environment configuration, credentials, personal information and fixture-key tests are excluded. The original repository remains private.

## Completed in this snapshot

- Explicit bot conditions: 24h price change (%), 24h USD-notional volume, BTC-relative strength 1h/4h/24h/7d/14d (%). Existing price/OI/funding choices remain.
- RS = ((asset_now / asset_then) / (BTC_now / BTC_then) - 1) * 100. This is a price-ratio measure, not percent-return subtraction and not RSI.
- RS uses persisted 1-minute Hyperliquid marks, including BTC_BENCH. Historical points must be at/before the target and within 90 seconds, with aligned historical asset/benchmark timestamps. Missing or invalid history yields null. No interpolation, candle backfill, cross-provider substitution or invented warm-up data.
- Incoming derived RS values are discarded; the server recalculates them. Batch validation occurs before inserts, and the whole batch is stored before derivation.
- The runner checks indicator freshness and repeats validity, source and threshold checks before normal submission. Changed signals cancel reserved intent; existing reduce-only exit/risk logic is retained.
- The source journal reports 247 local tests passed. This page does not independently reproduce the full test suite or assert live trading acceptance. Mock fixtures are not live exchange evidence.

## Separate work in progress

A new UTC current-day RS field has a local draft under separate review: UTC 00:00 through the latest observation, including the current day, distinct from rolling 24h. It is NOT present in this completed snapshot. Do not treat the five rolling fields as the UTC-day implementation.

## Working alongside the other implementation

James intends to use mobile GPT alongside the current development work. James selected ordinary mobile ChatGPT chat. It can read this public package, but its GitHub app is read-only. James can relay proposed edits here for review and testing. No exact new GPT assignment has been selected. Suggested non-overlapping first task: review this snapshot and report correctness concerns, missing regression cases and proposed changes, with file/line references. Avoid editing the current UTC-day field in parallel. Any later implementation needs an agreed scope and fresh source snapshot.

A public read link does not provide repository write access. This package grants no merge, deployment, wallet connection/signature, real key generation, funded order or spending authority. Current data collection warm-up and deployed service readiness must be verified separately. Long-window values require real retained history. Additional metrics are proposals only, not selected work.

## Reading the excerpts

Supporting imports reference files not included here. Engine and wizard sections are intentionally partial. Do not assume this package is sufficient to build or safely deploy the whole service. If a missing dependency is needed, ask for that specific file or function rather than assume its behavior.


## src/features/nuvystra/model.ts: condition and indicator types

Lines 1-66 from exact commit

```typescript
import Decimal from "decimal.js";

export type BotStatus =
  "draft" | "awaiting_agent" | "active" | "paused" | "stopped" | "error";
export const RELATIVE_STRENGTH_FIELDS = ["rs1h", "rs4h", "rs24h", "rs7d", "rs14d"] as const;
export const CONDITION_FIELDS = ["price", "oi", "funding", "change24h", "volume24h", ...RELATIVE_STRENGTH_FIELDS] as const;
export type Condition = {
  field: (typeof CONDITION_FIELDS)[number];
  op: "gt" | "lt";
  value: string;
};
export interface BotConfig {
  asset: string;
  venue: "hyperliquid" | "dex";
  side: "buy" | "sell";
  order_notional: string;
  entry: Condition;
  exit: Condition | null;
  max_notional_per_order: string;
  max_leverage: number;
  max_deviation_bps: number;
  max_orders_per_min: number;
  cooldown_seconds: number;
}
export interface Bot {
  id: string;
  name: string;
  status: BotStatus;
  config: BotConfig;
  config_version: number;
  applied_version: number;
  hl_subaccount: string | null;
  created_at: string;
  agent_address: string | null;
  agent_label: string | null;
  approved_at: string | null;
  revoked_at: string | null;
  revocation_requested_at: string | null;
  account: { balance: string; pnl: string; updated_at: string } | null;
  stop_level: number;
  error_message: string | null;
}
export interface Indicator {
  asset: string;
  tf: string;
  ts: string;
  payload: {
    venue: "hyperliquid" | "dex";
    source: "hyperliquid" | "geckoterminal";
    price: string;
    oi: string | null;
    funding: string | null;
    change24h: string | null;
    volume24h: string | null;
    rs1h?: string | null;
    rs4h?: string | null;
    rs24h?: string | null;
    rs7d?: string | null;
    rs14d?: string | null;
    candles?: number[][];
    pool?: string;
    network?: string;
    poolCreatedAt?: string;
  };
}
export interface SessionUser {
```

## src/features/nuvystra/model.ts: threshold validation and matching

Lines 107-182 from exact commit

```typescript
}
export function isDecimal(value: unknown, positive = false): value is string {
  return (
    typeof value === "string" &&
    /^-?\d{1,18}(\.\d{1,18})?$/.test(value) &&
    (!positive || new Decimal(value).gt(0))
  );
}
function condition(value: unknown, venue: string): Condition {
  const v = value as Partial<Condition> | null;
  if (
    !v ||
    !CONDITION_FIELDS.some((field) => field === v.field) ||
    !["gt", "lt"].includes(v.op ?? "") ||
    !isDecimal(v.value)
  )
    throw new Error("Enter a valid strategy condition.");
  if (venue === "dex" && ["oi", "funding", ...RELATIVE_STRENGTH_FIELDS].some((field) => field === v.field))
    throw new Error(
      "Open interest, funding and BTC-relative strength are not available for DEX bot conditions.",
    );
  return { field: v.field!, op: v.op!, value: v.value };
}
export function validateBotConfig(value: unknown): BotConfig {
  const v = value as Partial<BotConfig> | null;
  if (
    !v ||
    typeof v.asset !== "string" ||
    !/^[A-Za-z0-9_:.-]{1,96}$/.test(v.asset) ||
    !["hyperliquid", "dex"].includes(v.venue ?? "") ||
    !["buy", "sell"].includes(v.side ?? "")
  )
    throw new Error("Select a supported asset and side.");
  if (
    !isDecimal(v.order_notional, true) ||
    !isDecimal(v.max_notional_per_order, true) ||
    new Decimal(v.order_notional).lt(10) ||
    new Decimal(v.order_notional).gt(v.max_notional_per_order)
  )
    throw new Error(
      "Order notional must be at least 10 USDC and within the order limit.",
    );
  for (const [key, min, max] of [
    ["max_leverage", 1, 20],
    ["max_deviation_bps", 1, 500],
    ["max_orders_per_min", 1, 30],
    ["cooldown_seconds", 5, 86400],
  ] as const) {
    if (!Number.isInteger(v[key]) || v[key]! < min || v[key]! > max)
      throw new Error(`Invalid ${key.replaceAll("_", " ")}.`);
  }
  return {
    asset: v.asset,
    venue: v.venue!,
    side: v.side!,
    order_notional: v.order_notional,
    entry: condition(v.entry, v.venue!),
    exit: v.exit ? condition(v.exit, v.venue!) : null,
    max_notional_per_order: v.max_notional_per_order,
    max_leverage: v.max_leverage!,
    max_deviation_bps: v.max_deviation_bps!,
    max_orders_per_min: v.max_orders_per_min!,
    cooldown_seconds: v.cooldown_seconds!,
  };
}
export function matchesCondition(c: Condition, indicator: Indicator): boolean {
  const value = indicator.payload[c.field];
  return (
    value !== null && value !== undefined &&
    isDecimal(value) &&
    (c.op === "gt"
      ? new Decimal(value).gt(c.value)
      : new Decimal(value).lt(c.value))
  );
}

```

## server/nuvystra/indicators.ts: ingestion and RS calculation

Lines 1-166 from exact commit

```typescript
import {
  isDecimal,
  RELATIVE_STRENGTH_FIELDS,
  type Indicator,
} from "../../src/features/nuvystra/model.ts";
import Decimal from "decimal.js";
import type { Sql } from "./database.ts";
import { Problem } from "./repository.ts";

export function validateIndicators(
  value: unknown,
  now = Date.now(),
): Indicator[] {
  if (!Array.isArray(value) || !value.length || value.length > 200)
    throw new Problem(400, "Provide 1 to 200 indicators.");
  return value.map((v: Indicator) => {
    if (
      !v ||
      !/^[A-Za-z0-9_:.-]{1,96}$/.test(v.asset) ||
      !["1m", "5m", "1h", "1d"].includes(v.tf) ||
      typeof v.ts !== "string" ||
      !Number.isFinite(Date.parse(v.ts)) ||
      Date.parse(v.ts) > now + 10_000 ||
      Date.parse(v.ts) < now - 300_000
    )
      throw new Problem(400, "Invalid or stale indicator.");
    const p = v.payload;
    if (
      !p ||
      !["hyperliquid", "dex"].includes(p.venue) ||
      p.source !== (p.venue === "dex" ? "geckoterminal" : "hyperliquid") ||
      !isDecimal(p.price, true) ||
      ![p.oi, p.funding, p.change24h, p.volume24h].every(
        (x) => x === null || isDecimal(x),
      )
    )
      throw new Problem(400, "Invalid indicator payload.");
    if ([p.oi, p.volume24h].some((x) => x !== null && new Decimal(x).lt(0)))
      throw new Problem(400, "Open interest and volume cannot be negative.");
    if (p.venue === "dex" && (p.oi !== null || p.funding !== null))
      throw new Problem(
        400,
        "DEX assets do not have open interest or funding.",
      );
    if (v.asset === "BTC_BENCH" && p.source !== "hyperliquid")
      throw new Problem(400, "BTC benchmark must use Hyperliquid mark.");
    if (
      p.poolCreatedAt !== undefined &&
      (p.venue !== "dex" ||
        !Number.isFinite(Date.parse(p.poolCreatedAt)) ||
        Date.parse(p.poolCreatedAt) > now)
    )
      throw new Problem(400, "Invalid pool creation date.");
    if (
      p.candles &&
      (!Array.isArray(p.candles) ||
        p.candles.length > 120 ||
        p.candles.some(
          (c) =>
            !Array.isArray(c) ||
            c.length !== 6 ||
            c.some((n) => !Number.isFinite(n)) ||
            c[0]! < 0 || !Number.isInteger(c[0]) || c[0]! * 1000 > now + 10_000 ||
            c.slice(1, 5).some((n) => n <= 0) || c[5]! < 0 ||
            c[2]! < Math.max(c[1]!, c[3]!, c[4]!) ||
            c[3]! > Math.min(c[1]!, c[2]!, c[4]!),
        ))
    )
      throw new Problem(400, "Invalid candles.");
    return {
      asset: v.asset,
      tf: v.tf,
      ts: v.ts,
      payload: {
        venue: p.venue,
        source: p.source,
        price: p.price,
        oi: p.oi,
        funding: p.funding,
        change24h: p.change24h,
        volume24h: p.volume24h,
        ...(p.poolCreatedAt ? { poolCreatedAt: p.poolCreatedAt } : {}),
        ...(p.candles ? { candles: p.candles } : {}),
        ...(typeof p.pool === "string" ? { pool: p.pool.slice(0, 128) } : {}),
        ...(typeof p.network === "string"
          ? { network: p.network.slice(0, 32) }
          : {}),
      },
    };
  });
}
export async function ingest(sql: Sql, indicators: Indicator[]) {
  // Validate the entire batch before any insert, even for direct internal callers.
  const checked = validateIndicators(indicators);
  for (const i of checked)
    await sql.query(
      "INSERT INTO indicators(asset,tf,ts,payload) VALUES ($1,$2,$3,$4) ON CONFLICT(asset,tf,ts) DO UPDATE SET payload=EXCLUDED.payload",
      [i.asset, i.tf, i.ts, JSON.stringify(i.payload)],
    );
  // Insert the whole batch first so BTC_BENCH ordering cannot affect the result.
  for (const i of checked) {
    const relative = await relativeStrength(sql, i);
    await sql.query("UPDATE indicators SET payload=$4 WHERE asset=$1 AND tf=$2 AND ts=$3",
      [i.asset, i.tf, i.ts, JSON.stringify({ ...i.payload, ...relative })]);
  }
}
export async function readIndicators(sql: Sql, params: URLSearchParams) {
  const asset = params.get("asset"),
    tf = params.get("tf") || "1m",
    limit = Number(params.get("limit") || 120);
  if (
    !["1m", "5m", "1h", "1d"].includes(tf) ||
    !Number.isInteger(limit) ||
    limit < 1 ||
    limit > 200 ||
    (asset && !/^[A-Za-z0-9_:.-]{1,96}$/.test(asset))
  )
    throw new Problem(400, "Invalid indicator query.");
  return asset
    ? (
        await sql.query<Indicator>(
          "SELECT asset,tf,ts,payload FROM indicators WHERE asset=$1 AND tf=$2 ORDER BY ts DESC LIMIT $3",
          [asset, tf, limit],
        )
      ).rows
    : (
        await sql.query<Indicator>(
          "SELECT DISTINCT ON(asset) asset,tf,ts,payload FROM indicators WHERE tf=$1 ORDER BY asset,ts DESC LIMIT $2",
          [tf, limit],
        )
      ).rows;
}

const WINDOWS = [3600, 14400, 86400, 604800, 1209600] as const;
// Ratio-of-ratios with persisted same-source HL marks only. No interpolation or warm-up substitutes.
export async function relativeStrength(sql: Sql, current: Indicator) {
  const values: Record<(typeof RELATIVE_STRENGTH_FIELDS)[number], string | null> = {
    rs1h: null, rs4h: null, rs24h: null, rs7d: null, rs14d: null,
  };
  if (current.tf !== "1m" || current.payload.source !== "hyperliquid" ||
      current.payload.venue !== "hyperliquid") return values;
  const time = Date.parse(current.ts);
  async function point(asset: string, at: number): Promise<Indicator | null> {
    const row = (await sql.query<Indicator>(
      `SELECT asset,tf,ts,payload FROM indicators WHERE asset=$1 AND tf='1m'
       AND ts<=$2 AND ts>=$3 ORDER BY ts DESC LIMIT 1`,
      [asset, new Date(at).toISOString(), new Date(at - 90_000).toISOString()],
    )).rows[0];
    if (!row || row.payload.source !== "hyperliquid" || row.payload.venue !== "hyperliquid" ||
        !isDecimal(row.payload.price, true)) return null;
    return row;
  }
  const btc = await point("BTC_BENCH", time);
  if (!btc || !isDecimal(current.payload.price, true)) return values;
  for (let index = 0; index < WINDOWS.length; index++) {
    const at = time - WINDOWS[index]! * 1000;
    const prior = await point(current.asset, at), priorBtc = await point("BTC_BENCH", at);
    if (!prior || !priorBtc || Math.abs(new Date(prior.ts).getTime() - new Date(priorBtc.ts).getTime()) > 90_000) continue;
    const relative = new Decimal(current.payload.price).div(prior.payload.price)
      .div(new Decimal(btc.payload.price).div(priorBtc.payload.price))
      .minus(1).times(100).toFixed(8);
    if (isDecimal(relative)) values[RELATIVE_STRENGTH_FIELDS[index]!] = relative;
  }
  return values;
}

```

## src/features/nuvystra/Wizard.tsx: condition selector

Lines 581-638 from exact commit

```typescript
function ConditionFields({
  value,
  venue,
  onChange,
}: {
  value: Condition;
  venue: BotConfig["venue"];
  onChange: (v: Condition) => void;
}) {
  return (
    <div className="nv-condition">
      <label>
        Indicator
        <select
          value={value.field}
          onChange={(e) =>
            onChange({ ...value, field: e.target.value as Condition["field"] })
          }
        >
          <option value="price">Price</option>
          <option value="change24h">24h price change (%)</option>
          <option value="volume24h">24h volume (USD notional)</option>
          {venue === "hyperliquid" && (
            <>
              <option value="oi">Open interest (base units)</option>
              <option value="funding">Funding rate (decimal)</option>
              <option value="rs1h">BTC-relative strength 1h (%)</option>
              <option value="rs4h">BTC-relative strength 4h (%)</option>
              <option value="rs24h">BTC-relative strength 24h (%)</option>
              <option value="rs7d">BTC-relative strength 7d (%)</option>
              <option value="rs14d">BTC-relative strength 14d (%)</option>
            </>
          )}
        </select>
      </label>
      <label>
        Comparison
        <select
          value={value.op}
          onChange={(e) =>
            onChange({ ...value, op: e.target.value as Condition["op"] })
          }
        >
          <option value="gt">Above</option>
          <option value="lt">Below</option>
        </select>
      </label>
      <label>
        Threshold
        <input
          inputMode="decimal"
          value={value.value}
          onChange={(e) => onChange({ ...value, value: e.target.value })}
        />
      </label>
    </div>
  );
    }
```

## server/nuvystra/engine.ts: initial freshness guard

Lines 231-250 from exact commit

```typescript
        b.applied_version !== b.config_version
      )
        return null;
      const i = (
        await sql.query<Indicator>(
          `SELECT asset,tf,ts,payload FROM indicators WHERE asset=$1 AND tf='1m' ORDER BY ts DESC LIMIT 1`,
          [b.config.asset],
        )
      ).rows[0];
      if (!i || !Number.isFinite(new Date(i.ts).getTime()) ||
          new Date(i.ts).getTime() > Date.now() + 10_000 ||
          Date.now() - new Date(i.ts).getTime() > 90_000) {
        await risk(sql, id, "indicator_stale");
        return null;
      }
      i.ts = new Date(i.ts).toISOString();
      try { validateIndicators([i]); } catch {
        await risk(sql, id, "indicator_invalid");
        return null;
      }
```

## server/nuvystra/engine.ts: pre-submit signal check

Lines 362-381 from exact commit

```typescript
      }
      if (!intent.emergency) {
        const latest = (await sql.query<Indicator>(
          "SELECT asset,tf,ts,payload FROM indicators WHERE asset=$1 AND tf='1m' ORDER BY ts DESC LIMIT 1",
          [b.config.asset],
        )).rows[0];
        if (latest) latest.ts = new Date(latest.ts).toISOString();
        let valid = !!latest;
        try { validateIndicators(latest ? [latest] : []); } catch { valid = false; }
        const condition = order.reduceOnly ? b.config.exit : b.config.entry;
        if (!valid || !latest || Date.now() - Date.parse(latest.ts) > 90_000 ||
            latest.payload.source !== "hyperliquid" || latest.payload.venue !== "hyperliquid" ||
            !condition || !matchesCondition(condition, latest)) {
          await risk(sql, id, "signal_changed_before_send");
          await sql.query("UPDATE orders SET status='cancelled_before_send' WHERE hl_cloid=$1", [order.cloid]);
          return;
        }
      }
      const state = await exchange.account(b.hl_subaccount!),
        market = (await exchange.assets()).find((a) => a.name === order.coin);
```

## tests/nuvystra-market-logic.test.ts: focused regressions

Lines 1-66 from exact commit

```typescript
import test from "node:test";
import assert from "node:assert/strict";
import Decimal from "decimal.js";
import { defaultConfig, validateBotConfig, matchesCondition, RELATIVE_STRENGTH_FIELDS, type Indicator } from "../src/features/nuvystra/model.ts";
import { ingest, validateIndicators, relativeStrength } from "../server/nuvystra/indicators.ts";
import { testDatabase } from "./fixtures/nuvystra-db.ts";
const sample = (asset = "SOL", price = "110", time = Date.now()): Indicator => ({
  asset, tf: "1m", ts: new Date(time).toISOString(),
  payload: { venue: "hyperliquid", source: "hyperliquid", price, oi: "100", funding: "-0.001", change24h: "10", volume24h: "100000" },
});
test("new explicit thresholds support units and fail closed on missing values", () => {
  for (const field of ["change24h", "volume24h", ...RELATIVE_STRENGTH_FIELDS] as const) {
    const c = validateBotConfig({ ...defaultConfig("SOL"), entry: { field, op: "gt", value: "0" } });
    const i = sample();
    assert.equal(matchesCondition(c.entry, i), !RELATIVE_STRENGTH_FIELDS.some((f) => f === field));
    for (const f of RELATIVE_STRENGTH_FIELDS) i.payload[f] = "5";
    assert.equal(matchesCondition(c.entry, i), true);
    assert.equal(matchesCondition({ field, op: "lt", value: "-1" }, i), false);
  }
  assert.throws(() => validateBotConfig({ ...defaultConfig("SOL"), entry: { field: "invented", op: "gt", value: "1" } }));
  assert.throws(() => validateBotConfig({ ...defaultConfig("SOL", "dex"), entry: { field: "rs1h", op: "gt", value: "1" } }));
});
test("RS uses persisted same-source BTC marks for all windows and never substitutes missing history", async (t) => {
  const db = await testDatabase(); t.after(() => db.close());
  const now = Date.now();
  const sol = sample("SOL", "110", now), btc = sample("BTC_BENCH", "105", now);
  // Historical fixtures inserted directly, not a stale ingest accepted as live data.
  for (const window of [3600, 14400, 86400, 604800, 1209600]) {
    for (const i of [sample("SOL", "100", now - window * 1000), sample("BTC_BENCH", "100", now - window * 1000)])
      await db.query("INSERT INTO indicators(asset,tf,ts,payload) VALUES($1,$2,$3,$4)", [i.asset, i.tf, i.ts, JSON.stringify(i.payload)]);
  }
  // Asset comes BEFORE benchmark; derived results must not depend on batch order.
  await ingest(db, [sol, btc]);
  const stored = (await db.query<Indicator>("SELECT * FROM indicators WHERE asset='SOL' ORDER BY ts DESC LIMIT 1")).rows[0]!;
  const expected = new Decimal(110).div(100).div(new Decimal(105).div(100)).minus(1).times(100).toFixed(8);
  for (const field of RELATIVE_STRENGTH_FIELDS) assert.equal(stored.payload[field], expected);
  assert.equal((await relativeStrength(db, sample("NEW", "100", now))).rs1h, null);
  await db.query("DELETE FROM indicators WHERE asset='BTC_BENCH' AND ts<$1", [new Date(now - 90000).toISOString()]);
  for (const value of Object.values(await relativeStrength(db, sol))) assert.equal(value, null);
});
test("untrusted derived RS is stripped at ingest and malformed batches do not partially write", async (t) => {
  const db = await testDatabase(); t.after(() => db.close());
  const i = sample(); i.payload.rs1h = "999";
  await ingest(db, [i]);
  const stored = (await db.query<Indicator>("SELECT * FROM indicators WHERE asset='SOL'")).rows[0]!;
  assert.equal(stored.payload.rs1h, null);
  const bad = sample("BAD"); bad.payload.volume24h = "-1";
  await assert.rejects(ingest(db, [sample("OK"), bad]));
  assert.equal((await db.query("SELECT * FROM indicators WHERE asset='OK'")).rows.length, 0);
});
test("future/stale times, negative OI/volume, malformed candles and missing numbers are rejected", () => {
  for (const time of [Date.now() + 11000, Date.now() - 301000]) assert.throws(() => validateIndicators([sample("SOL", "100", time)]));
  for (const field of ["oi", "volume24h"] as const) { const i = sample(); i.payload[field] = "-1"; assert.throws(() => validateIndicators([i])); }
  const i = sample(); i.payload.candles = [[Math.floor(Date.now()/1000),100,99,90,110,1]];
  assert.throws(() => validateIndicators([i]));
  i.payload.candles = [[Math.floor(Date.now()/1000),100,110,90,105,1]];
  assert.equal(validateIndicators([i]).length, 1);
});

```

## Export layout

All relevant source excerpts are embedded below their original paths above. This fresh-history repository intentionally does not reproduce the original repository tree or dependency setup. It is a review and proposed-change workspace, not a buildable full application. No license has been added.

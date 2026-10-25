# XPeng Monitor

[English](README.md) | [中文](README.zh.md)

A comprehensive XPeng (小鹏汽车) data monitoring toolkit, built as a [Claude Code](https://claude.com/claude-code) Skill. It aggregates XPeng metrics across markets and dimensions through dedicated Node.js scripts — currently covering **China delivery lead times** and **Europe BEV daily registration volumes**, with more data sources to be added.

## Currently Supported

| Module | Market | Data | Source |
|--------|--------|------|--------|
| Delivery lead time | 🇨🇳 China | Wait time in **weeks** for each model / trim version | XPeng official store API (`store.xiaopeng.com`) |
| Daily registrations | 🇪🇺 Europe | BEV daily registration/delivery volume, last 12 months with MoM comparison | [eu-evs.com](https://eu-evs.com) |

> **Scope note:** the Europe module only covers the handful of European countries that publish daily-updated registration data — it is **not** XPeng's total European sales.

## How It Works

- All HTTP requests go through the Node.js scripts in [`scripts/`](scripts/) — anti-bot handling, login state and cookie management are encapsulated inside the scripts. Do not fetch the data sources with curl/Python/etc. directly.
- Scripts depend only on Node.js built-in modules (`https`, `fs`, `path`) — **no `npm install` required**, just Node.js.
- All signals (data, `LOGIN_REQUIRED`, `ERROR|...`) are printed to **stdout**, so callers only need to inspect stdout.
- Transient network errors (socket hang up, ECONNRESET, timeouts) are retried automatically inside the scripts with exponential backoff.

## Install as a Claude Code Skill

Copy this repository into your Claude Code skills directory:

```bash
git clone https://github.com/<your-username>/xpeng-monitor.git ~/.claude/skills/xpeng-monitor
```

Then in Claude Code, just ask naturally, e.g.:

- "小鹏 G6 交付周期多久？" / "How long is the XPeng G6 delivery wait?"
- "小鹏欧洲销量怎么样？" / "How are XPeng's Europe sales?"
- Or invoke directly with `/xpeng-monitor`

Claude will pick the right module based on your question and present the results as markdown tables.

## Module 1 · China Delivery Lead Time

### Scripts

| Script | Purpose | Usage |
|--------|---------|-------|
| `fetch_car_series.js` | List all car series | `node scripts/fetch_car_series.js` |
| `fetch_car_versions.js` | List trim versions of a series | `node scripts/fetch_car_versions.js <carSeriesCode>` |
| `fetch_delivery_period.js` | Query delivery period per version | see below |

`fetch_delivery_period.js` automatically picks the right API strategy based on the series code:

| Mode | Usage | Applies to | Output |
|------|-------|-----------|--------|
| Case 1 single | `node scripts/fetch_delivery_period.js <carVersionCode> <carSeriesCode>` | GX, M03, P7 series | `OK\|min\|max\|carName` |
| Case 2 single | `node scripts/fetch_delivery_period.js <carVersionCode> <carSeriesCode>` | other models | `OK\|min\|max\|carName` |
| Case 2 batch (recommended) | `node scripts/fetch_delivery_period.js <carSeriesCode> --all` | other models | `versionCode\|OK\|min\|max\|carName` per line |

### Example

```bash
$ node scripts/fetch_car_series.js
1|G6|G6
2|GX|GX
3|M03_2026|M03
...

$ node scripts/fetch_delivery_period.js GX --all
1585MAX|OK|4|6|GX 1585 四驱 Max 仰望绿
665MAX|OK|6|8|GX 665 Max 星云白
```

Result presented as:

| Model | Trim | Vehicle name | Min lead time | Max lead time |
|-------|------|--------------|---------------|---------------|
| GX | 1585 四驱 Max | GX 1585 四驱 Max 仰望绿 | 4 weeks | 6 weeks |
| GX | 665 Max | GX 665 Max 星云白 | 6 weeks | 8 weeks |

Notes:

- The unit is **weeks** — never convert to days.
- Figures follow the **official configurator page** caliber (`listDeliveryPeriodByVersion`), with automatic fallback to the older SKU snapshot when that endpoint fails.

## Module 2 · Europe BEV Daily Registrations

### Script

```bash
node scripts/xpeng_eu_daily.js [year] [--report] [--email <email> --password <password>]
```

- `year` — optional, defaults to the current year
- `--report` — outputs 4 pre-formatted markdown table modules (default entry point):
  - ① Month-over-month comparison (fair same-range 1–D comparison)
  - ② Full-month totals
  - ③ Per-model breakdown
  - ④ Weekday-caliber comparison (current vs previous month, day 1 ~ T-2, aligned by nth weekday)
- Without `--report`, prints a single-line compact JSON with `daily` / `monthly_full` / `monthly_partial` layers for flexible analysis (long-term trends, single-day anomalies, weekday vs weekend, etc.)
- Cross-year ranges are handled automatically (a 12-month window typically spans two calendar years)

### Login

eu-evs.com redirects unauthenticated/bot-flagged requests (302 → `/bots`, `/login`, `/onlyNamed`, …). The script treats any 302 on a data page as "login required" and prints `LOGIN_REQUIRED` to stdout (exit code 2).

- Pass `--email` / `--password` to log in automatically (CSRF flow handled internally)
- On success the session cookie is saved to `scripts/.eu-session.json` and reused afterwards — credentials are only needed once
- Register at <https://eu-evs.com/register> if you don't have an account; consider using a dedicated password rather than your usual one

### Output signals

| stdout signal | Meaning | Action |
|---------------|---------|--------|
| JSON / markdown report | Success | Display as-is |
| `LOGIN_REQUIRED` | 302 blocked, needs login | Provide `--email` / `--password` and rerun |
| `ERROR\|LOGIN_FAILED` | Wrong credentials | Re-enter credentials |
| `ERROR\|HTTP_<code>_year_<y>` | Unexpected status code | 5xx → retry later; 4xx → request issue |
| `ERROR\|PARSE_FAILED_year_<y>` | Page structure changed | Data source layout may have changed |
| `ERROR\|no_data_for_year_<y>` | No data for the year | Try another year |
| Other `ERROR\|<msg>` | Unlisted error | Report the message as-is |

## Project Layout

```
xpeng-monitor/
├── SKILL.md              # Skill definition (triggers, workflows, constraints)
├── README.md             # This file
├── README.zh.md          # 中文文档
└── scripts/
    ├── fetch_car_series.js        # China: list all car series
    ├── fetch_car_versions.js      # China: list trim versions
    ├── fetch_delivery_period.js   # China: delivery lead time per version
    ├── xpeng_eu_daily.js          # Europe: 12-month daily registration data
    └── diag_*.js                  # Diagnostic helpers
```

## Disclaimer

This project is for personal research/monitoring only. Data belongs to XPeng and eu-evs.com respectively; please follow the terms of service of the data sources. Figures may differ from official monthly announcements due to caliber differences.

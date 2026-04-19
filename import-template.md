# BullionView CSV import guide

BullionView imports only its **native CSV format** — the same format
you get out of **Settings → Export collection**. To import from a
spreadsheet or another app, restructure your data into this format
first. The [template file](./import-template.csv) is the canonical
shape: copy it, fill in your rows, save as CSV, then pick it in
**Settings → Import collection**.

## File structure

```
# BullionView Export v1 | YYYY-MM-DD | N holdings
subtype,holding_type,metal,weight_troy_oz,year,purchase_date,purchase_price_amount,...
<one row per holding>
```

- **Line 1** is the metadata header. Must start with
  `# BullionView Export v1 |`. The date and count are informational —
  the importer doesn't validate them.
- **Line 2** is the column header — 20 names in the exact order shown
  in the template. Don't rename or reorder.
- **Line 3+** is one row per holding.

Every row has **all 20 columns separated by commas** — leave unused
columns **empty** between the commas, don't omit them. The only
fields you actually have to fill are the required ones below.

## Required fields (6)

Import fails for any row missing one of these.

| Column | Notes |
| --- | --- |
| `subtype` | BullionView spec ID. Must match a known spec exactly — see the [valid-subtypes list](#valid-subtypes) below. |
| `year` | Mint year (integer). |
| `purchase_date` | ISO 8601 `YYYY-MM-DD`. |
| `purchase_price_amount` | Decimal in the original currency. |
| `purchase_price_currency` | ISO 4217 code — `USD`, `EUR`, `CZK`, `GBP`, etc. |
| `status` | `active` or `sold`. |

### Additionally required when `status` is `sold`

| Column | Notes |
| --- | --- |
| `sold_date` | ISO `YYYY-MM-DD`. |
| `sale_price_amount` | Decimal. |
| `sale_price_currency` | ISO 4217. |

## Optional — resolved from the spec

Leave these empty; BullionView reads them from the `subtype` spec.
If you fill them in *and they conflict with the spec*, the spec
wins and the preview shows a note explaining the override.

| Column | What fills in for you |
| --- | --- |
| `holding_type` | `coin` / `bar` — derived from the subtype's spec. |
| `metal` | Derived from the subtype's spec. |
| `weight_troy_oz` | Derived from the subtype's spec. |

## Optional — resolved by the backfill service

These get resolved automatically on the next online startup, from
ECB historical FX and the spot history archive.

| Column | What fills in for you |
| --- | --- |
| `purchase_fx_rate_to_usd` | FX rate on the purchase date. **See [FX limitation](#historical-fx-limitation) below for pre-1999 purchases.** |
| `spot_price_at_entry_usd` | Daily spot for the purchase date. |
| `spot_entry_precision` | Defaults to `unavailable`; upgraded after resolution. |
| `sale_fx_rate_to_usd` | Same as purchase — backfilled by date. |

## Optional — user convenience

Fill these if you know them; skip otherwise.

| Column | Notes |
| --- | --- |
| `vault_name` | Vault by **name** (case-insensitive). Missing vaults are created automatically. |
| `condition` | One of `poor`, `fair`, `fine`, `veryFine`, `extraFine`, `aboutUnc`, `uncirculated`, `brilliantUnc`, `proof`. |
| `notes` | Free text. Wrap in double quotes and double any inner `"` if the note contains `,` or `"` (RFC 4180). |
| `batch_id` | UUID. Use the same value across rows to keep batched holdings grouped. |

## Historical FX limitation

FX rates are automatically resolved for purchases from **1999
onwards**. For older purchases in non-USD currencies, the FX rate
cannot be determined automatically — these rows will import with a
pending FX rate and their values may show as em-dash until you
provide a rate manually. Pre-1999 purchases in USD are unaffected,
since no FX conversion is needed.

## Minimal example row

Six required columns filled, every optional left blank — perfectly
valid. Spec-resolved fields (`holding_type`, `metal`,
`weight_troy_oz`) come from the subtype; backfill-resolved fields
fill in on the next online startup.

```
seed-gold-maple-1oz,,,,2024,2024-01-15,2000.00,USD,,,,,,,active,,,,,
```

## What your data becomes

Here's a concrete before/after so you know exactly how a CSV row
maps to a BullionView holding.

### Active holding

**CSV row** (only the 6 required columns filled):

```
seed-gold-maple-1oz,,,,2023,2024-03-15,1850.00,EUR,,,,,,,active,,,,,
```

**Becomes in BullionView:**

- **Family:** Canadian Gold Maple Leaf
- **Metal:** Gold (resolved from spec)
- **Weight:** 1.000 oz (resolved from spec)
- **Year:** 2023
- **Purchase date:** 15 March 2024
- **Purchase price:** EUR 1,850.00
- **Purchase FX rate:** auto-resolved from ECB data
- **Spot at entry:** auto-resolved from historical spot data
- **Premium over spot:** calculated automatically once FX + spot resolve
- **Vault:** *(none — not specified)*
- **Status:** Active

### Sold holding

**CSV row** (6 required for active + 3 more for sold = 9 filled):

```
seed-gold-saint-gaudens-double-eagle,,,,1924,2022-11-03,2450.00,USD,,,,,,,sold,2025-03-12,2680.00,USD,,
```

**Becomes in BullionView:**

- **Family:** Saint-Gaudens Double Eagle
- **Metal:** Gold (resolved from spec)
- **Weight:** 0.9675 oz (resolved from spec)
- **Year:** 1924
- **Purchase date:** 3 November 2022
- **Purchase price:** USD 2,450.00
- **Purchase FX rate:** auto-resolved (1.0 — USD)
- **Sale date:** 12 March 2025
- **Sale price:** USD 2,680.00
- **Sale FX rate:** auto-resolved (1.0 — USD)
- **Realised P&L:** +USD 230.00 (calculated automatically)
- **Status:** Sold
- **Numismatic banner:** shown (historical coin — spot-based
  valuation flag applies)

## Valid subtypes

`subtype` must match a BullionView spec ID exactly. The **live list**
lives at
[coins.json](https://bullionvault70-cell.github.io/bullionvault/coins.json)
and
[bars.json](https://bullionvault70-cell.github.io/bullionvault/bars.json)
— copy the `id` field. Common examples:

### Gold coins (bullion)

- `seed-gold-eagle-1oz`, `seed-gold-eagle-0.5oz`, `seed-gold-eagle-0.25oz`, `seed-gold-eagle-0.1oz`
- `seed-gold-maple-1oz`, `seed-gold-maple-0.5oz`, `seed-gold-maple-0.25oz`, `seed-gold-maple-0.1oz`
- `seed-gold-philharmoniker-1oz`, `seed-gold-philharmoniker-0.5oz`, `seed-gold-philharmoniker-0.25oz`, `seed-gold-philharmoniker-0.1oz`
- `seed-gold-britannia-1oz`, `seed-gold-britannia-0.5oz`, `seed-gold-britannia-0.25oz`, `seed-gold-britannia-0.1oz`
- `seed-gold-krugerrand-1oz`, `seed-gold-krugerrand-0.5oz`, `seed-gold-krugerrand-0.25oz`, `seed-gold-krugerrand-0.1oz`
- `seed-gold-kangaroo-1oz`, `seed-gold-kangaroo-0.5oz`, `seed-gold-kangaroo-0.25oz`, `seed-gold-kangaroo-0.1oz`
- `seed-gold-libertad-1oz` … `-0.05oz`
- `seed-gold-panda-30g`, `-15g`, `-8g`, `-3g`, `-1g`
- `seed-gold-buffalo-1oz`
- `seed-gold-sovereign-full`, `seed-gold-sovereign-half`

### Gold coins (historical / numismatic)

- `seed-gold-saint-gaudens-double-eagle`
- `seed-gold-liberty-head-double-eagle`, `-eagle`, `-half-eagle`, `-quarter-eagle`
- `seed-gold-indian-head-eagle`, `-half-eagle`, `-quarter-eagle`
- `seed-gold-dollar`
- `seed-gold-french-20-franc-napoleon`
- `seed-gold-austrian-100-corona`, `-20-corona`, `-10-corona`
- `seed-gold-austrian-4-dukat`, `-1-dukat`
- `seed-gold-czechoslovak-1-dukat`, `-2-dukat`, `-5-dukat`, `-10-dukat`
- `seed-gold-hungarian-100-korona`, `-20-korona`, `-10-korona`
- `seed-gold-hungarian-8-forint`, `-4-forint`
- `seed-gold-swiss-20-franc-vreneli`
- `seed-gold-german-20-mark`
- `seed-gold-dutch-10-gulden`
- `seed-gold-belgian-20-franc`

### Silver coins

- `seed-silver-eagle-1oz`
- `seed-silver-maple-1oz`
- `seed-silver-philharmoniker-1oz`
- `seed-silver-britannia-1oz`
- `seed-silver-kangaroo-1oz`
- `seed-silver-libertad-1oz`, `seed-silver-libertad-2oz`

### Platinum coins

- `seed-platinum-eagle-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-platinum-maple-1oz`
- `seed-platinum-kangaroo-1oz`

### Bars

Bar subtypes follow the pattern `seed-<metal>-bar-<refinery>-<size>`,
e.g. `seed-gold-bar-pamp-fortuna-100g`,
`seed-silver-bar-argor-500g`. Consult
[bars.json](https://bullionvault70-cell.github.io/bullionvault/bars.json)
for the full list.

### If your coin or bar isn't listed

If your coin or bar is not listed among supported types, contact us
at **bullionvault70@gmail.com** or use the **"Request a coin/bar"**
form in the app. We'll add it to the spec database and bump the
remote version so it becomes available to everyone on the next sync.

## Common pitfalls

- **Wrong metadata header** → rejected as non-native.
- **Column reordered or renamed** → rejected as non-native.
- **Unknown subtype** → row flagged with "unknown type — will be
  skipped". Fix the subtype to the exact spec ID and re-import.
- **Wrong date format** → row flagged (`2024/05/14` is invalid; use
  `2024-05-14`).
- **Decimals with comma separator** → row flagged (use a dot:
  `1.0`, not `1,0`). CSV already treats comma as the delimiter.
- **Notes with commas or quotes** → wrap the cell in `"…"` and
  double any inner `"`. The exporter handles this for you; hand-
  editing in a spreadsheet handles it for you too, as long as you
  save as "CSV (UTF-8)".
- **Conflicting `holding_type` / `metal` / `weight_troy_oz`** →
  spec wins; preview shows an informational note and the row still
  imports.

## Tips for spreadsheet users

1. Open the [template](./import-template.csv) in your spreadsheet app.
2. Delete the example rows and add your own — one holding per row.
   Leave optional cells empty.
3. Save as **CSV (UTF-8)**. Excel's "CSV (Comma delimited)" works,
   but "CSV (Macintosh)" and "CSV (MS-DOS)" can break notes with
   embedded newlines; prefer UTF-8.
4. Open BullionView → **Settings → Import collection** and pick
   your file.
5. Review the preview. Warnings list rows that need attention
   (those won't import); notes list informational hints (those still
   import). Tick *Skip rows with errors* to import the rest.

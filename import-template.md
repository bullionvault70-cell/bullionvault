# BullionView CSV import guide

BullionView imports only its **native CSV format** — the same format
you get out of **Settings → Export collection**. To import from a
spreadsheet or another app, restructure your data into this format
first.

Two ready-to-use templates:

- [**Excel template (.xlsx)**](./BullionView_Import_Template.xlsx)
  — single sheet with all three sections, dropdown validation on the
  enum columns, and a live formula in the header that auto-counts your
  rows. Recommended for hand-editing.
- [**CSV template (.csv)**](./BullionView_Import_Template.csv) — the
  raw text shape the importer actually reads. Open in a text editor
  or spreadsheet, fill in your rows, save as CSV (UTF-8), then pick
  it in **Settings → Import collection**.

The CSV format was upgraded to **v2** to round-trip user-defined specs
and full vault data. v1 files still import, but new exports use v2.

## File structure

A v2 file is **stacked sections** inside a single .csv:

```
# BullionView Export v2 | YYYY-MM-DD | N holdings | M specs | V vaults

# USER_SPECS
id,holding_type,subtype,metal,weight_troy_oz,weight_g,purity,…
<one row per custom spec>

# VAULTS
name,vault_type,notes
<one row per vault>

# SPREAD_DEFAULTS
metal,band,sell_spread
<one row per set cell of your sell-spread table>

# SPEC_RATE_OVERRIDES
spec_id,premium_over_spot,sell_spread_over_spot
<one row per per-spec premium / sell-spread override>

# HOLDINGS
subtype,holding_type,metal,weight_troy_oz,year,purchase_date,…
<one row per holding>
```

- **Line 1** is the metadata header. Must start with
  `# BullionView Export v2 |`. The date is informational. The
  `| M specs` / `| V vaults` segments are present only when those
  sections exist.
- **`# USER_SPECS`** is optional — include it if you reference custom
  specs in your holdings.
- **`# VAULTS`** is optional — include it to set vault `type` and
  `notes`. Without it, vault names from the holdings section get
  auto-created with `vault_type = homeStorage` and empty notes.
- **`# SPREAD_DEFAULTS`** is optional — a BullionView export includes
  it so your sell-spread defaults table round-trips. You don't need
  to hand-author it; if present it **overwrites** your current table
  on import, if absent your table is left untouched. Columns:
  `metal`, `band` (one of `100g+`, `50g`, `1oz`, `1/2oz`, `1/4oz`,
  `1/10oz`, `under1/10oz`), `sell_spread` (decimal).
- **`# SPEC_RATE_OVERRIDES`** is optional — your per-spec premium /
  sell-spread overrides. Imported AFTER `# USER_SPECS` so any
  `user-*` ids it references already exist. Columns: `spec_id` (the
  catalogue id or `user-<uuid>`), `premium_over_spot` (decimal,
  optional), `sell_spread_over_spot` (decimal, optional). Rows
  where both rates are empty are silently skipped — the cascade
  treats them as "no override" anyway.
- **`# HOLDINGS`** is required — same 20 columns as before, in the
  same order. Don't rename or reorder.

Excel "Save As → CSV" pads every row out to the widest section's
column count — that's expected; the importer tolerates the
trailing commas.

## USER_SPECS section

Lets you ship custom specs (no-name bullion, generic bars, anything
not in the catalogue) alongside your holdings. A holdings row can
reference a custom spec by putting its `id` in the `subtype` column.

**Important:** when a `user-…` id already exists locally, the
incoming row's attributes overwrite the local row. The importer
treats the CSV as the source of truth.

| Column | Required | Notes |
| --- | --- | --- |
| `id` | ✓ | `user-<uuid>`. Generate a fresh UUID for each new spec; reuse the same id across exports to mean "the same spec". |
| `holding_type` | ✓ | `coin` or `bar`. Picks which Drift table the row maps to. |
| `subtype` | ✓ | Human-readable name shown in the picker, e.g. "Local Shop Round". Also drives the default-icon letter. |
| `metal` | ✓ | `gold` / `silver` / `platinum` / `palladium`. |
| `weight_troy_oz` | ✓ | **Pure metal content** in troy ounces. A 22 kt "1 oz" gold coin is `1.0`, not the gross `1.09`. Purity is not multiplied at valuation time. |
| `weight_g` | ✓ | Same weight in grams. Both columns are stored so the editor can echo the unit you originally chose. |
| `purity` | ✓ | Decimal in (0, 1]. e.g. `0.999`, `0.9999`. |
| `diameter_mm` | coin | Outer diameter for coins. Empty for bars. |
| `length_mm` | bar | Long edge for bars. Empty for coins. |
| `width_mm` | bar | Short edge for bars. Empty for coins. |
| `thickness_mm` | optional | Stack-height metric uses this when set. |
| `refinery_or_country` | ✓ | Refinery name for bars, issuing country for coins. |
| `series` | optional | Series name for coins (e.g. "Wildlife Series"). |
| `display_in_grams` | ✓ | `true` or `false`. Controls how the spec's weight reads back in the UI. |
| `premium_over_spot` | optional | Decimal (e.g. `0.15` for a 15 % premium). The markup over spot on the *buy* side — used for the holding's current value. Empty for no premium. |
| `sell_spread_over_spot` | optional | Decimal (e.g. `0.04` for a 4 % spread). The dealer's discount on the *sell* side — used for the "Sell value (est.)" figure. Empty falls back to a metal × weight default. |

## VAULTS section

| Column | Required | Notes |
| --- | --- | --- |
| `name` | ✓ | Join key against `vault_name` in the holdings section. Names are unique per user. |
| `vault_type` | ✓ | One of `safe`, `bankVault`, `storageUnit`, `homeStorage`, `other`. Defaults to `homeStorage` on import if the value is missing or unrecognised. |
| `notes` | optional | Free text. Wrap in `"…"` if it contains a comma. |

Vault rows enrich the holding-side `vault_name` reference with type
+ notes. Without a VAULTS section, vault names from the holdings
section still auto-create — just without type/notes.

## HOLDINGS section

Unchanged from v1 — same 20 columns, same semantics.

### Required fields (6)

Import fails for any row missing one of these.

| Column | Notes |
| --- | --- |
| `subtype` | A catalogue spec id (`seed-…`) or a `user-…` id you also defined in the USER_SPECS section. |
| `year` | Mint year (integer). |
| `purchase_date` | ISO 8601 `YYYY-MM-DD`. |
| `purchase_price_amount` | Decimal in the original currency. |
| `purchase_price_currency` | ISO 4217 — `USD`, `EUR`, `CZK`, `GBP`, etc. |
| `status` | `active` or `sold`. |

### Additionally required when `status` is `sold`

| Column | Notes |
| --- | --- |
| `sold_date` | ISO `YYYY-MM-DD`. |
| `sale_price_amount` | Decimal. |
| `sale_price_currency` | ISO 4217. |

### Optional — resolved from the spec

Leave empty; BullionView reads them from the `subtype` spec. If you
fill them in *and they conflict with the spec*, the spec wins and
the preview shows a note.

| Column | What fills in for you |
| --- | --- |
| `holding_type` | `coin` / `bar` — derived from the subtype's spec. |
| `metal` | Derived from the subtype's spec. |
| `weight_troy_oz` | Derived from the subtype's spec. |

### Optional — resolved by the backfill service

Auto-resolved on the next online startup from ECB historical FX and
the spot history archive.

| Column | What fills in for you |
| --- | --- |
| `purchase_fx_rate_to_usd` | FX rate on the purchase date. See **historical FX limitation** below. |
| `spot_price_at_entry_usd` | Daily spot for the purchase date. |
| `spot_entry_precision` | Defaults to `unavailable`; upgraded after resolution. |
| `sale_fx_rate_to_usd` | Same as purchase — backfilled by date. |

### Optional — user convenience

| Column | Notes |
| --- | --- |
| `vault_name` | Vault by name (case-insensitive). Missing vaults are auto-created. Type + notes come from the VAULTS section if present. |
| `condition` | One of `poor`, `fair`, `fine`, `veryFine`, `extraFine`, `aboutUnc`, `uncirculated`, `brilliantUnc`, `proof`. |
| `notes` | Free text. Wrap in double quotes and double any inner `"` if it contains `,` or `"`. |
| `batch_id` | UUID. Use the same value across rows to keep batched holdings grouped. |

## Historical FX limitation

FX rates are automatically resolved for purchases from **1999
onwards**. For older purchases in non-USD currencies, the FX rate
cannot be determined automatically — the row imports with a pending
FX rate and its values may show as em-dash until you provide a rate
manually. Pre-1999 purchases in USD are unaffected.

## Minimal example

Catalogue-only holding — six required columns filled, every optional
empty. No USER_SPECS or VAULTS sections needed:

```
# BullionView Export v2 | 2026-05-13 | 1 holdings

# HOLDINGS
subtype,holding_type,metal,weight_troy_oz,year,purchase_date,purchase_price_amount,purchase_price_currency,purchase_fx_rate_to_usd,spot_price_at_entry_usd,spot_entry_precision,vault_name,condition,notes,status,sold_date,sale_price_amount,sale_price_currency,sale_fx_rate_to_usd,batch_id
seed-gold-maple-1oz,,,,2024,2024-01-15,2000.00,USD,,,,,,,active,,,,,
```

The `# HOLDINGS` marker is required only when other sections precede
it. When holdings are the only section, the v1 single-table shape
(no marker) is also accepted.

## What your data becomes

### Active catalogue holding

**CSV row:**

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
- **Status:** Active

### Active holding referencing a user-defined spec

**CSV (just the relevant rows):**

```
# USER_SPECS
id,holding_type,subtype,metal,weight_troy_oz,weight_g,purity,diameter_mm,length_mm,width_mm,thickness_mm,refinery_or_country,series,display_in_grams,premium_over_spot,sell_spread_over_spot
user-abc-123,coin,Local Shop Round,silver,1.0,31.1035,0.999,38.6,,,2.98,Czech Republic,,false,,

# HOLDINGS
…
user-abc-123,,,,2024,2024-06-01,30.00,USD,1.0,28.00,precise,Home safe,,,active,,,,,
```

**Becomes in BullionView:**

- **Family:** Local Shop Round (user-defined, gets a default icon)
- **Metal:** Silver (resolved from the user spec)
- **Weight:** 1.000 oz
- **Year:** 2024
- **Purchase price:** USD 30.00
- **Vault:** Home safe (created/enriched from the VAULTS section)
- **Status:** Active

### Sold catalogue holding

**CSV row:**

```
seed-gold-saint-gaudens-double-eagle,,,,1924,2022-11-03,2450.00,USD,,,,,,,sold,2025-03-12,2680.00,USD,,
```

**Becomes:**

- **Family:** Saint-Gaudens Double Eagle
- **Weight:** 0.9675 oz (resolved from spec)
- **Purchase date:** 3 November 2022 — USD 2,450.00
- **Sale date:** 12 March 2025 — USD 2,680.00
- **Realised P&L:** +USD 230.00
- **Status:** Sold
- **Numismatic banner:** shown (historical coin)

## Valid subtypes

For catalogue holdings, `subtype` must match a BullionView spec ID
exactly. The live list lives at
[coins.json](https://bullionvault70-cell.github.io/bullionvault/specs/coins.json)
and
[bars.json](https://bullionvault70-cell.github.io/bullionvault/specs/bars.json)
— copy the `id` field. Common examples:

### Gold coins (bullion)

- `seed-gold-eagle-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-gold-maple-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-gold-philharmoniker-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-gold-britannia-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-gold-krugerrand-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-gold-kangaroo-1oz`, `-0.5oz`, `-0.25oz`, `-0.1oz`
- `seed-gold-libertad-1oz` … `-0.05oz`
- `seed-gold-panda-30g`, `-15g`, `-8g`, `-3g`, `-1g`
- `seed-gold-buffalo-1oz`
- `seed-gold-sovereign-full`, `-half`

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
[bars.json](https://bullionvault70-cell.github.io/bullionvault/specs/bars.json)
for the full list.

### If your coin or bar isn't listed

Two options:

1. **Use a USER_SPECS row.** Add a custom spec with a fresh
   `user-<uuid>` id and reference it from your holdings — see the
   USER_SPECS section above.
2. **Request a catalogue entry.** Contact
   **bullionvault70@gmail.com** or use the in-app
   *"Request a coin/bar"* form. We'll add it to the spec database
   and bump the remote version so it becomes available to everyone
   on the next sync.

## Common pitfalls

- **Wrong metadata header** → rejected as non-native.
- **Column reordered or renamed** → rejected as non-native.
- **Unknown subtype** with no matching USER_SPECS row → row flagged
  with "unknown type — will be skipped".
- **Wrong date format** → row flagged (`2024/05/14` is invalid; use
  `2024-05-14`).
- **Decimals with comma separator** → row flagged (use a dot:
  `1.0`, not `1,0`). CSV already treats comma as the delimiter.
- **Notes with commas or quotes** → wrap the cell in `"…"` and
  double any inner `"`. The exporter handles this for you;
  spreadsheet apps handle it for you if you save as "CSV (UTF-8)".
- **Conflicting `holding_type` / `metal` / `weight_troy_oz`** →
  spec wins; preview shows an informational note and the row still
  imports.
- **USER_SPECS row with the same id as an existing local spec** →
  incoming row's attributes overwrite the local row. If that's not
  what you want, delete the row from your CSV before importing.

## Tips for spreadsheet users

1. Open the [Excel template](./BullionView_Import_Template.xlsx) (or
   the [.csv version](./BullionView_Import_Template.csv)) in your
   spreadsheet app.
2. The Excel template has dropdowns on the enum columns
   (`metal`, `holding_type`, `vault_type`, `condition`, `status`,
   etc.) — use them to avoid typos. The metadata header at the top
   auto-counts your rows when you save.
3. Edit your rows in each section. Use the existing placeholder
   row as a template — copy-paste it down to keep the dropdown
   validation. Leave optional cells empty.
4. Save as **CSV (UTF-8)**. Excel's "CSV (Comma delimited)" works,
   but "CSV (Macintosh)" / "CSV (MS-DOS)" can mangle notes with
   embedded newlines; prefer UTF-8.
5. Open BullionView → **Settings → Import collection** and pick
   your file.
6. Review the preview. **Warnings** list rows that need attention
   (those won't import); **notes** list informational hints (those
   still import). Tick *Skip rows with errors* to import the rest.

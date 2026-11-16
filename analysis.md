# fashion_sales.csv — dataset analysis

A plain-language guide to what this file contains, how it is put together, which
columns drive which, and what is odd about it.

Written after a full pass over all 240 rows: blank-value scan, type checks,
duplicate checks, cross-column arithmetic checks, distribution checks, and
correlation analysis. Every number quoted below was computed from the file.

---

## 1. The short version

`fashion_sales.csv` is a **two-year monthly sales summary for a clothing
retailer**, broken out by **product category** and **sales channel**. It holds
**240 rows and 14 columns**, covers **January 2024 through December 2025**, and
totals **$13.22 million of revenue** from **202,500 units** sold.

It does not contain individual transactions, customers, products, costs, or
profit. Each row is already an aggregate. That single fact determines almost
everything about how the file should be used.

| Headline | Value |
|---|---|
| Rows (excluding header) | 240 |
| Columns | 14 |
| Period | 2024-01 → 2025-12 (24 months) |
| Categories | 5 (Dresses, Denim, Outerwear, Footwear, Accessories) |
| Channels | 2 (Online, Store) |
| Total revenue | $13,220,481.53 |
| Total units sold | 202,500 |
| Total advertising spend | $2,337,530 |
| Missing values | none |
| Duplicate rows | none |
| File size / format | 18 KB, comma-separated, CRLF line endings, no quoted fields |

---

## 2. How the data is organized

### Grain: what one row means

One row = **one product category, in one sales channel, in one calendar month**.

For example:

```
2024-01,Dresses,Online,...,609 units,...,33074.79
```

means "during January 2024, Dresses sold through the Online channel moved 609
units and generated $33,074.79". It is a summary of many unknown transactions,
not a single sale.

### The grid is complete and balanced

The file is a **full factorial panel**: every combination of the 24 months, 5
categories, and 2 channels appears exactly once.

```
24 months × 5 categories × 2 channels = 240 rows
```

Verified consequences:

- every month has exactly 10 rows (5 categories × 2 channels),
- every month–category pair has exactly 2 rows (Online and Store),
- no combination is missing, and none is duplicated.

This matters for analysis: you never need to worry about uneven coverage,
seasonal gaps, or a category that launched mid-period. Every comparison across
time, category, or channel is apples-to-apples.

### How to picture it

Think of a cube with three axes — month, category, channel — with exactly one
observation in each cell. Most questions in this dataset are answered by
collapsing one or two of those axes:

| Question | Collapse |
|---|---|
| How did revenue trend over time? | sum over category and channel |
| Which category is biggest? | sum over month and channel |
| Online vs Store? | sum over month and category |
| Is there a seasonal pattern by category? | sum over channel, keep month |
| Which single cell is strongest? | keep all three axes |

### Time columns are redundant by design

`month` (`2024-01`), `year` (`2024`), and `month_num` (`1`) carry the same
information in three encodings. They agree on all 240 rows, with zero
mismatches. Keep one, drop two — `month` alone is enough, or derive a real date
from `year` + `month_num` when you need a time axis.

---

## 3. Column-by-column reference

| Column | Type | Unit | Meaning | Range / values |
|---|---|---|---|---|
| `month` | text | — | Month key, `YYYY-MM` | 2024-01 … 2025-12 (24) |
| `year` | integer | — | Calendar year | 2024, 2025 |
| `month_num` | integer | month | Month number | 1–12 |
| `category` | text | — | Product category | 5 values |
| `channel` | text | — | Sales channel | Online, Store |
| `avg_temp_F` | float | °F | Average temperature in that month | 31.3–80.4 |
| `list_price` | integer | USD | Category's full ("sticker") price | 35, 70, 85, 110, 180 |
| `discount_pct` | integer | % | Advertised discount off list price | 0, 10, 15, 20, 25, 30, 40 |
| `sale_price` | float | USD | Price actually charged per unit | 19.57–197.18 |
| `ad_spend` | integer | USD | Advertising spend for that cell | 2,050–19,270 |
| `avg_rating` | float | stars | Average customer rating | 3.4–5.0, 15 distinct values |
| `units_sold` | integer | units | Units sold in the cell | 220–2,488 |
| `revenue` | float | USD | Revenue booked in the cell | 17,257–154,512 |
| `return_rate` | float | fraction | Share of units returned | 0.01–0.36 |

### Two things that trip people up

1. **`return_rate` is a fraction, not a percent.** `0.17` means 17%, not 0.17%.
   Multiply by 100 before displaying it, or you will understate returns by 100×.
2. **`revenue` is gross of returns.** It records what was sold, not what was
   kept. Net revenue is `revenue × (1 − return_rate)`, which reduces the
   two-year total from $13.22M to **$10.85M** — an 18% haircut.

### What the columns are *not* telling you

- `list_price` is **one price per category**, not per product. There are only 5
  distinct values in the whole file, so it describes price tier, not SKU pricing.
- `avg_temp_F` is **one value per month**, identical across all 10 rows of that
  month. It is a city-level monthly average, not a reading per category or channel.
- `avg_rating` is a category-channel-month average, not a customer-level score.
  It takes only 15 distinct values, all to one decimal.
- There is **no cost, margin, COGS, inventory, traffic, conversion, customer, or
  product identifier**. Ad spend is the only cost you can subtract, so "profit"
  is not computable here.

---

## 4. Which columns depend on which

Most columns are independent inputs. Two are not, and understanding them is the
key to the whole file.

### Dependency map

```
avg_temp_F  (one value per month) ─┐
                                   ├──> independent drivers
ad_spend                           │
avg_rating     (independent) ──────┘
discount_pct   (independent)
      │
      ▼
list_price ──> sale_price ──> revenue <── units_sold
   (per            (per unit)    (total)      (independent)
  category)     ↑
                └── ±10% random noise applied on top
```

### Dependency 1 (exact): revenue = sale_price × units_sold

This holds **on every one of the 240 rows, to the cent**. The largest arithmetic
difference is 3×10⁻¹¹ — floating-point noise — so `revenue` is literally computed
from the other two, then rounded to two decimals.

Consequences:

- `revenue` carries no independent information. It is a derived column.
- Any change to price or volume propagates 1:1 into revenue.
- Because it is gross of returns, it overstates money kept.

### Dependency 2 (approximate): sale_price ≈ list_price × (1 − discount_pct / 100)

The discount formula explains most of the price, but a **random ±10% multiplier**
is applied on top. The observed multiplier across all rows behaves like a uniform
draw: minimum 0.902, maximum 1.099, mean 0.998, standard deviation 0.059 — almost
exactly the signature of a uniform distribution on [0.90, 1.10].

Practical effect:

- The formula predicts `sale_price` to within about 5.5% at the median, but never
  exactly: **219 of 240 rows deviate by more than 1%**, and none by more than 10%.
- A separate set of rows — 21 of them, not the same rows — is **priced above
  `list_price`** (up to $197.18 against a $180 list). All of them carry
  `discount_pct = 0`, i.e. "no discount" yet still a premium price. That is
  contradictory and is the file's one genuine data defect.
- `discount_pct` cannot be used to reconstruct `sale_price`, and vice versa.

### What is genuinely independent

| Column | Why it is a free input |
|---|---|
| `units_sold` | Not derived from price or spend; correlates with them but is not computed from them |
| `ad_spend` | Independent marketing decision, unrelated to price |
| `avg_rating` | Independent quality signal, essentially flat across the dataset |
| `return_rate` | Independent outcome, strongly tied to channel rather than price |
| `avg_temp_F` | External weather variable, one value per month |
| `discount_pct` | Marketing decision, not derived from anything else |

So the causal-ish story the file supports is:
**price and discount affect units; units × price gives revenue; ad spend pushes
units; channel and category set the return rate; temperature shifts demand for
outerwear.**

---

## 5. Relationships the numbers actually show

Correlations are Pearson coefficients over all 240 rows. They are descriptive
only — this file has no experimental variation, so none of them prove causation.

| Pair | r | Reading |
|---|---:|---|
| `discount_pct` ~ `units_sold` | **+0.51** | Deeper discounts do move volume — the strongest demand relationship in the file |
| `sale_price` ~ `units_sold` | **−0.52** | Same effect seen from the price side: cheaper units sell more |
| `list_price` ~ `revenue` | **+0.67** | Revenue is driven mostly by price tier, not volume |
| `sale_price` ~ `revenue` | +0.53 | Expensive categories dominate the top line |
| `units_sold` ~ `revenue` | +0.32 | Volume alone explains little, because prices span 5× |
| `ad_spend` ~ `units_sold` | +0.25 | Ads help, weakly and inconsistently |
| `ad_spend` ~ `revenue` | +0.21 | Same, on the revenue side |
| `avg_rating` ~ `units_sold` | +0.13 | Ratings barely move demand |
| `avg_rating` ~ `return_rate` | −0.35 | Better-rated items come back less — sensible, moderate |
| `discount_pct` ~ `return_rate` | −0.10 | Discounting does not drive returns |
| `sale_price` ~ `return_rate` | +0.05 | No relationship at all |
| `avg_temp_F` ~ `revenue` | −0.12 | Overall weather effect is negligible |
| `avg_temp_F` ~ `units_sold` | −0.08 | Same |

By category, temperature tells a sharper story: **Outerwear units vs temperature
r = −0.77** (about 16 fewer units per °F as it warms), while **Dresses r = +0.62**
(warm months sell more dresses). Denim (−0.27), Footwear (−0.22), and Accessories
(−0.25) are weak, and the aggregate hides all of this.

The lesson: relationships in this file are **category-specific and channel-specific**.
Pooled averages routinely point the wrong way.

One exception is worth stating: the discount–volume and price–volume
relationships are **not** a category-mix artefact. Computed inside each category
separately, discount vs units ranges from **+0.55 (Accessories, Dresses) to
+0.75 (Denim)**, and price vs units from −0.56 to −0.77. The pooled figures
(+0.51 and −0.52) understate it. Since `sale_price` is approximately
`list_price × (1 − discount)`, those two rows of the table are two views of the
same underlying discount decision, not two independent effects.

---

## 6. What is surprising

1. **A "sale price" that is higher than the list price.** 21 rows do this, always
   at 0% discount, because of the ±10% noise. Any chart or model that treats
   `sale_price` as "list price minus discount" will be wrong on those rows.
2. **Discounting is permanent, not promotional.** Average discount is 22.6%, the
   median is 25%, and only 43 of 240 cells are at 0%. Realized price sits 22.7%
   below list on average. That is an "always on sale" pricing model.
3. **Zero missing values, everywhere.** Not one blank, null, or non-numeric entry
   in 3,360 cells. Real extracts are almost never this clean.
4. **A perfectly balanced grid.** Exactly 240 cells, no gaps, no duplicates, every
   combination present. Real category×channel data usually has holes.
5. **Revenue is arithmetically exact on every row.** Too exact for a system
   export — it is computed, not recorded.
6. **Prices and temperatures are suspiciously round.** List prices are 35, 70, 85,
   110, 180. Temperature takes one value per month. Ad spend is always a multiple
   of 10. All signs of **simulated/synthetic data**, not a live warehouse extract.
   Treat conclusions as illustrative rather than factual about a real retailer.
7. **Online returns are 2.5× store returns** — 24.6% vs 9.9% of units — in *every*
   category without exception. Returns are a channel phenomenon here, not a
   product one. Returns cost **$2.37M overall, 18% of gross revenue**, and
   **$1.73M of that is online** — 73% of the total damage comes from the channel
   that books only 53% of the revenue.
8. **Outerwear produces 30.6% of revenue from 15.8% of units**, while Accessories
   sells the most units (24.2%) for the least revenue (9.9%). A $127-per-unit
   category and a $27-per-unit category live in the same file.
9. **Ad spend is 35.6% of revenue for Accessories** and only 11.7% for Outerwear.
   The cheapest category is the most expensive to advertise — a margin trap that
   only becomes visible when you join ad spend to revenue.
10. **Ratings are flat by construction.** 4.20–4.25 for every category, never
    below 3.4, only 15 distinct values. There is no quality ranking to discover.
11. **Returns are not a rounding error.** A 17.6% weighted return rate means
    roughly one in six units comes back; any revenue figure quoted without this
    is overstated by about a fifth.
12. **The discount ladder has gaps.** Available values are 0/10/15/20/25/30/40 —
    no 5, no 35, no 45. Whatever process set discounts skipped those steps.

---

## 7. Data-quality verdict

| Check | Result |
|---|---|
| Blank / missing cells | **Pass** — zero across 3,360 cells |
| Non-numeric text in numeric fields | **Pass** |
| Duplicate rows | **Pass** — zero |
| Duplicate month/category/channel keys | **Pass** — zero |
| `month` vs `year` vs `month_num` agreement | **Pass** — 240/240 |
| `revenue` = `sale_price` × `units_sold` | **Pass** — exact on 240/240 |
| Negative or zero prices/spend/units | **Pass** — none |
| `return_rate` within [0, 1] | **Pass** — 0.01–0.36 |
| `sale_price` ≤ `list_price` | **FAIL** — 21 rows above list price |
| `sale_price` = list × (1 − discount) | **FAIL** — ±10% noise, 219/240 rows off by >1% |
| External realism | **Unverified** — no source, no cost data, appears synthetic |

**What to fix, in priority order:**

1. Decide what `sale_price` means. Either remove the noise and make it a strict
   function of `list_price` and `discount_pct`, or keep the noise and drop
   `discount_pct` as a derived/approximate field. Do not use both as if consistent.
2. Flag or cap the 21 rows where `sale_price > list_price`.
3. Document that `return_rate` is a fraction and that `revenue` is gross of returns.
4. Note that `list_price` and `avg_temp_F` are repeated per category and per
   month respectively, so they cannot explain variation within those groups.

---

## 8. Practical guidance for using it

**Loading.** Standard CSV, no quoting or escaping, so any reader works (pandas,
R, Excel, SQL `COPY`). Watch for CRLF line endings on Unix tooling. `wc -l`
returns 241 lines because of the header — the data has 240 rows.

**Aggregating return rates.** Always weight by `units_sold`:

```
category return rate = Σ(units_sold × return_rate) / Σ(units_sold)
```

A plain average of the `return_rate` column gives 17.3% instead of the correct
17.6%, because small-volume cells would count as much as large ones. The same
applies to `avg_rating`.

**Never sum `discount_pct`, `sale_price`, or `avg_temp_F`.** They are per-unit or
per-period values. Only `units_sold`, `revenue`, and `ad_spend` are additive.

**Useful derived metrics that are safe to compute:**

| Metric | Formula |
|---|---|
| Effective price per unit | `revenue / units_sold` |
| Effective discount | `1 − sale_price / list_price` |
| Net revenue after returns | `revenue × (1 − return_rate)` |
| Return on ad spend (ROAS) | `revenue / ad_spend` |
| Ad cost ratio | `ad_spend / revenue` |
| Total units returned | `units_sold × return_rate` |

**What you cannot compute:** gross margin, profit, customer acquisition cost,
lifetime value, sell-through, basket size, or anything product-level. Those
require data this file does not have.

---

## 9. Key numbers, one last time

| Dimension | Leader | Figure |
|---|---|---|
| Revenue by category | Outerwear | $4.05M (30.6%) |
| Units by category | Accessories | 49,086 (24.2%) |
| Revenue per unit | Outerwear | $126.81 |
| Lowest return rate | Accessories | 14.1% |
| Highest return rate | Dresses / Footwear | 22.4% each |
| Best rating | Dresses / Outerwear / Accessories | 4.25 tied |
| Revenue by channel | Online | $6.97M (52.7%) |
| Units by channel | Online | 105,687 (52.2%) |
| Return rate by channel | Online vs Store | 24.6% vs 9.9% |
| Net revenue after returns | **Store** | **$5.61M vs Online $5.24M** |
| Best ad efficiency | Outerwear | 11.7% of revenue |
| Worst ad efficiency | Accessories | 35.6% of revenue |
| Peak months | Nov / Dec | ~$719k / $728k per month |
| Weakest month | June 2024 | $453k |

The single most important takeaway: **gross revenue says Online is the bigger
channel, but after returns and advertising, Store is the more valuable one.**
Any analysis that stops at the `revenue` column will get that backwards.

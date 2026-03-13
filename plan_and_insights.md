# Brand Strategy Analysis — Plan & Insights

---

## 1. Define the Problem

**Business Question:**
Which brands on ASOS have a stockout problem relative to their price point, and how much revenue are they likely losing because of it?

**Why it matters:**
A brand selling at a premium price with half its sizes constantly out of stock is leaving high-margin revenue on the table every single day. Identifying these brands gives the business a clear, actionable restock priority.

**What "success" looks like:**
A clear segmentation of brands by price tier and stockout severity, with a revenue-at-risk estimate to support restocking and budget reallocation decisions.

---

## 2. Data Collection

**Source:** `products_asos.csv` — scraped ASOS product listings

**What we have (9 columns):**

| Column | What it tells us |
|---|---|
| `url` | Product page link |
| `name` | Product title |
| `size` | Available sizes + stockout flags (e.g. `"UK 8, UK 10 - Out of stock"`) |
| `category` | Product category |
| `price` | Listed price (£) |
| `color` | Color label |
| `sku` | Product ID |
| `description` | Full text — **contains brand name** |
| `images` | Image URLs |

**Key observation from the head:**
- Rows 0–3 are the *same product* (same SKU `126704571`) appearing under different brand URLs — this is ASOS cross-listing, not a data error.
- Brand name is **not a direct column** — it must be extracted from `description`.
- Stockout status is **embedded inside the `size` string**, not a separate flag column.

---

## 3. Data Cleaning

### 3a. Fix Price
```python
df['price'] = pd.to_numeric(df['price'], errors='coerce')
df = df.dropna(subset=['price'])
```
Some prices come in as strings or are blank. We force them to numeric and drop anything that can't be converted — a row with no price is useless for revenue analysis.

### 3b. Extract Brand from Description
The description text looks like:
```
"[{'Product Details': 'Coats & Jackets by New Look...'}]"
```
We split on `"by"` and take the next word as the raw brand token:

```python
def get_brand(text):
    if isinstance(text, str) and 'by' in text.lower():
        return text.split('by')[1].strip().split(' ')[0]
    return 'Unknown'
```

**Problem:** Multi-word brands get truncated — `"New Look"` becomes `"New"`, `"River Island"` becomes `"River"`.  
**Fix:** Map known truncated tokens back to their full brand names manually:

```python
brand_map = {
    'New': 'New Look',
    'River': 'River Island',
    'Miss': 'Miss Selfridge',
    'TopshopWelcome': 'Topshop'
}
```

### 3c. Remove Noise Brands
Brands with ≤5 products are statistically unreliable — one outlier skews their metrics. We filter them out. Brands with ≤10 products are also excluded at the aggregation step for the same reason.

---

## 4. Analyzing the Data

### 4a. Parse Stockout from Size String
The `size` column stores availability like:
```
"UK 4, UK 6, UK 8 - Out of stock, UK 10, UK 12 - Out of stock, UK 14"
```

We extract two metrics per product:
- **Stock_Count** — how many size slots are out of stock
- **Stock_Rate** — fraction of sizes out of stock (0.0 to 1.0)

```python
def calculate_phantom_revenue(size_str):
    sizes = size_str.split(',')
    total_sizes = len(sizes)
    out_of_stock_count = size_str.count('Out of stock')
    rate = out_of_stock_count / total_sizes
    return out_of_stock_count, rate
```

### 4b. Estimate Lost Revenue
```python
df_clean['Lost_Revenue'] = df_clean['price'] * df_clean['Stock_Count']
```
This is a **proxy**, not exact sales data. It assumes every OOS size *could have sold*. Likely an overestimate, but excellent for relative prioritization across brands. As a next step, summing all brands in the top-right quadrant gives a concrete £ figure to present to stakeholders.

### 4c. Aggregate by Brand
```python
brand_strategy = df_clean.groupby('Brand').agg({
    'price': 'mean',        # where the brand sits in the market
    'Stock_Rate': 'mean',   # how often they run out
    'Lost_Revenue': 'sum',  # total revenue at risk
    'name': 'count'         # number of products (used as filter)
})
brand_strategy = brand_strategy[brand_strategy['name'] > 10]
```

---

## 5. Data Visualization

**Chart type:** Bubble scatter plot

| Axis / Property | Meaning |
|---|---|
| X-axis (Average Price) | Brand's market positioning — budget vs. premium |
| Y-axis (Stockout Rate) | How severe the availability problem is |
| Bubble size (Lost Revenue) | How much money is at risk |
| Red dashed lines | Thresholds: Price > £40, Stockout Rate > 40% |

**Why these thresholds?**
- £40 separates fast-fashion budget brands from mid-tier — higher margin per unit means stockouts are costlier above this line.
- 40% stockout rate means nearly half the sizes are unavailable — operationally significant.

Only brands in the top-right quadrant are labeled to avoid visual clutter — these are the most urgent cases.

---

## 6. Interpret & Make Decisions

### How to Read the Chart

The bulk of inventory clusters in the **bottom-left** — low price, low stockout rate. This is the "safe zone" but also **low growth potential**. The red lines reveal an outlier group that tells the real story.

---

### 🔴 Top-Right — "The Gold Mine" (High Price + High Stockout)
**Brands: Mango, Pull&Bear, Topshop, VilaAll**

Customers are willing to pay a premium *and* they're buying so fast that brands can't restock in time. This is the most valuable quadrant — high price, high demand — and we are **losing money every single day** these items stay out of stock.

**Recommendation:** Double down on inventory for these specific brands immediately. Reallocate restocking budget here first.

---

### 🟡 Top-Left — "Essential Items" (Low Price + High Stockout)
Budget items with very high demand — think black t-shirts, leggings, basics that customers constantly buy regardless of price. High stockout here means volume lost at scale.

**Recommendation:** Increase reorder frequency. These items don't need a price review — just better supply chain responsiveness.

---

### 🔵 Bottom-Right — "Underperforming Luxury" (High Price + Low Stockout)
Premium-priced items that aren't moving fast. Stock is available but demand is low — they sit on the shelf.

**Recommendation:** Consider liquidating or reducing catalogue size. Budget tied up here is better redirected to the top-right quadrant.

---

### ⚪ Bottom-Left — "Reduce & Simplify" (Low Price + Low Stockout)
Low price, low demand. These items add catalogue noise without contributing much revenue or engagement to the platform.

**Recommendation:** Reduce SKU count in this segment. Fewer, better-performing products are more valuable than a bloated low-performing catalogue.

---

### Final Strategic Summary

> **Reallocate budget from the bottom-right (underperforming high-price items) → into inventory for the top-right (high-demand premium brands).**

The Lost Revenue column gives us the ability to quantify this in £ — summing the top-right quadrant brands produces a concrete number to bring to a stakeholder meeting and justify the restock investment.

---

## Limitations

- Stockout data is a **snapshot** — not a sustained measurement over time
- Lost Revenue assumes every OOS size would have sold — overestimates actual impact
- Brand extraction via `"by"` parsing is imperfect; some brands may still be mislabeled
- Cross-listed products (same SKU, multiple brand URLs) may inflate certain brand counts

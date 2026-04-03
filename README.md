# Data Wrangling with Pandas — Reference README

## What is data wrangling?

Real data arrives from multiple sources — spreadsheets, APIs, database dumps — and none of them agree on format, column names, or which rows exist. Data wrangling is the process of combining, cleaning, and reshaping this raw material into one coherent table before any analysis happens. Pandas is the tool Python gives you to do that.

Pandas is built on top of NumPy. Where NumPy thinks in arrays, Pandas thinks in labeled tables (DataFrames). The labels are what make combining data possible.

---

## Exercise 1: Concatenate

### The problem it solves

You have two tables with the **same columns** — think two spreadsheets covering different regions or time periods. You want to stack them vertically into one.

### Key concept: the index

Every DataFrame has an index — a label for each row. By default it's integers: 0, 1, 2...

When you concatenate two DataFrames, Pandas stacks them but keeps both original indexes. You end up with duplicate index labels (two rows labeled `0`). Fix this with `reset_index(drop=True)` — it throws away the old labels and assigns a fresh 0, 1, 2, 3.

`drop=True` means: don't add the old index as a column, just discard it.

### Code

```python
import pandas as pd

df1 = pd.DataFrame([['a', 1], ['b', 2]], columns=['letter', 'number'])
df2 = pd.DataFrame([['c', 1], ['d', 2]], columns=['letter', 'number'])

result = pd.concat([df1, df2]).reset_index(drop=True)
```

`pd.concat()` takes a **list** of DataFrames. The result has a clean `RangeIndex(start=0, stop=4, step=1)`.

---

## Exercise 2: Merge

### The problem it solves

Concatenation was simple — same columns, just stack. But what if two tables are about **related things** and you want to combine them **horizontally** by matching on a shared value? That shared column is called a **key**. This is the same concept as SQL joins.

### Key concept: join types

When you match two tables on a key, not every row will have a match on both sides. You decide what to do with the leftovers:

| Join type | What it keeps |
|:----------|:--------------|
| `inner` | Only rows where the key exists in **both** tables |
| `outer` | Everything from both tables, fills gaps with `NaN` |
| `left` | Everything from the left table, fills gaps from the right with `NaN` |
| `right` | Everything from the right table, fills gaps from the left with `NaN` |

### Key concept: suffixes

When both DataFrames have columns with the same name (e.g. `Feature1`), Pandas needs to tell them apart after merging. By default it adds `_x` and `_y`. You can control this with the `suffixes` parameter.

### Code

```python
# Inner merge — only rows where id exists in both
q1 = pd.merge(df1, df2, on='id', how='inner')

# Outer merge — everything from both, custom suffixes
q2 = pd.merge(df1, df2, on='id', how='outer', suffixes=('_df1', '_df2'))
```

---

## Exercise 3: Merge MultiIndex

### The problem it solves

Sometimes a row is identified by **two things at once** — a stock price needs both a date and a ticker symbol to be unique. A MultiIndex uses pairs (or more) of labels instead of one label per row.

```
                    Open    Close
(2021-01-01, AAPL)  0.45   0.47
(2021-01-01, FB)   -0.12   0.31
```

### Key concept: merging on the index

In previous exercises we merged on a column. Here the shared key is **baked into the index itself**. We tell Pandas to use the index as the key with `left_index=True, right_index=True`.

### Key concept: left merge with mismatched data

`market_data` has only business days. `alternative_data` has every calendar day but is missing the last 15 days. Using `market_data` as the left side of a left merge means: keep all market rows, pull in alternative data where it exists, fill the rest with `NaN`. Then fill those `NaN`s with 0.

### Code

```python
# Merge using the index as the key
merged = market_data.merge(alternative_data, how='left', left_index=True, right_index=True)

# Fill missing values with 0
filled = merged.fillna(0)
```

The resulting shape is `(1305, 5)` — 261 business days × 5 tickers, with 5 columns.

---

## Exercise 4: Groupby Apply

### The problem it solves

You want to run a calculation **separately on each group** within a dataset, not on the whole thing at once. The pattern is: split the data into groups → apply a function to each group → combine the results back.

### Key concept: winsorizing

Real data has outliers — one absurdly large value that skews everything. Winsorizing caps extreme values instead of removing them:

- Values **above** the upper percentile → replaced by that percentile value
- Values **below** the lower percentile → replaced by that percentile value
- Everything in between → left alone

`np.quantile(df, 0.05)` answers: "what value sits at the 5th percentile of this data?" That becomes your floor. `np.quantile(df, 0.95)` becomes your ceiling.

`.clip(lower=min_value, upper=max_value)` enforces that floor and ceiling on every value.

### Key concept: groupby + apply

```python
df.groupby("group")[['sequence']].apply(winsorize, [0.05, 0.95])
```

This splits the DataFrame into one mini-DataFrame per group, calls `winsorize` on each mini-DataFrame separately (so each group gets its own percentile boundaries), then stitches the results back together.

This is why groupby matters — if you ran winsorize on the whole dataset at once, group 1's values would be capped using group 5's percentiles. That would be wrong.

### Code

```python
def winsorize(df, quantiles):
    min_value = np.quantile(df, quantiles[0])  # floor: value at lower percentile
    max_value = np.quantile(df, quantiles[1])  # ceiling: value at upper percentile
    return df.clip(lower=min_value, upper=max_value)

# Apply winsorize to each group separately
result = df.groupby("group")[['sequence']].apply(winsorize, [0.05, 0.95])
```

---

## Exercise 5: Groupby Agg

### The problem it solves

Instead of applying a custom function to groups, you want standard summary statistics — min, max, mean — computed per group in one shot.

### Key concept: .agg()

`.agg()` stands for aggregate. You pass a dictionary that says "for this column, compute these statistics":

```python
df.groupby('product').agg({'value': ['min', 'max', 'mean']})
```

This reads as: group by product, then for the `value` column compute min, max, and mean — all in one line. The result has a MultiIndex on the columns: `('value', 'min')`, `('value', 'max')`, `('value', 'mean')`.

### Code

```python
result = df.groupby('product').agg({'value': ['min', 'max', 'mean']})
```

---

## Exercise 6: Unstack

### The problem it solves

A MultiIndex is great for storage and merging, but terrible for plotting or cross-ticker calculations. If you want to plot AAPL's price over time, its rows are scattered throughout the table mixed with other tickers.

Unstacking takes one level of the MultiIndex and turns it into columns. Each ticker becomes its own column, each row becomes one date. Now each column is one time series and plotting is trivial.

### Key concept: .unstack()

Called on a MultiIndex DataFrame, `.unstack()` pivots the innermost index level into columns.

Before:
```
Date        Ticker   Prediction
2021-01-01  AAPL     0.38
2021-01-01  FB      -0.06
```

After:
```
Date        AAPL    FB
2021-01-01  0.38   -0.06
```

### Code

```python
# Unstack the Ticker level into columns
unstacked = market_data.unstack()

# Plot all 5 time series
unstacked.plot(title='Stocks 2021')
```

---

## Quick Reference: Key Methods

| Method | What it does |
|:-------|:-------------|
| `pd.concat([df1, df2])` | Stack DataFrames vertically |
| `.reset_index(drop=True)` | Reassign a clean integer index |
| `pd.merge(df1, df2, on='id', how='inner')` | Merge on a shared column |
| `pd.merge(..., left_index=True, right_index=True)` | Merge on the index |
| `pd.merge(..., suffixes=('_df1', '_df2'))` | Control suffix on duplicate column names |
| `.fillna(0)` | Replace NaN with 0 |
| `np.quantile(df, 0.05)` | Value at the 5th percentile |
| `.clip(lower=x, upper=y)` | Cap values to a floor and ceiling |
| `.groupby('col')[['col2']].apply(fn, args)` | Apply a function to each group |
| `.groupby('col').agg({'col2': ['min', 'max', 'mean']})` | Compute summary stats per group |
| `.unstack()` | Pivot innermost index level into columns |

# Advanced Slicing Guide

> Use case: when a single query matches a very large number of assets (e.g. millions), and the API limits how many results a **single query** can return, splitting the query into multiple mutually-exclusive sub-queries lets you bypass the per-query depth limit and retrieve more data.

---

## 1. Background: The 60-result per-query hard limit

Verified by real API testing (2026-10-17):

- Each request's `page × size` **must not exceed 60**, i.e. a single query can only ever return its **first 60 results**:

| Request | Result |
|---|---|
| `page=2, size=30` (=60) | ✅ OK |
| `page=6, size=10` (=60) | ✅ OK |
| `page=2, size=31` (=62) | ❌ 502 |
| `page=7, size=10` (=70) | ❌ 502 |
| `size=100` | truncated to 50 (API caps `size` at 50) |

- This limit comes from the **upstream data source** and is **independent of your API Key**: requesting the same over-limit combination with any other key still returns 502.
- In short: a single query can return at most 60 results. Going past result #61 by paging alone is not possible.

---

## 2. Principle: The limit applies per query statement

Key insight: **the 60-result limit is calculated independently for each query statement**.

```
title="login"                     → first 60 results available
title="login" && country="CN"     → another 60 results (no interference)
title="login" && country="US"     → yet another 60 results
```

Therefore: **split a large query into N mutually-exclusive sub-queries, each sub-query can retrieve its own 60 results, total output ≈ N × 60**.

---

## 3. Choosing slicing dimensions

The essence of slicing: pick a field that partitions the result set into **mutually-exclusive, roughly balanced** buckets.

Common dimensions (in recommended order):

| Dimension | Syntax | Notes |
|---|---|---|
| **Country** | `country="CN"` | Most common; good exclusivity, volumes vary by country but acceptable |
| **Region** | `region="Beijing"` | Slice further when a country has too many results |
| **Port** | `port="443"`, `port="80"` | Good for enumerating common ports |
| **Org / ASN** | `org="xxx"` | Good for concentrated asset sets |
| **IP range** | `ip="1.2.3.0/24"` | Good when known ranges exist |
| **Protocol** | `protocol="https"` | Combine with others when needed |
| **Exclusion chain** | `country!="CN" && country!="US"` | Catch-all for "everything else" to guarantee full coverage |

> Rules of thumb: sub-queries must **not overlap** (exclusive), and their union must **cover the full result set** (complete).
> If enumeration doesn't cover everything, add an `exclusion chain` sub-query as a catch-all.

---

## 4. Complete example (Python)

The following code slices a large query by country, pages each sub-query up to its limit, then merges and deduplicates:

```python
import requests
import time

API = "https://fofa.shanshuiapi.com/api/search"
KEY = "<your key>"

def search(query, page=1, size=50):
    resp = requests.post(API, headers={"X-API-Key": KEY},
                         json={"query": query, "page": page, "size": size})
    resp.raise_for_status()
    return resp.json()

BASE_QUERY = 'title="login"'

# 1. Probe each sub-query's total count first (decides how many pages per slice)
countries = ["CN", "US", "JP"]   # adjust as needed
slices = []
for c in countries:
    total = search(f'{BASE_QUERY} && country="{c}"', size=2).get("total", 0)
    if total:
        slices.append((c, total))
    print(f"country={c}: total={total}")

# 2. Pull each slice: at most 60 results per sub-query (page×size ≤ 60)
seen = set()
assets = []
for c, total in slices:
    # Page 1: size=50
    data = search(f'{BASE_QUERY} && country="{c}"', page=1, size=50)
    for a in data.get("assets", []):
        key = (a.get("ip"), a.get("port"))
        if key not in seen:
            seen.add(key)
            assets.append(a)
    # Page 2: size=10 (fills up to 60)
    if total > 50:
        time.sleep(0.5)
        data = search(f'{BASE_QUERY} && country="{c}"', page=2, size=10)
        for a in data.get("assets", []):
            key = (a.get("ip"), a.get("port"))
            if key not in seen:
                seen.add(key)
                assets.append(a)
    time.sleep(0.5)

print(f"Total after slicing: {len(assets)} results (single query is capped at 60)")
```

### Measured results (2026-10-17)

Neutral test query `title="login"` (about 29.71M matches):

| Approach | Results retrieved |
|---|---|
| Single query (no slicing) | 60 (capped) |
| Sliced by 3 countries | **154 (about 3×)** |
| Sliced by 6 countries | 212 (about 3.5×) |
| Finer slicing (region × port × ASN) | 390 (about 6.5×) |

> The finer the slices, the higher the multiplier; the trade-off is **more requests** (about 2 per slice), which consumes more daily quota.

---

## 5. Notes

1. **Quota usage**: each request costs 1 credit (regardless of result count). Slicing multiplies the request count, so budget your daily quota first.
2. **Deduplication**: some duplicates may occur across pages/slices (measured ~5% across pages); deduplicate by `(ip, port)` or `host`.
3. **Rate limiting**: the API has a QPS limit (429 when exceeded); add a small delay (e.g. 0.5–1s) between requests in loops.
4. **Exclusivity**: slicing dimensions must be mutually exclusive, otherwise the same asset appears in multiple sub-queries, wasting quota.
5. **Completeness**: if enumeration doesn't cover everything, add an `exclusion chain` sub-query as a catch-all so no data is lost.
6. **502 handling**: if a sub-query returns 502, it usually means `page×size` exceeded 60; fix the parameters.

---

## 6. Quick troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `page=2, size=31` returns 502 | page×size > 60 | adjust size so the product ≤ 60 |
| Still 502 after switching keys | limit is upstream, not key-related | use slicing |
| `size=100` returns only 50 | API caps size at 50 | paginate instead |
| 429 | requests too fast | add delay and retry |
| Lots of duplicates | overlapping slices/pages | deduplicate by (ip, port) |

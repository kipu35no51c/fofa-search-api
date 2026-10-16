# FOFA Search Service API Documentation

**[中文](./README.md) | [English](./README.en.md)**

FOFA asset search HTTP API. An access token (API Key) is required before use.

- **Base URL**: `https://fofa.shanshuiapi.com`
- **Protocol**: HTTPS / JSON

---

## 1. Authentication

Every request must carry a token. Choose one of the following:

**Request header (recommended)**

```
X-API-Key: <your-token>
```

**Query parameter**

```
?api_key=<your-token>
```

### Test Key

> Quota: **500 searches/day**, **auto-resets at midnight**.
>
> `X-API-Key: 4d9101c1a62728bcf23f6bd62a0fec33`

---

## 2. Search Endpoint

```
POST /api/search
Content-Type: application/json
```

### Request Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| query | string | ✅ | Search syntax, see「Query Syntax」below |
| page | int | ❌ | Page number, starts at 1, default 1 |
| size | int | ❌ | Results per page, range **2 ~ 50**, default 10 |

### Request Example

```bash
# Normal search (costs 1 credit)
curl -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-token>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

### Response Example

```json
{
  "code": 0,
  "query": "title=\"login\"",
  "page": 1,
  "size": 10,
  "fields": ["ip", "port", "protocol", "host", "title", "domain", "country", "region", "city", "server", "banner", "status", "org", "mtime"],
  "total": 30081754,
  "remaining_today": 997,
  "assets": [
    {
      "ip": "52.70.38.235",
      "port": 443,
      "protocol": "https",
      "host": "https://52.70.38.235",
      "title": "Airflow - Login",
      "domain": "",
      "country": "United States",
      "region": "Virginia",
      "city": "Ashburn",
      "server": "gunicorn",
      "banner": "",
      "status": 200,
      "org": "",
      "mtime": "2026-08-19 08:00:00"
    }
  ]
}
```

### Response Fields

| Field | Type | Description |
|---|---|---|
| code | int | 0 = success, non-zero = error, see「Error Codes」 |
| query | string | Echo of the query string |
| page / size | int | Echo of pagination parameters |
| fields | array | List of field names in this response |
| **total** | int | **Total number of assets matching the query** (real match count, useful for display/pagination planning) |
| remaining_today | int | Remaining searches today (resets daily) |
| warning | string | Notice: daily quota nearly used (≤10% remaining) |
| assets | array | Asset list, each item contains `ip`, `port`, `protocol`, `host`, `title`, `domain`, `country`, `region`, `city`, `server`, `banner`, `status`, `org`, `mtime` |

### Pagination

Change the `page` parameter to paginate; combine with `total` to compute the number of pages:

```bash
# Page 1
-d '{"query":"domain=\"baidu.com\"","page":1,"size":50}'
# Page 2
-d '{"query":"domain=\"baidu.com\"","page":2,"size":50}'
# Page 3
-d '{"query":"domain=\"baidu.com\"","page":3,"size":50}'
```

> Note: `size` is capped at 50 per request. For more data, paginate.

> 💡 **Pro tip**: when a single query matches a huge number of assets (e.g. hundreds of thousands),
> direct pagination can only reach the first 60 results. By splitting the query into multiple
> mutually-exclusive sub-queries (e.g. by country / port / region), you can bypass the per-query
> depth limit and retrieve more data.
> See: [Advanced Slicing Guide](./ADVANCED-SLICING.en.md)

---

## 3. Query Syntax

Same syntax as the FOFA web console. Common examples:

| Purpose | Query |
|---|---|
| Title contains "login" | `title="login"` |
| By domain | `domain="baidu.com"` |
| By IP | `ip="1.2.3.4"` |
| By port | `port="443"` |
| By protocol | `protocol="https"` |
| By country/region | `country="CN"`, `region="Beijing"` |
| Combined conditions | `title="login" && country="CN"` |
| Exclusion | `title="login" && country!="US"` |

---

## 4. Error Codes

| code | HTTP | Meaning | Action |
|---|---|---|---|
| 0 | 200 | Success | — |
| 401 | 401 | Invalid or missing token | Check `X-API-Key` |
| 403 | 403 | Daily quota exhausted | Wait for reset |
| 400 | 400 | Malformed request, or FOFA query syntax error | Check request body / query syntax |
| 405 | 405 | Method not POST | Use POST |
| 429 | 429 | Rate limit exceeded | Slow down and retry |
| 502 | 502 | Upstream data source unavailable | Retry later |

---

## 5. Token Info

```
GET /api/token-info
```

**Request** (with token)

```bash
curl -H "X-API-Key: <your-token>" https://fofa.shanshuiapi.com/api/token-info
```

**Response example**

```json
{
  "code": 0,
  "user_name": "card001",
  "remark": "batch-20260819",
  "daily_quota": 1000,
  "today_used": 1,
  "today_remaining": 999
}
```

---

## 6. Health Check

```
GET /health
```

Returns `{"ok":true}`, useful for connectivity monitoring.

---

## 7. Code Examples

### Python

```python
import requests

API = "https://fofa.shanshuiapi.com/api/search"
KEY = "<your-token>"

def search(query, page=1, size=10):
    resp = requests.post(API, headers={"X-API-Key": KEY},
                         json={"query": query, "page": page, "size": size})
    resp.raise_for_status()
    return resp.json()

# Single search: use "total" to know the real match count
data = search('title="login"', size=50)
print("Total matches:", data.get("total"))
for asset in data.get("assets", []):
    print(asset["ip"], asset["port"], asset["title"])

# Paginate through multiple pages
for page in range(1, 4):
    data = search('title="login"', page=page, size=50)
    for asset in data.get("assets", []):
        print(asset["ip"], asset["port"], asset["title"])
```

### Command Line

```bash
curl -s -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-token>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

---

## 8. Community

Telegram group: https://t.me/+dBrFr66Y8uo1ZDgx
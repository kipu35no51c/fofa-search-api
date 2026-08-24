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
| download | bool | ❌ | Also export all matching assets. `true` costs **2 credits** and returns an export task |

### Request Example

```bash
# Normal search (costs 1 credit)
curl -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-token>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'

# Search + export (costs 2 credits, creates an export task)
curl -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-token>" \
  -d '{"query":"title=\"login\"","page":1,"size":10,"download":true}'
```

### Response Example

```json
{
  "code": 0,
  "query": "title=\"login\"",
  "page": 1,
  "size": 10,
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
| fields | array/null | List of field names, may be null |
| remaining_today | int | Remaining searches today (resets daily) |
| warning | string | Notice: daily quota nearly used (≤10% remaining) |
| assets | array | Asset list, each item contains `ip`, `port`, `protocol`, `host`, `title`, `domain`, `country`, `region`, `city`, `server`, `banner`, `status`, `org`, `mtime` |
| export_task_id | int | **Only when download=true**: export task ID |
| export_status_url | string | **Only when download=true**: export task status URL |

### Pagination

Change the `page` parameter to paginate:

```bash
# Page 1
-d '{"query":"domain=\"baidu.com\"","page":1,"size":50}'
# Page 2
-d '{"query":"domain=\"baidu.com\"","page":2,"size":50}'
# Page 3
-d '{"query":"domain=\"baidu.com\"","page":3,"size":50}'
```

---

## 3. Export / Download (after download=true)

A search with `download:true` asynchronously creates an export task that exports all assets matching the query (cap is determined by the FOFA free quota, typically 2500~2800 rows).

### 3.1 Check Export Task Status

```
GET /api/export/{task_id}
```

**Request** (with token)

```bash
curl -H "X-API-Key: <your-token>" https://fofa.shanshuiapi.com/api/export/1
```

**Response example** (processing)

```json
{
  "code": 0,
  "task_id": 1,
  "status": "processing",
  "query": "title=\"login\"",
  "limit": 2800,
  "format": "csv"
}
```

**Response example** (done)

```json
{
  "code": 0,
  "task_id": 1,
  "status": "done",
  "query": "title=\"login\"",
  "limit": 2614,
  "format": "csv",
  "file_size": 59697,
  "download_url": "/api/export/1/file"
}
```

**Response example** (failed)

```json
{
  "code": 0,
  "task_id": 1,
  "status": "failed",
  "query": "title=\"login\"",
  "limit": 2800,
  "format": "csv",
  "message": "create download: ..."
}
```

| Status | Description |
|---|---|
| processing | Export in progress, retry later (usually 5~30 seconds) |
| done | Complete, `download_url` is ready |
| failed | Failed, `message` explains why (e.g. FOFA free quota exhausted) |

### 3.2 Download the Export File

```
GET /api/export/{task_id}/file
```

**Request** (with token)

```bash
curl -o result.csv -H "X-API-Key: <your-token>" https://fofa.shanshuiapi.com/api/export/1/file
```

Returns a CSV file (`Content-Type: text/csv`, supports range/resume). Files are cached on the server for **24 hours**; after expiry, re-search with download to regenerate.

---

## 4. Query Syntax

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

## 5. Error Codes

| code | HTTP | Meaning | Action |
|---|---|---|---|
| 0 | 200 | Success | — |
| 401 | 401 | Invalid or missing token | Check `X-API-Key` |
| 403 | 403 | Daily quota exhausted | Wait for reset |
| 400 | 400 | Malformed request (missing query / invalid JSON) | Check request body |
| 405 | 405 | Method not POST | Use POST |
| 429 | 429 | Rate limit exceeded | Slow down and retry |
| 409 | 409 | Export file not ready (still processing) | Retry later |
| 410 | 410 | Export file expired (>24h) | Re-search with download |
| 502 | 502 | Upstream data source unavailable | Retry later |

---

## 6. Token Info

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

## 7. Health Check

```
GET /health
```

Returns `{"ok":true}`, useful for connectivity monitoring.

---

## 8. Code Examples

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

# Paginate through multiple pages
for page in range(1, 4):
    data = search('title="login"', page=page, size=50)
    for asset in data.get("assets", []):
        print(asset["ip"], asset["port"], asset["title"])

# Search + export all matching assets (costs 2 credits)
data = search('title="login"', download=True)
task_id = data["export_task_id"]

# Poll export status until done
import time
while True:
    st = requests.get(f"https://fofa.shanshuiapi.com/api/export/{task_id}",
                      headers={"X-API-Key": KEY}).json()
    if st["status"] == "done":
        break
    time.sleep(3)

# Download the export file
r = requests.get(f"https://fofa.shanshuiapi.com/api/export/{task_id}/file",
                 headers={"X-API-Key": KEY})
open("result.csv", "wb").write(r.content)
```

### Command Line

```bash
curl -s -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-token>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

---

## 9. Community

Telegram group: https://t.me/+dBrFr66Y8uo1ZDgx

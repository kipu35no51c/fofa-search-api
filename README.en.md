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

> Tokens are issued by the administrator. Each token has a quota of **1000 searches/day** (resets at midnight), valid for **30 days**.

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
| warning | string | Notice: quota nearly used (≤10%) or token expiring soon (≤7 days) |
| expires_at / expires_days | string/int | Token expiry time / days remaining (only shown within 7 days of expiry) |
| assets | array | Asset list, each item contains `ip`, `port`, `protocol`, `host`, `title`, `domain`, `country`, `region`, `city`, `server`, `banner`, `status`, `org`, `mtime` |

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

## 4. Rate Limits & Quota

- **QPS**: Per-token rate limit, default **10 requests/second**, returns `429` when exceeded.
- **Daily quota**: Each token has a daily search limit (default 1000/day), resets at midnight. Every successful response includes `remaining_today` showing remaining searches for today.
- **Quota notice**: When remaining searches ≤10%, the response includes a `warning`.
- **Token validity**: Default 30 days. A `warning` is included within 7 days of expiry; after expiry, requests return `403`.

---

## 5. Error Codes

| code | HTTP | Meaning | Action |
|---|---|---|---|
| 0 | 200 | Success | — |
| 401 | 401 | Invalid or missing token | Check `X-API-Key` |
| 403 | 403 | Daily quota exhausted, or token expired | Wait for reset, or contact admin to renew |
| 400 | 400 | Malformed request (missing query / invalid JSON) | Check request body |
| 405 | 405 | Method not POST | Use POST |
| 429 | 429 | Rate limit exceeded | Slow down and retry |
| 502 | 502 | Upstream data source unavailable | Retry later; contact admin if persistent |

---

## 6. Code Examples

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

# Fetch multiple pages
for page in range(1, 4):
    data = search('title="login"', page=page, size=50)
    for asset in data.get("assets", []):
        print(asset["ip"], asset["port"], asset["title"])
```

### Go

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"net/http"
)

func main() {
	body, _ := json.Marshal(map[string]any{
		"query": `title="login"`,
		"page":  1,
		"size":  10,
	})
	req, _ := http.NewRequest("POST", "https://fofa.shanshuiapi.com/api/search", bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-API-Key", "<your-token>")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	var result map[string]any
	json.NewDecoder(resp.Body).Decode(&result)
	fmt.Printf("%+v\n", result)
}
```

### Command Line

```bash
curl -s -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-token>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

---

## 7. Health Check

```
GET /health
```

Returns `{"ok":true}`, useful for connectivity monitoring.

---

## 8. Community

Telegram group: https://t.me/+dBrFr66Y8uo1ZDgx

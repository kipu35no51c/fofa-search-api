# FOFA 搜索服务 API 文档

FOFA 资产搜索 HTTP 接口。使用前请先获取访问令牌（API Key）。

- **接口地址**：`https://fofa.shanshuiapi.com`
- **协议**：HTTPS / JSON

---

## 1. 认证

所有请求需携带令牌，二选一：

**请求头方式（推荐）**
```
X-API-Key: <你的令牌>
```

**查询参数方式**
```
?api_key=<你的令牌>
```

---

## 2. 搜索接口

```
POST /api/search
Content-Type: application/json
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| query | string | ✅ | 搜索语法，见下方「查询语法」 |
| page | int | ❌ | 页码，从 1 开始，默认 1 |
| size | int | ❌ | 每页返回条数，范围 **2 ~ 50**，默认 10 |

**请求示例**
```bash
curl -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <你的令牌>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

**响应示例**
```json
{
  "code": 0,
  "query": "title=\"login\"",
  "page": 1,
  "size": 10,
  "assets": [
    {
      "ip": "52.70.38.235",
      "port": 443,
      "protocol": "https",
      "host": "https://52.70.38.235",
      "title": "Airflow - Login",
      "domain": "",
      "country": "美国",
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

### 响应字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| code | int | 0 表示成功，非 0 见「错误码」 |
| query | string | 回显请求的查询语句 |
| page / size | int | 回显本次翻页参数 |
| fields | array/null | 字段名列表，可能为 null（当前接口未返回） |
| assets | array | 资产列表，每条含：`ip`、`port`、`protocol`、`host`、`title`、`domain`、`country`、`region`、`city`、`server`、`banner`、`status`、`org`、`mtime` |

### 翻页

通过修改 `page` 参数翻页：

```bash
# 第 1 页
-d '{"query":"domain=\"baidu.com\"","page":1,"size":50}'
# 第 2 页
-d '{"query":"domain=\"baidu.com\"","page":2,"size":50}'
# 第 3 页
-d '{"query":"domain=\"baidu.com\"","page":3,"size":50}'
```

---

## 3. 查询语法

与 FOFA 网页版语法一致，常用示例：

| 目的 | 查询语句 |
|---|---|
| 标题包含 login | `title="login"` |
| 域名 | `domain="baidu.com"` |
| 指定 IP | `ip="1.2.3.4"` |
| 端口 | `port="443"` |
| 协议 | `protocol="https"` |
| 国家/地区 | `country="CN"`、`region="Beijing"` |
| 组合条件 | `title="login" && country="CN"` |
| 排除 | `title="login" && country!="US"` |

---

## 4. 限流

- 每个令牌独立限流，互不影响。
- 默认 **10 次/秒**。
- 超出后返回 `429`，请降低请求频率后重试。

---

## 5. 错误码

| code | HTTP | 含义 | 处理方式 |
|---|---|---|---|
| 0 | 200 | 成功 | — |
| 401 | 401 | 令牌无效或缺失 | 检查 `X-API-Key` |
| 400 | 400 | 请求格式错误（缺 query 或 JSON 非法） | 检查请求体 |
| 405 | 405 | 请求方法不是 POST | 使用 POST |
| 429 | 429 | 超过限流 | 降速后重试 |
| 502 | 502 | 上游数据源暂不可用 | 稍后重试，持续失败请联系管理员 |

---

## 6. 代码示例

### Python

```python
import requests

API = "https://fofa.shanshuiapi.com/api/search"
KEY = "<你的令牌>"

def search(query, page=1, size=10):
    resp = requests.post(API, headers={"X-API-Key": KEY},
                         json={"query": query, "page": page, "size": size})
    resp.raise_for_status()
    return resp.json()

# 翻页拉取多页结果
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
	req.Header.Set("X-API-Key", "<你的令牌>")

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

### 命令行

```bash
curl -s -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <你的令牌>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

---

## 7. 健康检查

```
GET /health
```
返回 `{"ok":true}`，可用于连通性监控。

---

## 8. 交流群

Telegram 交流群：https://t.me/+dBrFr66Y8uo1ZDgx

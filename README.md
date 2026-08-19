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

### 令牌列表

每个令牌额度 **1000 次/天**（每天零点重置），有效期 **30 天**。

| 令牌 | 备注 |
|---|---|
| `ebad3a957bf1a1ed541f07f473956807` | 用户1 |
| `d826d1a40200fde4efa433ee1c5529ab` | 用户2 |
| `a06eae6f2c59eff45339d1512b25b06f` | 用户3 |
| `ffdd9309719902444c025da87ad95866` | 用户4 |
| `62cdd8dbdf73eab528e6737e70b48566` | 用户5 |
| `d0211999ea190986242fea182a4ff027` | 用户6 |
| `9d2e486b35d0559ef2685bdfae22aaf9` | 用户7 |
| `a34383ec3d55bce478fda8024ac7d772` | 用户8 |
| `bd91c120a3b758b146637982579f0bf8` | 用户9 |
| `3627f5868f81dbc85915569bc7f3494d` | 用户10 |

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
  "remaining_today": 997,
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
| remaining_today | int | 今日剩余搜索次数（每日重置） |
| warning | string | 提示：额度即将用完（剩余≤10%）或 token 即将到期（≤7天） |
| expires_at / expires_days | string/int | token 到期时间/剩余天数（仅到期前 7 天内出现） |
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

## 4. 限流与额度

- **QPS**：每个令牌独立限流，默认 **10 次/秒**，超出返回 `429`。
- **每日额度**：每个令牌每天有搜索次数上限（默认 1000 次/天，可配置），每天零点自动重置。每次成功响应带 `remaining_today` 字段显示今日剩余次数。
- **额度提示**：今日剩余次数 ≤10% 时，响应带 `warning` 提示。
- **令牌有效期**：默认 30 天，到期前 7 天响应带 `warning` 提示；到期后返回 `403`。

---

## 5. 错误码

| code | HTTP | 含义 | 处理方式 |
|---|---|---|---|
| 0 | 200 | 成功 | — |
| 401 | 401 | 令牌无效或缺失 | 检查 `X-API-Key` |
| 403 | 403 | 今日额度用完，或令牌已到期 | 等次日额度重置，或联系管理员续期 |
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

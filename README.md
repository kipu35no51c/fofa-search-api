# FOFA 搜索服务 API 文档

**[中文](./README.md) | [English](./README.en.md)**

FOFA 资产搜索 HTTP 接口，使用前请先获取访问令牌（API Key）。

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

### 测试 Key

> 额度 **500 次/天**，**每晚 12 点自动重置**。
>
> `X-API-Key: 4d9101c1a62728bcf23f6bd62a0fec33`

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
| download | bool | ❌ | 是否同时导出全部匹配资产。`true` 时消耗 **2 次**额度，响应附带导出任务 |

### 请求示例

```bash
# 普通搜索（消耗 1 次额度）
curl -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <你的令牌>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'

# 带导出搜索（消耗 2 次额度，同时触发导出任务）
curl -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <你的令牌>" \
  -d '{"query":"title=\"login\"","page":1,"size":10,"download":true}'
```

### 响应示例

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
| fields | array/null | 字段名列表，可能为 null |
| remaining_today | int | 今日剩余搜索次数（每日重置） |
| warning | string | 提示：今日额度即将用完（剩余 ≤10%） |
| assets | array | 资产列表，每条含 `ip`、`port`、`protocol`、`host`、`title`、`domain`、`country`、`region`、`city`、`server`、`banner`、`status`、`org`、`mtime` |
| export_task_id | int | **仅 download=true**：导出任务 ID |
| export_status_url | string | **仅 download=true**：导出任务状态查询地址 |

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

## 3. 导出下载（download=true 后）

带 `download:true` 的搜索会异步创建导出任务，全量导出匹配该查询的资产（上限由 FOFA 免费额度决定，通常 2500~2800 条）。

### 3.1 查询导出任务状态

```
GET /api/export/{task_id}
```

**请求**（带令牌）

```bash
curl -H "X-API-Key: <你的令牌>" https://fofa.shanshuiapi.com/api/export/1
```

**响应示例**（进行中）

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

**响应示例**（完成）

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

**响应示例**（失败）

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

| 状态 | 说明 |
|---|---|
| processing | 导出任务进行中，稍后重试（一般 5~30 秒） |
| done | 完成，`download_url` 可直接下载 |
| failed | 失败，`message` 说明原因（如 FOFA 免费额度不足） |

### 3.2 下载导出文件

```
GET /api/export/{task_id}/file
```

**请求**（带令牌）

```bash
curl -o result.csv -H "X-API-Key: <你的令牌>" https://fofa.shanshuiapi.com/api/export/1/file
```

返回 CSV 文件（`Content-Type: text/csv`，支持断点续传）。文件在服务端缓存 **24 小时**，过期后需重新搜索导出。

---

## 4. 查询语法

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

## 5. 错误码

| code | HTTP | 含义 | 处理方式 |
|---|---|---|---|
| 0 | 200 | 成功 | — |
| 401 | 401 | 令牌无效或缺失 | 检查 `X-API-Key` |
| 403 | 403 | 今日额度用完 | 等次日重置 |
| 400 | 400 | 请求格式错误（缺 query 或 JSON 非法） | 检查请求体 |
| 405 | 405 | 请求方法不是 POST | 使用 POST |
| 429 | 429 | 超过限流 | 降速后重试 |
| 409 | 409 | 导出文件未就绪（导出中） | 稍后重试 |
| 410 | 410 | 导出文件已过期（超过 24h） | 重新搜索导出 |
| 502 | 502 | 上游数据源暂不可用 | 稍后重试 |

---

## 6. 令牌信息查询

```
GET /api/token-info
```

**请求**（带令牌）

```bash
curl -H "X-API-Key: <你的令牌>" https://fofa.shanshuiapi.com/api/token-info
```

**响应示例**

```json
{
  "code": 0,
  "user_name": "月卡001",
  "remark": "月卡批次20260819",
  "daily_quota": 1000,
  "today_used": 1,
  "today_remaining": 999
}
```

---

## 7. 健康检查

```
GET /health
```

返回 `{"ok":true}`，可用于连通性监控。

---

## 8. 代码示例

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

# 带导出：搜索 + 异步导出全部资产
data = search('title="login"', download=True)   # 消耗 2 次额度
task_id = data["export_task_id"]
# 轮询导出状态，直到 done
import time
while True:
    st = requests.get(f"https://fofa.shanshuiapi.com/api/export/{task_id}",
                      headers={"X-API-Key": KEY}).json()
    if st["status"] == "done":
        break
    time.sleep(3)
# 下载导出文件
r = requests.get(f"https://fofa.shanshuiapi.com/api/export/{task_id}/file",
                 headers={"X-API-Key": KEY})
open("result.csv", "wb").write(r.content)
```

### 命令行

```bash
curl -s -X POST https://fofa.shanshuiapi.com/api/search \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <你的令牌>" \
  -d '{"query":"title=\"login\"","page":1,"size":10}'
```

---

## 9. 交流群

Telegram 交流群：https://t.me/+dBrFr66Y8uo1ZDgx

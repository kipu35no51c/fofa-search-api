# 切片搜索（Slicing）进阶指南

> 适用场景：单个查询的匹配总数很大（如几百万条），而接口对**单次查询**能返回的结果数量有限制时，通过把查询拆分为多个互斥的子查询，实现"绕过单查询深度限制、拉取更多数据"。

---

## 1. 背景：单查询的 60 条硬上限

实测确认（2026-10-17，真实接口验证）：

- 每次请求的 `page × size` **不得超过 60**，即单个查询最多只能看到**前 60 条结果**：

| 请求参数 | 结果 |
|---|---|
| `page=2, size=30`（=60）| ✅ 正常 |
| `page=6, size=10`（=60）| ✅ 正常 |
| `page=2, size=31`（=62）| ❌ 502 |
| `page=7, size=10`（=70）| ❌ 502 |
| `size=100` | 截断为 50（接口限制 size ≤ 50）|

- 该限制由**上游数据源**决定，**与令牌（API Key）无关**：换任意 Key 请求同一个超限组合，同样返回 502。
- 也就是说：**单次查询拿满 60 条**，想拿第 61 条起的数据，直接翻页是翻不过去的。

---

## 2. 原理：限制按"查询语句"独立计算

关键规律：**60 条上限是按每一条查询语句独立计算的**。

```
title="login"                     → 可以看到前 60 条
title="login" && country="CN"     → 又可以看前 60 条（互不干扰）
title="login" && country="US"     → 再可以看前 60 条
```

所以：**把一个大查询拆成 N 个互斥的子查询，每个子查询都能拿到自己的 60 条结果，总产出 ≈ N × 60**。

---

## 3. 切片维度选择

切片的本质：找到一个字段，把结果集**互斥且尽量均匀**地划分。

常用维度（按推荐顺序）：

| 维度 | 语法 | 说明 |
|---|---|---|
| **国家** | `country="CN"` | 最常用，互斥性好，各国的量差异大但可接受 |
| **地区** | `region="Beijing"` | 国家量太大时再往下切 |
| **端口** | `port="443"`、`port="80"` | 适合按常见端口枚举切 |
| **组织/ASN** | `org="xxx"` | 适合资产集中的场景 |
| **IP 段** | `ip="1.2.3.0/24"` | 适合已知网段枚举 |
| **协议** | `protocol="https"` | 量不够细时可组合使用 |
| **排除链** | `country!="CN" && country!="US"` | 用于"其他剩余"兜底，保证全覆盖 |

> 组合原则：子查询之间**不允许有交集**（互斥），并集要**能覆盖原查询的全部结果**（完备）。
> 若只有部分维度枚举不完，用 `排除链` 兜底剩余部分。

---

## 4. 完整示例（Python）

以下代码把一个大查询按国家切片，逐片子查询翻页取满，最后合并去重：

```python
import requests
import time

API = "https://fofa.shanshuiapi.com/api/search"
KEY = "<你的令牌>"

def search(query, page=1, size=50):
    resp = requests.post(API, headers={"X-API-Key": KEY},
                         json={"query": query, "page": page, "size": size})
    resp.raise_for_status()
    return resp.json()

BASE_QUERY = 'title="login"'

# 1. 先探明各子查询的匹配总数（决定每片要翻几页）
countries = ["CN", "US", "JP"]   # 按需调整
slices = []
for c in countries:
    total = search(f'{BASE_QUERY} && country="{c}"', size=2).get("total", 0)
    if total:
        slices.append((c, total))
    print(f"country={c}: total={total}")

# 2. 逐片拉取：每片最多翻到 60 条（page×size ≤ 60）
seen = set()
assets = []
for c, total in slices:
    # 第一页 size=50
    data = search(f'{BASE_QUERY} && country="{c}"', page=1, size=50)
    for a in data.get("assets", []):
        key = (a.get("ip"), a.get("port"))
        if key not in seen:
            seen.add(key)
            assets.append(a)
    # 第二页 size=10（凑满 60）
    if total > 50:
        time.sleep(0.5)
        data = search(f'{BASE_QUERY} && country="{c}"', page=2, size=10)
        for a in data.get("assets", []):
            key = (a.get("ip"), a.get("port"))
            if key not in seen:
                seen.add(key)
                assets.append(a)
    time.sleep(0.5)

print(f"切片后总产出: {len(assets)} 条（单查询只能拿 60 条）")
```

### 实测效果（2026-10-17）

中性靶子查询 `title="login"`（匹配总数约 2971 万）：

| 方式 | 拿到的条数 |
|---|---|
| 单查询（不切片）| 60 条（上限）|
| 按 3 个国家切片 | **154 条（约 3 倍）** |
| 按 6 个国家切片 | 212 条（约 3.5 倍）|
| 按 省 × 端口 × ASN 细切 | 390 条（约 6.5 倍）|

> 切得越细，倍数越高；代价是**请求次数增加**（每片约 2 次请求），会消耗更多当日额度。

---

## 5. 注意事项

1. **额度消耗**：每次请求消耗 1 次额度（与返回条数无关）。切片后请求次数成倍增加，请先算好当日额度再跑。
2. **去重**：翻页/切片之间可能存在少量重复（实测约 5% 跨页重复），建议按 `(ip, port)` 或 `host` 去重。
3. **限速**：接口有 QPS 限制（超限返回 429），循环请求时请加适当间隔（如 0.5~1 秒）。
4. **互斥性**：切片维度必须互斥，否则同一资产会在多个子查询中重复出现，浪费额度。
5. **完备性**：如果枚举维度覆盖不全，最后加一个 `排除链` 子查询兜底，保证不丢数据。
6. **502 处理**：单条子查询若返回 502，通常是该子查询翻页超深（page×size>60），检查参数即可。

---

## 6. 快速排查

| 现象 | 原因 | 处理 |
|---|---|---|
| `page=2, size=31` 返回 502 | page×size 超 60 | 调整 size 使乘积 ≤ 60 |
| 换 Key 后仍 502 | 限制在上游，与 Key 无关 | 用切片 |
| `size=100` 只回 50 条 | 接口上限 size ≤ 50 | 翻页取，别加 size |
| 429 | 请求过快 | 加间隔重试 |
| 重复数据多 | 切片/翻页重叠 | 按 (ip,port) 去重 |

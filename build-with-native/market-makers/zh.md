---
description: External Market Maker Orderbook Push API 中文对照版。如有歧义，以英文版为准。
layout:
  width: wide
---

# 中文对照版

{% hint style="info" %}
Version 0.2 · 2026-11-13。本节与 [英文版](README.md) 同步维护（内容一一对应）；如有歧义，以英文版为准。[限额](#限额)表中的默认值为暂定值，最终以 onboarding 文件为准。
{% endhint %}

## 概述

Native 的外部做市商集成，让你通过**推送订单簿**为自己被分配的交易对在 Native 做市。

- **你推书，我们报价。** 你推送自己的 bid 和 ask；Native 根据你最新有效的订单簿为每一笔 firm quote 定价、加上 Native 自己的费率并完成签名。
- **仅入站。** 报价链路上 Native 绝不回调你——不拉取价格、没有 last look、没有签名请求。
- **你是唯一价格源。** 分配给你的交易对，你的书就是这本书。书缺失、过期或被撤销时，该交易对直接停止报价——没有任何兜底。
- **两种传输、同一套语义。** 每次更新走一次 HTTPS 请求，或通过 WebSocket 流式推送。订单簿格式、校验规则、顺序保证完全相同。

{% hint style="warning" %}
**无 last look——已签名的报价是终局的。** firm quote 在请求时刻根据你的书定价，并在其全部有效期内保持可执行（该时长跟随上游要求、由 Native 全局设定：MM 不可按 pair 单独指定，需要调整请通知 Native 全局修改）。新推送、书过期、撤书或暂停只会阻止新报价，绝不会撤销已签名的报价。你必须按已签价格接受成交，直到该报价截止。请据此定价，并谨慎选择过期时间。
{% endhint %}

## 开始之前

Native ops 会按做市商账户提供：

| 项目 | 说明 |
| --- | --- |
| API key | 用于 `Apikey` 请求头的值 |
| IP 白名单 | 允许使用该 key 的来源 IP |
| 分配的交易对 | 每个交易对的链、base token、quote token，以及你必须推送的方向 |
| 交易对限额 | 最小档位量、档位数上限、费率 |
| Token 映射 | 每个交易对的合约地址，以及 base/quote 的确切 symbol 字符串（严格区分大小写） |

所有流量走 TLS（HTTPS / WSS）。本文档以 `https://api.native.org` 与 `wss://api.native.org` 作为示例 URL——请使用 onboarding 文件中的确切 URL。

## 核心概念

### 订单簿（Book）

订单簿是你在某个交易对上的 bid 和 ask。

| 边 | 含义 | 排序 |
| --- | --- | --- |
| `bids` | 你**买入 base**（支付 quote）的档位 | 价格降序，最优在前 |
| `asks` | 你**卖出 base**（收到 quote）的档位 | 价格升序，最优在前 |

一个档位是：

```json
{ "price": "0.98", "qty": "12000" }
```

- `price`——每 1 个 base 对应的 quote 数量。
- `qty`——该档位的 base 数量。**逐档给出，不是累计值。**
- 两者均为十进制字符串，精度在传输中不丢失。
- 推你自己的价格。Native 报价时会在其上加自己的费率；你不需要、也不应该为 Native 额外加价。
- 最优 bid 必须严格低于最优 ask——交叉的书会被拒绝。

### 交易对与方向

订单簿用 `chain_id` 加两个 token 标识交易对。**所有字段均为必填且都会校验。**EVM 地址必须为**小写** hex——checksum 或混合大小写形式会被拒绝（`invalid_field`）；symbol **严格区分大小写**，必须与注册 symbol 完全一致（`MEME` ≠ `meme`），地址→symbol 映射见 onboarding 文件，`GET /v1/orderbook` 会对每个当前生效的 pair 回显该映射。必须严格按分配方向推送——颠倒被拒（`orientation_mismatch`）、symbol 不符被拒（`symbol_mismatch`）。系统不会替你静默反转或纠正。

### 整本替换，不做增量

每本被接受的书会**完整**替换你该交易对此前的书：

| 你推送的内容 | 效果 |
| --- | --- |
| 两边都非空 | 该交易对的书被整体替换 |
| 一边为空（`"bids": []`） | 替换整本书；该边停止报价 |
| 两边都为空 | 撤销整个交易对 |
| 请求中未提到的交易对 | 其书保持不变 |

没有增量、没有档位 id、没有部分更新——每次推送都携带完整的书。

推空边或空书就是**撤单**。一边为空：撤掉该边，另一边继续报价；双边皆空：撤销整个交易对。撤销立即生效且**没有过期时间**——持续到该交易对的下一次推送；空书携带的 `expire_at_ms` 会被忽略。

### 原子性：整体成功或整体失败

每次推送都是完整的一组书，一个请求就是一个完整的工作单元：

- 一个请求可以携带**不同链、不同交易对**的书——每本书自带 `chain_id`。
- 请求是**原子的**：要么每本书都通过校验、全部落地，要么一本都不落地；不存在中间的“部分成功”状态。
- nonce **只在请求落地时消费**。失败的请求零副作用——什么都没变，包括 nonce。用同一或更大的 nonce 重推即可。
- 请求落地时，响应只是纯确认——不返回逐书结果；失败时，`results` 列出**每一本书**：`rejected`（校验失败，附原因）或 `skipped`（这本书本身没问题，但整单未应用）。
- 单交易对推送就是 orderbooks 数组只有一个元素的同一协议。将来若放开请求内部分成功，**响应结构不变**——按 `results[].status` 逐书解析即为向前兼容。

### 过期时间：`expire_at_ms`

- 毫秒 Unix 时间戳的绝对时刻——你的书停止为新报价定价的那一刻。
- 由 Native 的时钟读取。请保持主机 NTP 对时；每个 ack 都带 `server_time_ms`，便于你观察时钟漂移。
- 收到时刻的剩余寿命必须**不少于 5 秒**。没有上限——更长的过期只会延长你自己的陈旧风险。
- 新推送会立即替换当前的书，无论旧书写了什么过期时间。

### Nonce

每个请求携带整数 `nonce`：

- 必须**严格大于**你的做市商账户上一次**已应用（applied）**请求的 nonce。
- HTTP 与你的**所有** WebSocket 连接共用同一个计数器。并发推送会在计数器上竞争——输者得到 `stale_nonce`。多条连接做高可用没问题；同时推送就预期竞争。
- 携带旧 nonce 的迟到推送会被拒绝（`stale_nonce`）；它永远不会覆盖更新的书。
- 计数器仅在请求**应用**时前进。校验失败的请求零副作用——修好后用同一或更大的 nonce 重推即可。

### 裁剪（Trimming）

Native 会按你的交易对剩余的仓位空间裁剪书的每一边；某边空间归零时整边移除（另一边的成交可以带回空间）。推送响应不携带任何裁剪数据——各 pair 仍可报价的深度请用 `GET /v1/orderbook` 查询；实际生效的裁剪发生在报价时刻，以当时的空间为准。若裁剪后某边剩余深度低于该 pair 的最小量，该边等效移除。

## HTTP API

### 推送订单簿 — `POST /v1/orderbook`

请求头：

| Header | 值 |
| --- | --- |
| `Apikey` | 你的 API key |
| `Content-Type` | `application/json` |

请求体：

```json
{
  "nonce": 1731318400123,
  "orderbooks": [
    {
      "chain_id": 56,
      "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
      "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
      "bids": [
        { "price": "0.98", "qty": "12000" },
        { "price": "0.97", "qty": "25000" }
      ],
      "asks": [
        { "price": "1.02", "qty": "8000" },
        { "price": "1.03", "qty": "15000" }
      ],
      "expire_at_ms": 1731318406000,
      "book_id": "b-000123"
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `nonce` | integer | 是 | 做市商级单调递增计数器（见 Nonce） |
| `orderbooks` | array | 是 | 每请求 1–20 本 |
| `orderbooks[].chain_id` | integer | 是 | 交易对所在 EVM 链 id |
| `orderbooks[].base.address` | string | 是 | base token 合约地址，EVM hex；**必须小写** |
| `orderbooks[].quote.address` | string | 是 | quote token 合约地址，EVM hex；**必须小写** |
| `orderbooks[].base.symbol`、`orderbooks[].quote.symbol` | string | 是 | 注册 symbol，**严格区分大小写**——用映射（onboarding 文件 / `GET /v1/orderbook`）中的确切字符串 |
| `orderbooks[].bids`、`orderbooks[].asks` | array | 是 | 档位，最优价在前；可为空数组以撤销该边 |
| `orderbooks[].expire_at_ms` | integer | 是 | 绝对过期时刻，毫秒 Unix；不早于收到时刻 + 5 秒 |
| `orderbooks[].book_id` | string | 是 | 你给这本书的 id（非空，≤ 64 字符，**请求内唯一**）；会回显在 ack 与你的对账单中 |

响应——推送响应不携带任何订单簿与裁剪数据：成功时只是纯确认，`results` 仅在失败时出现——只列问题书，按请求顺序、以 `book_id` 关联（必填、请求内唯一）。

```json
{
  "applied": true,
  "trace_id": "01JD8Z9W3C",
  "server_time_ms": 1731318400450
}
```

| 字段 | 说明 |
| --- | --- |
| `applied` | `true`——全部书通过校验并已落地，nonce 已消费；`false`——什么都没落地，nonce **未**消费 |
| `server_time_ms` | 构造 ack 时 Native 的时钟 |
| `trace_id` | 本次推送的追踪 id |
| `results` | 仅当 `applied` 为 `false` 时出现；只列问题书，按请求顺序、以 `book_id` 关联 |
| `results[].status` | `rejected`——该书校验失败（见 `reason`）；`skipped`——该书本身合法，但整单未应用（`atomic_abort`）或 nonce 门槛未过（`stale_nonce`）。当前不会出现 `accepted`（整单落地时不返回 results）；将来放开部分成功后可能出现 |
| `results[].reason`、`results[].detail` | 出现在 rejected/skipped。`detail` 指明违规档位与数值（如 `level[3] qty 50 < min 100`）；rejected 会重复 pair 身份（`chain_id`、`base`、`quote`），失败信息自含可定位 |

`applied` 为 `false` 时什么都没有变——之前的每本书继续可报价直到自然过期，且同一 nonce 可以复用。rejected 会重复 pair 身份并精确描述问题；skipped 保持精简。示例——一本书不合格，整单作废：

```json
{
  "applied": false,
  "trace_id": "01JD8ZA112F",
  "server_time_ms": 1731318400900,
  "results": [
    {
      "book_id": "b-000123",
      "status": "skipped",
      "reason": "atomic_abort"
    },
    {
      "chain_id": 137,
      "base": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "quote": "0xcccccccccccccccccccccccccccccccccccccccccc",
      "book_id": "b-000124",
      "status": "rejected",
      "reason": "crossed_book",
      "detail": "best bid 1.05 >= best ask 1.02"
    }
  ]
}
```

curl 快速测试：

```bash
curl -X POST "https://api.native.org/v1/orderbook" \
  -H "Apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "nonce": 1731318400123,
        "orderbooks": [{
          "chain_id": 56,
          "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
          "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
          "bids": [{ "price": "0.98", "qty": "12000" }],
          "asks": [{ "price": "1.02", "qty": "8000" }],
          "expire_at_ms": 1731318406000,
          "book_id": "b-000123"
        }]
      }'
```

### 查询当前订单簿 — `GET /v1/orderbook`

返回当前 api_key 对应做市商**所有当前生效（live）的书**——服务端最新持有的档位，加上当前仍可报价的深度。只返回当前生效的书：已过期、已撤销、从未推送的 pair 不会出现在响应中。这是唯一的查询端点；pair 约束、费率与权威映射以 onboarding 文件为准。本端点独立限频：**每做市商 5 请求/秒**，与推送限频分开。

```json
{
  "server_time_ms": 1731318401000,
  "trace_id": "01JD8ZK444T",
  "orderbooks": [
    {
      "chain_id": 56,
      "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
      "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
      "bids": [ { "price": "0.98", "qty": "12000" } ],
      "asks": [ { "price": "1.02", "qty": "8000" } ],
      "nonce": 1731318400123,
      "book_id": "b-000123",
      "recv_time_ms": 1731318400450,
      "expire_at_ms": 1731318406000,
      "quoted_bid_base_qty": "24800",
      "quoted_ask_base_qty": "20900"
    }
  ]
}
```

`bids`/`asks` 为服务端当前持有的档位（过校验的原始值，未裁剪）；`quoted_*` 为当前仍可报价的深度——**裁剪反馈只在这里看，推送响应不携带任何深度数字**。对响应中每个 pair，此处展示的确切 `symbol` 字符串（严格区分大小写）与方向即推送必须匹配的值；未出现在响应中的 pair，以 onboarding 文件的映射为准。

### HTTP 状态码

| 码 | 含义 | 处理 |
| --- | --- | --- |
| 200 | 请求已处理——看 `applied`；`results` 仅失败时出现 | — |
| 400 | JSON 解析失败，或请求级必填字段缺失/非法 | 修正指明的字段 |
| 401 | API key 缺失、未知或已停用 | 检查 `Apikey` 请求头 |
| 403 | 来源 IP 不在白名单 | 联系 ops |
| 413 | 请求体超过 1 MiB 或超过 20 本书 | 拆分请求 |
| 429 | 超过每秒 10 次限制 | 退避重试；参考响应体中的 `retry_after_ms` |

## WebSocket 流

持续推送场景可保持长连 WebSocket，以帧为单位流式推送——允许多条连接做高可用，上限见下方规则。流是一个独立完整的接口——本节完整叙述。

**语义**——以下规则适用于每一个 push 帧：

- 每本书都是该交易对的**完整订单簿**：逐档 `price`/`qty`（非累计），bids 价格降序最优在前、asks 价格升序，最优 bid 严格低于最优 ask；均为十进制字符串。
- 每个交易对按分配方向推送；EVM 地址必须小写、symbol 严格区分大小写——颠倒被拒（`orientation_mismatch`）、symbol 不符被拒（`symbol_mismatch`）。
- 被接受的书**完整**替换该交易对此前的书；一边为空即撤销该边；双边皆空撤销交易对；帧中未提到的交易对保持原书。
- 每本书携带绝对 `expire_at_ms`（毫秒 Unix，以 Native 时钟为准）；收到时剩余寿命不少于 5 秒。
- 帧是**原子的**：要么每本书都通过校验、全部落地，要么一本都不落地。nonce 仅在帧落地时消费；失败的帧零副作用。`nonce` 必须严格大于上一次**已应用**的值——与 HTTP 共用一个计数器。
- **限额**一节的尺寸与频率限制同样适用于帧；拒绝码见**拒绝原因**一节。

**连接**——`wss://api.native.org/v1/ws`，升级请求携带 `Apikey` 请求头。IP 白名单同样生效。仅支持服务端客户端（浏览器无法为 WebSocket 设置自定义请求头）。

**发送**帧（JSON 文本）。Push 帧——完整 schema：

```json
{
  "op": "push",
  "nonce": 1731318400124,
  "orderbooks": [
    {
      "chain_id": 56,
      "base":  { "address": "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", "symbol": "MEME" },
      "quote": { "address": "0x55d398326f99059ff775485246999027b3197955", "symbol": "USDT" },
      "bids": [
        { "price": "0.98", "qty": "12000" },
        { "price": "0.97", "qty": "25000" }
      ],
      "asks": [
        { "price": "1.02", "qty": "8000" },
        { "price": "1.03", "qty": "15000" }
      ],
      "expire_at_ms": 1731318406000,
      "book_id": "b-000124"
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `op` | string | 是 | `"push"` |
| `nonce` | integer | 是 | 做市商级单调计数器；严格大于上一次已应用的值 |
| `orderbooks` | array | 是 | 每帧 1–20 本 |
| `orderbooks[].chain_id` | integer | 是 | 交易对所在 EVM 链 id |
| `orderbooks[].base.address` | string | 是 | base token 合约地址，EVM hex；**必须小写** |
| `orderbooks[].quote.address` | string | 是 | quote token 合约地址，EVM hex；**必须小写** |
| `orderbooks[].base.symbol`、`orderbooks[].quote.symbol` | string | 是 | 注册 symbol，**严格区分大小写**——用映射（onboarding 文件 / `GET /v1/orderbook`）中的确切字符串 |
| `orderbooks[].bids`、`orderbooks[].asks` | array | 是 | 档位，最优价在前；可为空数组以撤销该边 |
| `orderbooks[].expire_at_ms` | integer | 是 | 绝对过期时刻，毫秒 Unix；不早于收到时刻 + 5 秒 |
| `orderbooks[].book_id` | string | 是 | 你给这本书的 id（非空，≤ 64 字符，**帧内唯一**）；会回显在 ack 与你的对账单中 |

Ping 帧——保活。两种机制均可且都有应答：本应用层帧（服务端回 `pong`，可选 `ts` 字段原样回显，供你测 RTT），或 WebSocket 协议层 Ping 控制帧（服务端自动回 Pong 控制帧）：

```json
{ "op": "ping" }
```

**接收**帧。Ack 帧——每个 push 帧恰一个，按推送顺序返回。已应用：

```json
{
  "type": "ack",
  "nonce": 1731318400124,
  "applied": true,
  "server_time_ms": 1731318400450,
  "trace_id": "01JD8Z9W3C"
}
```

未应用——一本书不合格，整帧作废：

```json
{
  "type": "ack",
  "nonce": 1731318400125,
  "applied": false,
  "server_time_ms": 1731318400900,
  "trace_id": "01JD8ZA112F",
  "results": [
    {
      "book_id": "b-000124",
      "status": "skipped",
      "reason": "atomic_abort"
    },
    {
      "chain_id": 137,
      "base": "0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "quote": "0xcccccccccccccccccccccccccccccccccccccccccc",
      "book_id": "b-000125",
      "status": "rejected",
      "reason": "crossed_book",
      "detail": "best bid 1.05 >= best ask 1.02"
    }
  ]
}
```

| 字段 | 说明 |
| --- | --- |
| `type` | 恒为 `ack` |
| `nonce` | 回显 push 帧的 `nonce` |
| `applied` | `true`——全部书通过校验并已落地，nonce 已消费；`false`——什么都没落地，nonce **未**消费 |
| `server_time_ms` | 构造 ack 时 Native 的时钟 |
| `trace_id` | 本帧的追踪 id |
| `results` | 仅当 `applied` 为 `false` 时出现；只列问题书，按推送顺序、以 `book_id` 关联 |
| `results[].status` | `rejected`——该书校验失败（见 `reason`）；`skipped`——该书本身合法，但整帧未应用（`atomic_abort`）或 nonce 门槛未过（`stale_nonce`）。当前不会出现 `accepted`（整帧落地时不返回 results）；将来放开部分成功后可能出现 |
| `results[].reason`、`results[].detail` | 出现在 rejected/skipped。`detail` 指明违规档位与数值（如 `level[3] qty 50 < min 100`）；rejected 会重复 pair 身份（`chain_id`、`base`、`quote`），失败信息自含可定位 |

Pong 帧——对 `ping` 的应答：

```json
{ "type": "pong" }
```

Error 帧——协议层问题（帧超限、JSON 错误、未知 `op` 等）；连接保持不断：

```json
{ "type": "error", "code": "frame_too_large", "msg": "frame exceeds 64 KiB" }
```

规则：

- **连接数上限，允许高可用。** 每 maker 最多 **3** 条连接（默认值）——足够主备/双活部署；超出将被关闭，关闭码 `4005`。
- **服务端只看每个瞬间的结果。** 任一时刻，某 pair 的书就是最后一笔落地请求携带的书——不感知连接、连接之间无顺序保证。你自己的实例乱推、竞争导致的交错结果，责任在 maker 自己。
- **空闲上限 60 秒**，定义为无任意类型的入站帧——推送本身就是存活信号，持续推送时无需 ping；不推送时至少每 30 秒 ping 一次。空闲超时以 `4003` 关闭连接。两种 ping 均计入入站速率限额（每 30 秒一次可忽略）。
- 单帧最大 64 KiB。
- 每连接入站消息速率 ≤ **25 msg/s**（含所有帧与 ping）——超限以 `4004` 关闭连接。
- ack 只保证**同一连接内**按推送顺序返回。

示例（Node.js）：

```javascript
const WebSocket = require("ws");

const ws = new WebSocket("wss://api.native.org/v1/ws", {
  headers: { "Apikey": "YOUR_API_KEY" },
});

ws.on("open", () => {
  ws.send(JSON.stringify({
    op: "push",
    nonce: 1731318400124,
    orderbooks: [{
      chain_id: 56,
      base:  { address: "0x698250914c54eb2f1bb7e0eb1e6a24c0f9b1c1a1", symbol: "MEME" },
      quote: { address: "0x55d398326f99059ff775485246999027b3197955", symbol: "USDT" },
      bids: [{ price: "0.98", qty: "12000" }],
      asks: [{ price: "1.02", qty: "8000" }],
      expire_at_ms: Date.now() + 5000,
      book_id: "b-000124",
    }],
  }));
});

ws.on("message", (raw) => console.log(raw.toString()));

// 保活
setInterval(() => ws.send(JSON.stringify({ op: "ping" })), 25000);
```

关闭码：

| 码 | 含义 |
| --- | --- |
| 4001 | 鉴权失败 |
| 4002 | 来源 IP 不在白名单 |
| 4003 | 空闲超时 |
| 4004 | 协议违规（含超过入站消息速率） |
| 4005 | 连接数达到上限（每 maker） |
| 1001 | 服务端重启——重连并继续推送；书从你的下一次推送恢复 |

## 拒绝原因

| 码 | 含义 | 常见处理 |
| --- | --- | --- |
| `stale_nonce`（请求级） | `nonce` 不大于上一次**已应用**的值；什么都没被动过 | 用更大的 nonce 重推 |
| `atomic_abort`（状态为 `skipped`） | 该书本身合法但未应用：同一请求里另一本书被拒 | 修好被拒的书，整单重推 |
| `invalid_field`（请求级或 book 级） | 必填字段缺失、为空或格式非法（地址非 hex、地址非小写、`book_id` 为空、`nonce` 非整数、symbol 缺失等） | 修正指明的字段；什么都没被动过 |
| `symbol_mismatch` | base 或 quote symbol 与该地址注册的 symbol 不完全一致（区分大小写） | 用映射中的确切 symbol |
| `pair_not_found` | (chain, base, quote) 不是已上线的交易对 | 核对地址与链 id |
| `pair_not_owned` | 该交易对未分配给你 | 联系 ops |
| `orientation_mismatch` | token 方向颠倒 | 按分配方向推送 |
| `duplicate_pair` | 同一交易对在一个请求中出现两次 | 每个交易对只发一本书 |
| `duplicate_book_id` | 同一 `book_id` 在一个请求中出现两次 | 用唯一 id——结果按 `book_id` 关联 |
| `expire_time_below_min` | 收到时剩余寿命不足 5 秒 | 推更晚的 `expire_at_ms`；检查对时 |
| `invalid_level` | 价格或数量非正、无法解析，或十进制字符串超过 64 字符 | 修正该档 |
| `levels_out_of_order` | bids 未降序 / asks 未升序，或同边出现相同价格 | 排序并合并 |
| `crossed_book` | 最优 bid ≥ 最优 ask | 重新居中 |
| `too_many_levels` | 某边档数超过上限 | 削减梯子 |
| `level_below_min_size` | 某档低于交易对的最小档量 | 补足或去掉该档 |

## 限额

| 项 | 默认值 |
| --- | --- |
| 推送频率 | 每做市商 10 请求/秒（HTTP + WS 合计） |
| 每请求书本数 | 20 |
| 每边档数 | 100 |
| 请求体 | 1 MiB |
| WS 帧 | 64 KiB |
| 每 maker WS 连接数上限 | 3 |
| WS 空闲超时 | 60 秒 |
| 最小剩余书寿命 | 5 秒 |
| 每连接 WS 入站消息 | 25 msg/s |
| 十进制字符串最大长度 | 64 字符 |
| 查询限频（GET） | 每做市商 5 请求/秒 |

默认值可能在试运行前后按做市商或交易对调整；以你的 onboarding 文件为准。

## 补充说明

- **节奏。** 一本书在其 `expire_at_ms` 之前都可为报价定价。停止推送后，最后一本书过期时交易对即停止报价——过期时间就是你的陈旧度控制手段。
- **成交不会通过本 API 推送给你。** 请通过池子链上的 `RFQTrade` 事件、按你交易对的 token 过滤来跟踪成交；仓位、限额与费用由 Native ops 的对账单覆盖。
- **`book_id` 值得写。** 它会进入你的成交对账单，每推一个 id，每笔成交都能追溯到确切的那本书。
- **时钟。** `expire_at_ms` 以 Native 的时钟为准。请 NTP 对时，并关注 ack 里的 `server_time_ms`。
- **用空推送撤单。** 空一边撤该边；双边皆空撤整个交易对。撤销立即生效、无过期、持续到下一次推送——比拉宽价差明确无歧义。
- **原子重试。** `applied: false` 即零副作用——修好不合格的书后整单重推，同一或更大的 nonce 均可。

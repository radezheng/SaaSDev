# Azure Marketplace SaaS（Transactable）API 参考（按时序图整理）

- English version: [API_REF_EN.md](./API_REF_EN.md)
- Implementation guide (EN): [README_EN.md](./README_EN.md)

> 目标：把你方 SaaS 需要实现/调用的 Marketplace 接口，按“首次购买/订阅变更/Webhook/按量计费”三条时序串起来，方便开发直接对接。
>
> 适用：Commercial Marketplace 的 **SaaS Offer（Transactable）**，对接 **SaaS Fulfillment APIs v2** +（可选）**Marketplace Metering APIs**。

## 0. 统一约定

- **Base URL**：`https://marketplaceapi.microsoft.com`
- **api-version**：`2018-08-31`
- **通用 Header（建议每次请求都带）**
  - `Content-Type: application/json`
  - `x-ms-requestid: <guid>`（客户端生成）
  - `x-ms-correlationid: <guid>`（同一业务流保持一致，便于追踪）
- **认证 Header**（除 Resolve 的 marketplaceToken 以外，几乎所有调用都需要）
  - `Authorization: Bearer <access_token>`

### 0.1 两种“token”的区别（务必区分）

- **Marketplace purchase identification token**：来自客户跳转到你方 Landing Page URL 时的 `?token=...`，你方只能用它去调用 **Resolve**（通过 `x-ms-marketplace-token` header 传入）。
- **Microsoft Entra access token（publisher authorization token）**：你方服务端通过 OAuth2 client credentials 获取，用于调用 **Fulfillment APIs / Operations APIs / Metering APIs**（通过 `Authorization: Bearer ...` 传入）。

官方参考：
- SaaS Fulfillment Subscription APIs v2：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api
- SaaS Fulfillment Operations APIs v2：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api
- SaaS Webhook：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook
- Marketplace Metering APIs：https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis

---

## 1. 你方服务端如何拿到 `access_token`（Publisher authorization token）

你方服务端调用 Fulfillment/Metering API 使用 **Microsoft Entra ID OAuth2 client credentials** 获取访问令牌。

### 1.1 Token 请求

- **POST** `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`
- **Content-Type**：`application/x-www-form-urlencoded`

#### Request Body（form）

| 字段 | 必填 | 类型 | 说明 |
|---|---:|---|---|
| `grant_type` | 是 | string | 固定 `client_credentials` |
| `client_id` | 是 | string | 你在 Entra App Registration 的应用（客户端）ID |
| `client_secret` | 是 | string | client secret |
| `scope` | 是 | string | 固定使用 `20e940b3-4c77-4b0b-9a53-9e16a1b010a7/.default` |

#### Response（200）

| 字段 | 类型 | 说明 |
|---|---|---|
| `token_type` | string | `Bearer` |
| `expires_in` | string/int | 通常 3600 秒 |
| `access_token` | string | 后续放到 `Authorization` header |

#### 完整示例（获取 Entra access_token）

请求：

```bash
curl -sS -X POST \
  "https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=${CLIENT_ID}" \
  --data-urlencode "client_secret=${CLIENT_SECRET}" \
  --data-urlencode "scope=20e940b3-4c77-4b0b-9a53-9e16a1b010a7/.default"
```

响应（示例）：

```json
{
  "token_type": "Bearer",
  "expires_in": "3600",
  "ext_expires_in": "0",
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIs..."
}
```

官方参考：
- 注册 SaaS 并获取 token：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-registration

---

## 2. 首次购买（Landing Page）— Resolve → Provision（你方）→ Activate

### 2.1 Resolve：把 Marketplace `token` 换成持久 `subscriptionId`

当客户完成购买并跳转到你方 Landing Page 时，URL 会携带 `token=...`（purchase identification token）。你方需调用 Resolve 换取 `subscriptionId` 等信息。

- **POST** `/api/saas/subscriptions/resolve?api-version=2018-08-31`

#### Request Headers

| Header | 必填 | 说明 |
|---|---:|---|
| `Authorization` | 是 | 你方服务端 access token |
| `x-ms-marketplace-token` | 是 | Landing Page URL 中的 `token`（注意 URL decode 后再传） |

#### 完整示例（Resolve）

请求：

```bash
curl -sS -X POST \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/resolve?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "x-ms-marketplace-token: ${MARKETPLACE_TOKEN}"
```

响应（200 示例，来自官方文档）：

```json
{
  "id": "<guid>",
  "subscriptionName": "Contoso Cloud Solution",
  "offerId": "offer1",
  "planId": "silver",
  "quantity": 20,
  "subscription": {
    "id": "<guid>",
    "publisherId": "contoso",
    "offerId": "offer1",
    "name": "Contoso Cloud Solution",
    "saasSubscriptionStatus": " PendingFulfillmentStart ",
    "beneficiary": {
      "emailId": "test@test.com",
      "objectId": "<guid>",
      "tenantId": "<guid>",
      "puid": "<ID of the user>"
    },
    "purchaser": {
      "emailId": "test@test.com",
      "objectId": "<guid>",
      "tenantId": "<guid>",
      "puid": "<ID of the user>"
    },
    "planId": "silver",
    "term": {
      "termUnit": "P1M",
      "startDate": "2022-03-07T00:00:00Z",
      "endDate": "2022-04-06T00:00:00Z"
    },
    "autoRenew": true,
    "isTest": false,
    "isFreeTrial": false,
    "allowedCustomerOperations": ["Read", "Update", "Delete"],
    "sandboxType": "None",
    "lastModified": "0001-01-01T00:00:00",
    "quantity": 5,
    "sessionMode": "None"
  }
}
```

#### Response（200）核心字段

> Resolve 返回结构里既有顶层字段，也包含 `subscription` 子对象（字段更全）。你方一般至少需要：`id`（subscriptionId）、`offerId`、`planId`、`quantity`、`saasSubscriptionStatus`。

| 路径 | 类型 | 说明 |
|---|---|---|
| `id` | string(guid) | **订阅 ID（subscriptionId）** |
| `offerId` | string | offer 标识 |
| `planId` | string | plan 标识 |
| `quantity` | number | seat 数（如果非 per-seat 可能为空） |
| `subscription.saasSubscriptionStatus` | string | `PendingFulfillmentStart` / `Subscribed` / `Suspended` / `Unsubscribed` |
| `subscription.allowedCustomerOperations` | string[] | `Read`/`Update`/`Delete`（CSP 场景可能只允许 Read） |

#### 常见状态码

- `200`：成功
- `400`：token 缺失/无效/过期（token 有效期 24h）
- `401/403`：access token 无效或注册不匹配

官方参考：
- Resolve：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#resolve-a-purchased-subscription

### 2.2 Provision（你方业务，不是 Marketplace API）

你方需要基于 Resolve 返回的 `subscriptionId` 在你方系统做“订阅绑定/开通/对账”：

- 建议最小落库字段：`subscriptionId`、`offerId`、`planId`、`quantity`、`beneficiary.tenantId`、`purchaser/beneficiary`、当前状态、最近一次 operationId（如果有）
- 幂等：同一个 `subscriptionId` 重复 resolve/回调必须不会重复创建

### 2.3 Activate：告知 Marketplace“已完成配置，开始计费”

- **POST** `/api/saas/subscriptions/{subscriptionId}/activate?api-version=2018-08-31`

#### Request Headers

| Header | 必填 | 说明 |
|---|---:|---|
| `Authorization` | 是 | 你方服务端 access token |

#### 完整示例（Activate）

请求：

```bash
curl -sS -X POST \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/activate?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -i
```

响应：

- `200 OK`（无响应 body）

#### Response

- `200`：请求已接收（无 body）。可随后通过 Get Subscription/Operations 观察最终状态。
- 失败时常见：`400`（Suspended）、`404`（Unsubscribed）

官方参考：
- Activate：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#activate-a-subscription

---

## 3. 订阅查询与变更（Subscription APIs v2）

### 3.1 Get Subscription：查询单个订阅

- **GET** `/api/saas/subscriptions/{subscriptionId}?api-version=2018-08-31`

#### 完整示例（Get Subscription）

请求：

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

响应（200 示例，来自官方文档）：

```json
{
  "id": "<guid>",
  "name": "Contoso Cloud Solution",
  "publisherId": "contoso",
  "offerId": "offer1",
  "planId": "silver",
  "quantity": 10,
  "beneficiary": {
    "emailId": "test@contoso.com",
    "objectId": "<guid>",
    "tenantId": "<guid>",
    "puid": "<ID of the user>"
  },
  "purchaser": {
    "emailId": "test@test.com",
    "objectId": "<guid>",
    "tenantId": "<guid>",
    "puid": "<ID of the user>"
  },
  "allowedCustomerOperations": ["Read", "Update", "Delete"],
  "sessionMode": "None",
  "isFreeTrial": false,
  "autoRenew": true,
  "isTest": false,
  "sandboxType": "None",
  "created": "2022-03-01T22:59:45.5468572Z",
  "lastModified": "0001-01-01T00:00:00",
  "saasSubscriptionStatus": " Subscribed ",
  "term": {
    "startDate": "2022-03-04T00:00:00Z",
    "endDate": "2022-04-03T00:00:00Z",
    "termUnit": "P1M"
  }
}
```

#### Response（200）核心字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string(guid) | subscriptionId |
| `offerId` / `planId` | string | 当前购买的 offer/plan |
| `quantity` | number | seat 数（可能为空） |
| `saasSubscriptionStatus` | string | 订阅状态 |
| `term.termUnit` | string | `P1M` / `P1Y` 等 |
| `term.startDate` / `endDate` | string(datetime) | 仅 Active 后可用 |

官方参考：
- Get Subscription：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#get-subscription

### 3.2 List Subscriptions：分页列出订阅

- **GET** `/api/saas/subscriptions?api-version=2018-08-31[&continuationToken=...]`

#### 完整示例（List Subscriptions）

请求（第一页，不带 continuationToken）：

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

响应（200 示例，来自官方文档）：

```json
{
  "subscriptions": [
    {
      "id": "<guid>",
      "name": "Contoso Cloud Solution",
      "publisherId": "contoso",
      "offerId": "offer1",
      "planId": "silver",
      "quantity": 10,
      "beneficiary": {
        "emailId": " test@contoso.com",
        "objectId": "<guid>",
        "tenantId": "<guid>",
        "puid": "<ID of the user>"
      },
      "purchaser": {
        "emailId": " test@contoso.com",
        "objectId": "<guid>",
        "tenantId": "<guid>",
        "puid": "<ID of the user>"
      },
      "term": {
        "startDate": "2022-03-04T00:00:00Z",
        "endDate": "2022-04-03T00:00:00Z",
        "termUnit": "P1M"
      },
      "autoRenew": true,
      "allowedCustomerOperations": ["Read", "Update", "Delete"],
      "sessionMode": "None",
      "isFreeTrial": true,
      "isTest": false,
      "sandboxType": "None",
      "saasSubscriptionStatus": "Subscribed"
    }
  ],
  "@nextLink": "https://marketplaceapi.microsoft.com/api/saas/subscriptions/?continuationToken=...&api-version=2018-08-31"
}
```

#### Response（200）

| 字段 | 类型 | 说明 |
|---|---|---|
| `subscriptions` | array | 订阅列表（包含 id/name/offerId/planId/quantity/beneficiary/purchaser/term/status 等） |
| `@nextLink` | string | 下一页链接（含 continuationToken） |

官方参考：
- List Subscriptions：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#get-list-of-all-subscriptions

### 3.3 List Available Plans：查询允许切换的 Plan

- **GET** `/api/saas/subscriptions/{subscriptionId}/listAvailablePlans?api-version=2018-08-31[&planId=...]`

#### 完整示例（List Available Plans）

请求：

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/listAvailablePlans?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

响应（200 示例，来自官方文档）：

```json
{
  "plans": [
    {
      "planId": "Platinum001",
      "displayName": "plan display name",
      "isPrivate": true,
      "description": "plan description",
      "minQuantity": 5,
      "maxQuantity": 100,
      "hasFreeTrials": false,
      "isPricePerSeat": true,
      "isStopSell": false,
      "market": "US",
      "planComponents": {
        "recurrentBillingTerms": [
          {
            "currency": "USD",
            "price": 1,
            "termUnit": "P1M",
            "termDescription": "term description",
            "meteredQuantityIncluded": [
              {
                "dimensionId": "Dimension001",
                "units": "Unit001"
              }
            ]
          }
        ],
        "meteringDimensions": [
          {
            "id": "MeteringDimension001",
            "currency": "USD",
            "pricePerUnit": 1,
            "unitOfMeasure": "unitOfMeasure001",
            "displayName": "unit of measure display name"
          }
        ]
      },
      "sourceOffers": [
        {
          "externalId": "<guid>"
        }
      ]
    }
  ]
}
```

> 常用于你方 UI 展示“可切换的计划”。返回会包含 public/private plan 以及（可选）private offer 关联信息。

官方参考：
- List Available Plans：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#list-available-plans

### 3.4 Change Plan / Change Quantity：由你方触发的变更（异步 + 需要 webhook/operation 协同）

> 这两个 API 返回 `202` 并给出 `Operation-Location`，你方需轮询 operations，并等待 webhook 通知“可以完成”，然后在你方系统真正变更，再对 operation ACK（见第 5 节）。

- **PATCH** `/api/saas/subscriptions/{subscriptionId}?api-version=2018-08-31`

#### Change Plan Request

```json
{ "planId": "gold" }
```

#### Change Quantity Request

```json
{ "quantity": 5 }

#### 完整示例（ChangePlan）

请求：

```bash
curl -sS -X PATCH \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -d '{"planId":"gold"}' \
  -i
```

响应：

- `202 Accepted`（无 body）
- Headers 包含 `Operation-Location: https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations/${OPERATION_ID}?api-version=2018-08-31`

#### 完整示例（ChangeQuantity）

请求：

```bash
curl -sS -X PATCH \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -d '{"quantity":5}' \
  -i
```

响应：

- `202 Accepted`（无 body）
- Headers 包含 `Operation-Location: .../operations/${OPERATION_ID}?api-version=2018-08-31`
```

#### Response Headers（202）

| Header | 说明 |
|---|---|
| `Operation-Location` | 形如 `/api/saas/subscriptions/{subscriptionId}/operations/{operationId}?api-version=2018-08-31` |

官方参考：
- Change Plan：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#change-the-plan-on-the-subscription
- Change Quantity：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#change-the-quantity-of-licenses-on-the-saas-subscription

### 3.5 Cancel（Unsubscribe）：由你方触发取消（不推荐，但可实现）

- **DELETE** `/api/saas/subscriptions/{subscriptionId}?api-version=2018-08-31`

#### 完整示例（Cancel / Unsubscribe）

请求：

```bash
curl -sS -X DELETE \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -i
```

响应：

- `202 Accepted`（常见，异步；Header 里有 `Operation-Location`）
- `200 OK`（订阅已处于 Unsubscribed 时）

> 多数场景建议引导客户回 Marketplace 取消。若你方实现取消，需要同样走异步 operation，最终再在你方侧撤销服务。

官方参考：
- Cancel：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#cancel-a-subscription

---

## 4. Webhook（订阅变更通知入口）

你方需要实现一个 **Connection webhook URL**（Partner Center 技术配置里填写），Microsoft 会对该 URL 发起 POST 通知。

### 4.1 Webhook 事件类型与处理策略

| event (`action`) | 语义 | 你方需要做什么 |
|---|---|---|
| `ChangePlan` / `ChangeQuantity` | 客户发起变更，需要你方“接收/拒绝” | **先 200 ACK**，再校验并最终对 operation PATCH 成功/失败（第 5 节） |
| `Renew` / `Suspend` / `Unsubscribe` | 仅通知 | 200 ACK，然后你方同步状态（一般建议再 Get Subscription 对账） |
| `Reinstate` | 订阅恢复 | 200 ACK，必要时同步恢复；若无法接受，可按文档建议处理（例如触发删除） |

重要注意：
- Webhook 必须 7x24 可用；Microsoft 有重试（文档提到 8 小时内 500 次）
- 不要对 webhook payload 做“严格反序列化”（Microsoft 可能扩展字段）

官方参考：
- Webhook 总览：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook

### 4.2 Webhook 请求体（Schema 摘要）

不同事件 payload 字段基本一致，核心如下（示例见官方文档）。

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string(guid) | **operationId**（后续用于 operations API） |
| `subscriptionId` | string(guid) | subscriptionId |
| `action` | string | `ChangePlan` / `ChangeQuantity` / `Renew` / `Suspend` / `Unsubscribe` / `Reinstate` |
| `planId` | string | 目标 plan（ChangePlan 时） |
| `quantity` | number | 目标数量（ChangeQuantity 时） |
| `status` | string | 通常 `InProgress` 或 `Succeeded`（notify-only 事件） |
| `timeStamp` | string(datetime) | 时间戳（UTC） |
| `subscription` | object | 订阅快照（包含 offerId/planId/quantity/beneficiary/purchaser/status/term 等） |

#### 完整示例（ChangePlan payload，来自官方文档）

Webhook 请求 headers（你方会收到）：

```http
POST /webhook/marketplace HTTP/1.1
Content-Type: application/json
Authorization: Bearer <jwt>
```

Webhook 请求 body：

```json
{
  "id": "<guid>",
  "activityId": "<guid>",
  "publisherId": "XXX",
  "offerId": "YYY",
  "planId": "plan2",
  "quantity": 10,
  "subscriptionId": "<guid>",
  "timeStamp": "2023-02-10T18:48:58.4449937Z",
  "action": "ChangePlan",
  "status": "InProgress",
  "operationRequestSource": "Azure",
  "subscription": {
    "id": "<guid>",
    "name": "Test",
    "publisherId": "XXX",
    "offerId": "YYY",
    "planId": "plan1",
    "quantity": 10,
    "beneficiary": {
      "emailId": "XX@outlook.com",
      "objectId": "<guid>",
      "tenantId": "<guid>",
      "puid": "1234567890"
    },
    "purchaser": {
      "emailId": "XX@outlook.com",
      "objectId": "<guid>",
      "tenantId": "<guid>",
      "puid": "1234567890"
    },
    "allowedCustomerOperations": ["Delete", "Update", "Read"],
    "sessionMode": "None",
    "isFreeTrial": false,
    "isTest": false,
    "sandboxType": "None",
    "saasSubscriptionStatus": "Subscribed",
    "term": {
      "startDate": "2022-02-10T00:00:00Z",
      "endDate": "2022-03-12T00:00:00Z",
      "termUnit": "P1M",
      "chargeDuration": null
    },
    "autoRenew": true,
    "created": "2022-01-10T23:15:03.365988Z",
    "lastModified": "2022-02-14T20:26:04.5632549Z"
  },
  "purchaseToken": null
}
```

你方响应（用于 ACK webhook 已收到）：

```http
HTTP/1.1 200 OK
```

### 4.3 Webhook 安全（必须验证 JWT）

Microsoft 会在 webhook 请求里带 `Authorization: Bearer <jwt>`。你方必须验证该 JWT。

文档给出的关键校验点（claims）：
- `aud`：应等于你在 Partner Center 技术配置里填写的 **Entra Application ID**（用于 webhook 的受众）
- `appid` 或 `azp`：应等于你用于获取 publisher token 的 **resource ID**（文档说明可能在 `appid` 或 `azp` 出现）
- `tid`：应等于你在 Partner Center 技术配置里填写的 **Entra tenant ID**

官方参考（Marketplace webhook）：
- Webhook 安全章节：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook#securing-your-webhooks

---

## 5. Operations APIs（对 ChangePlan/ChangeQuantity 等“需要 ACK 的操作”做状态管理）

> 适用：`ChangePlan`、`ChangeQuantity`、`Reinstate`（文档说明：这些会产生可 ACK 的 operation）。

### 5.1 List Outstanding Operations：列出待处理操作

- **GET** `/api/saas/subscriptions/{subscriptionId}/operations?api-version=2018-08-31`

#### 完整示例（List Outstanding Operations）

请求：

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

#### Response（200）

```json
{ "operations": [ { "id": "<guid>", "action": "Reinstate", "status": "InProgress", "planId": "silver", "quantity": 20, "timeStamp": "..." } ] }
```

官方参考：
- List operations：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api#list-outstanding-operations

### 5.2 Get Operation Status：轮询某个 operation 的状态

- **GET** `/api/saas/subscriptions/{subscriptionId}/operations/{operationId}?api-version=2018-08-31`

#### 完整示例（Get Operation Status）

请求：

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations/${OPERATION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

响应（200 示例，来自官方文档）：

```json
{
  "id": "<guid>",
  "activityId": "<guid>",
  "subscriptionId": "<guid>",
  "offerId": "offer1",
  "publisherId": "contoso",
  "planId": "silver",
  "quantity": 20,
  "action": "ChangePlan",
  "timeStamp": "2018-12-01T00:00:00",
  "status": "InProgress",
  "errorStatusCode": "",
  "errorMessage": ""
}
```

#### Response（200）核心字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string(guid) | operationId |
| `action` | string | `ChangePlan` / `ChangeQuantity` / `Reinstate` |
| `status` | string | `NotStarted` / `InProgress` / `Failed` / `Succeeded` / `Conflict` |
| `planId` / `quantity` | string/number | 目标计划/数量 |
| `errorStatusCode` / `errorMessage` | string | 失败时的错误信息 |

官方参考：
- Get operation：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api#get-operation-status

### 5.3 Patch Operation：你方完成变更后，回写 Success/Failure

- **PATCH** `/api/saas/subscriptions/{subscriptionId}/operations/{operationId}?api-version=2018-08-31`

#### 完整示例（Patch Operation）

请求（完成变更后回写 Success）：

```bash
curl -sS -X PATCH \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations/${OPERATION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -d '{"status":"Success"}' \
  -i
```

响应：

- `200 OK`（通常无 body）

#### Request Body

```json
{ "status": "Success" }
```

> `status` 仅允许：`Success` / `Failure`

官方参考：
- Patch operation：https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api#update-the-status-of-an-operation

---

## 6. Marketplace Metering（按量计费）

> 适用：你的 SaaS plan 定义了 custom metering dimensions（维度）。你方需要汇总用量并上报。

### 6.1 单条 usageEvent

- **POST** `/api/usageEvent?api-version=2018-08-31`

#### 完整示例（usageEvent）

请求：

```bash
curl -sS -X POST \
  "https://marketplaceapi.microsoft.com/api/usageEvent?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -d '{
    "resourceId":"'"${SUBSCRIPTION_ID}"'",
    "quantity":5.0,
    "dimension":"dim1",
    "effectiveStartTime":"2018-12-01T08:30:14Z",
    "planId":"plan1"
  }'
```

响应（200 Accepted 示例，来自官方文档）：

```json
{
  "usageEventId": "<guid>",
  "status": "Accepted",
  "messageTime": "2020-01-12T13:19:35.3458658Z",
  "resourceId": "<guid>",
  "quantity": 5.0,
  "dimension": "dim1",
  "effectiveStartTime": "2018-12-01T08:30:14",
  "planId": "plan1"
}
```

响应（409 Duplicate 示例，来自官方文档）：

```json
{
  "additionalInfo": {
    "acceptedMessage": {
      "usageEventId": "<guid>",
      "status": "Duplicate",
      "messageTime": "2020-01-12T13:19:35.3458658Z",
      "resourceId": "<guid>",
      "quantity": 1.0,
      "dimension": "dim1",
      "effectiveStartTime": "2020-01-12T11:03:28.14Z",
      "planId": "plan1"
    }
  },
  "message": "This usage event already exist.",
  "code": "Conflict"
}
```

响应（400 BadArgument 示例，来自官方文档）：

```json
{
  "message": "One or more errors have occurred.",
  "target": "usageEventRequest",
  "details": [
    {
      "message": "The resourceId is required.",
      "target": "ResourceId",
      "code": "BadArgument"
    }
  ],
  "code": "BadArgument"
}
```

#### Request Body（SaaS）

| 字段 | 必填 | 类型 | 说明 |
|---|---:|---|---|
| `resourceId` | 是 | guid | **SaaS 场景：subscriptionId** |
| `quantity` | 是 | number | > 0，建议按小时汇总 |
| `dimension` | 是 | string | 自定义计量维度 id |
| `effectiveStartTime` | 是 | string(datetime) | UTC，且 **只能上报过去 24 小时内** |
| `planId` | 是 | string | planId |

#### Response（200 Accepted）

返回会包含 `usageEventId`、`status: Accepted`、回显 `resourceId/quantity/dimension/effectiveStartTime/planId` 等。

#### 关键限制/错误

- 同一 `resourceId + dimension + 小时` 只能成功一次；重复会 `409 Conflict`（Duplicate）
- 过期（超过 24h）会 `400`

官方参考：
- usageEvent：https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis#metered-billing-single-usage-event

### 6.2 批量 batchUsageEvent（最多 25 条）

- **POST** `/api/batchUsageEvent?api-version=2018-08-31`

#### 完整示例（batchUsageEvent）

请求：

```bash
curl -sS -X POST \
  "https://marketplaceapi.microsoft.com/api/batchUsageEvent?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -d '{
    "request": [
      {
        "resourceId": "'"${SUBSCRIPTION_ID}"'",
        "quantity": 5.0,
        "dimension": "dim1",
        "effectiveStartTime": "2018-12-01T08:30:14Z",
        "planId": "plan1"
      },
      {
        "resourceId": "'"${SUBSCRIPTION_ID_2}"'",
        "quantity": 1.0,
        "dimension": "email",
        "effectiveStartTime": "2018-12-01T09:10:00Z",
        "planId": "gold"
      }
    ]
  }'
```

#### Request Body（SaaS）

```json
{
  "request": [
    { "resourceId": "<guid>", "quantity": 5.0, "dimension": "dim1", "effectiveStartTime": "2018-12-01T08:30:14Z", "planId": "plan1" }
  ]
}
```

#### Response（200）

返回 `result[]`，每条可能是 `Accepted` / `Duplicate` / `Expired` / `Error` 等（文档有完整枚举）。你方需要逐条处理重试/忽略。

响应（200 示例，来自官方文档）：

```json
{
  "count": 2,
  "result": [
    {
      "usageEventId": "<guid>",
      "status": "Accepted",
      "messageTime": "2020-01-12T13:19:35.3458658Z",
      "resourceId": "<guid1>",
      "quantity": 5.0,
      "dimension": "dim1",
      "effectiveStartTime": "2018-12-01T08:30:14",
      "planId": "plan1"
    },
    {
      "status": "Duplicate",
      "messageTime": "0001-01-01T00:00:00",
      "error": {
        "additionalInfo": {
          "acceptedMessage": {
            "usageEventId": "<guid>",
            "status": "Duplicate",
            "messageTime": "2020-01-12T13:19:35.3458658Z",
            "resourceId": "<guid2>",
            "quantity": 1.0,
            "dimension": "email",
            "effectiveStartTime": "2020-01-12T11:03:28.14Z",
            "planId": "gold"
          }
        },
        "message": "This usage event already exist.",
        "code": "Conflict"
      },
      "resourceId": "<guid2>",
      "quantity": 1.0,
      "dimension": "email",
      "effectiveStartTime": "2020-01-12T11:03:28.14Z",
      "planId": "gold"
    }
  ]
}
```

官方参考：
- batchUsageEvent：https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis#metered-billing-batch-usage-event

### 6.3 查询 usageEvents（对账/审计）

- **GET** `/api/usageEvents?api-version=2018-08-31&usageStartDate=<iso8601>[&usageEndDate=...&offerId=...&planId=...&dimension=...&reconStatus=...]`

#### 完整示例（usageEvents 查询）

请求：

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/usageEvents?api-version=2018-08-31&usageStartDate=2020-12-03" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

响应（示例，来自官方文档）：

```json
[
  {
    "usageDate": "2020-11-30T00:00:00Z",
    "usageResourceId": "11111111-2222-3333-4444-555555555555",
    "dimension": "tokens",
    "planId": "silver",
    "planName": "Silver",
    "offerId": "mycooloffer",
    "offerName": "My Cool Offer",
    "offerType": "SaaS",
    "azureSubscriptionId": "12345678-9012-3456-7890-123456789012",
    "reconStatus": "Accepted",
    "submittedQuantity": 17.0,
    "processedQuantity": 17.0,
    "submittedCount": 17
  }
]
```

返回一个数组，包含 `usageDate`、`usageResourceId`、`dimension`、`reconStatus`、`submittedQuantity`、`processedQuantity` 等。

官方参考：
- usageEvents 查询：https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis#metered-billing-retrieve-usage-events

---

## 7. 把接口落到你方实现的“最小闭环”清单

- Landing Page：拿到 `token` → Resolve → 以 `subscriptionId` 做幂等开通 → Activate
- 订阅对账：定期 Get Subscription / List Subscriptions（可选）
- Webhook：7x24 接收 + **JWT 验证** + 快速 200 ACK
- 变更处理（ChangePlan/ChangeQuantity）：Get Operation 校验 → 你方实际变更 → Patch Operation Success/Failure
- Metering（如有维度）：按小时聚合 → usageEvent/batchUsageEvent → 409 Duplicate/400 Expired 的处理策略 + 审计


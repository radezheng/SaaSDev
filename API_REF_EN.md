# Azure Marketplace SaaS (Transactable) API Reference (organized by sequence)

> Goal: list the Marketplace APIs your SaaS must implement/call, structured along the sequences of “first purchase / subscription updates / webhook / metered billing”, so engineers can integrate directly.
>
> Scope: Commercial Marketplace **SaaS Offer (Transactable)**, integrating **SaaS Fulfillment APIs v2** + (optional) **Marketplace Metering APIs**.

- 中文版 / Chinese: [API_REF.md](./API_REF.md)
- Implementation guide: [README_EN.md](./README_EN.md)

## 0. Common conventions

- **Base URL**: `https://marketplaceapi.microsoft.com`
- **api-version**: `2018-08-31`
- **Common headers** (recommended on every request)
  - `Content-Type: application/json`
  - `x-ms-requestid: <guid>` (client-generated)
  - `x-ms-correlationid: <guid>` (keep consistent within the same business flow for tracing)
- **Auth header** (except for the marketplace purchase token used by Resolve)
  - `Authorization: Bearer <access_token>`

### 0.1 Two kinds of “token” (do not confuse them)

- **Marketplace purchase identification token**: from the redirect to your Landing Page as `?token=...`. You can only use it for **Resolve** (pass via `x-ms-marketplace-token`).
- **Microsoft Entra access token (publisher authorization token)**: obtained by your backend via OAuth2 client credentials. Used to call **Fulfillment APIs / Operations APIs / Metering APIs** (pass via `Authorization: Bearer ...`).

Official references:
- SaaS Fulfillment Subscription APIs v2: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api
- SaaS Fulfillment Operations APIs v2: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api
- SaaS Webhook: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook
- Marketplace Metering APIs: https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis

---

## 1. How your backend obtains `access_token` (publisher authorization token)

Your backend calls Fulfillment/Metering APIs using a **Microsoft Entra ID OAuth2 client credentials** access token.

### 1.1 Token request

- **POST** `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`
- **Content-Type**: `application/x-www-form-urlencoded`

#### Request Body (form)

| Field | Required | Type | Description |
|---|---:|---|---|
| `grant_type` | Yes | string | Fixed: `client_credentials` |
| `client_id` | Yes | string | Application (client) ID of your Entra app registration |
| `client_secret` | Yes | string | Client secret |
| `scope` | Yes | string | Fixed: `20e940b3-4c77-4b0b-9a53-9e16a1b010a7/.default` |

#### Response (200)

| Field | Type | Description |
|---|---|---|
| `token_type` | string | `Bearer` |
| `expires_in` | string/int | typically 3600 seconds |
| `access_token` | string | put into `Authorization` header |

#### Full example (get Entra access_token)

Request:

```bash
curl -sS -X POST \
  "https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=${CLIENT_ID}" \
  --data-urlencode "client_secret=${CLIENT_SECRET}" \
  --data-urlencode "scope=20e940b3-4c77-4b0b-9a53-9e16a1b010a7/.default"
```

Response (example):

```json
{
  "token_type": "Bearer",
  "expires_in": "3600",
  "ext_expires_in": "0",
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIs..."
}
```

Official reference:
- Register SaaS and obtain token: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-registration

---

## 2. First purchase (Landing Page) — Resolve → (your Provision) → Activate

### 2.1 Resolve: exchange Marketplace `token` for durable `subscriptionId`

After purchase, Marketplace redirects the customer to your Landing Page with `token=...` (purchase identification token). You must call Resolve to obtain `subscriptionId` and other subscription details.

- **POST** `/api/saas/subscriptions/resolve?api-version=2018-08-31`

#### Request headers

| Header | Required | Description |
|---|---:|---|
| `Authorization` | Yes | your backend access token |
| `x-ms-marketplace-token` | Yes | the `token` in Landing URL (URL-decode before sending) |

#### Full example (Resolve)

Request:

```bash
curl -sS -X POST \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/resolve?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "x-ms-marketplace-token: ${MARKETPLACE_TOKEN}"
```

Response (200 example from official docs):

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

#### Key fields (200)

Resolve returns both top-level fields and a richer `subscription` object. At minimum you typically need: `id` (subscriptionId), `offerId`, `planId`, `quantity`, and `saasSubscriptionStatus`.

| Path | Type | Description |
|---|---|---|
| `id` | string(guid) | **subscriptionId** |
| `offerId` | string | offer identifier |
| `planId` | string | plan identifier |
| `quantity` | number | seat count (may be empty if not per-seat) |
| `subscription.saasSubscriptionStatus` | string | `PendingFulfillmentStart` / `Subscribed` / `Suspended` / `Unsubscribed` |
| `subscription.allowedCustomerOperations` | string[] | `Read`/`Update`/`Delete` (in CSP scenarios may be Read-only) |

#### Common status codes

- `200`: OK
- `400`: token missing/invalid/expired (token is valid for 24h)
- `401/403`: access token invalid or registration mismatch

Official reference:
- Resolve: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#resolve-a-purchased-subscription

### 2.2 Provision (your business logic, not a Marketplace API)

Based on the `subscriptionId` from Resolve, you need to perform internal provisioning/binding/reconciliation.

- Recommended minimum fields to persist: `subscriptionId`, `offerId`, `planId`, `quantity`, `beneficiary.tenantId`, purchaser/beneficiary, current status, last operationId (if any)
- Idempotency: repeated resolve/webhook must not create duplicates

### 2.3 Activate: notify Marketplace “setup complete, start billing”

- **POST** `/api/saas/subscriptions/{subscriptionId}/activate?api-version=2018-08-31`

#### Request headers

| Header | Required | Description |
|---|---:|---|
| `Authorization` | Yes | your backend access token |

#### Full example (Activate)

Request:

```bash
curl -sS -X POST \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/activate?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -i
```

Response:

- `200 OK` (no response body)

#### Notes

- `200`: request accepted (no body). You can later observe via Get Subscription/Operations.
- Common failures: `400` (Suspended), `404` (Unsubscribed)

Official reference:
- Activate: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#activate-a-subscription

---

## 3. Subscription queries & updates (Subscription APIs v2)

### 3.1 Get Subscription

- **GET** `/api/saas/subscriptions/{subscriptionId}?api-version=2018-08-31`

#### Full example (Get Subscription)

Request:

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Response (200 example from official docs):

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

#### Key fields (200)

| Field | Type | Description |
|---|---|---|
| `id` | string(guid) | subscriptionId |
| `offerId` / `planId` | string | current offer/plan |
| `quantity` | number | seat count (may be empty) |
| `saasSubscriptionStatus` | string | subscription status |
| `term.termUnit` | string | `P1M` / `P1Y` etc. |
| `term.startDate` / `term.endDate` | string(datetime) | available after Active |

Official reference:
- Get Subscription: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#get-subscription

### 3.2 List Subscriptions (paged)

- **GET** `/api/saas/subscriptions?api-version=2018-08-31[&continuationToken=...]`

#### Full example (List Subscriptions)

Request (first page, without continuationToken):

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Response (200 example from official docs):

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

#### Fields

| Field | Type | Description |
|---|---|---|
| `subscriptions` | array | subscription list (id/name/offerId/planId/quantity/beneficiary/purchaser/term/status etc.) |
| `@nextLink` | string | next-page link with continuationToken |

Official reference:
- List Subscriptions: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#get-list-of-all-subscriptions

### 3.3 List Available Plans

- **GET** `/api/saas/subscriptions/{subscriptionId}/listAvailablePlans?api-version=2018-08-31[&planId=...]`

#### Full example (List Available Plans)

Request:

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/listAvailablePlans?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Response snippet (example from official docs):

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

> Commonly used to show “switchable plans” in your UI. The response can include public/private plans and (optional) private offer association.

Official reference:
- List Available Plans: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#list-available-plans

### 3.4 Change Plan / Change Quantity (publisher-initiated, async)

> These APIs return `202` with `Operation-Location`. You must poll operations, wait for webhook indicating you can complete, then apply the change in your system and ACK the operation (see Section 5).

- **PATCH** `/api/saas/subscriptions/{subscriptionId}?api-version=2018-08-31`

#### Change Plan request

```json
{ "planId": "gold" }
```

#### Change Quantity request

```json
{ "quantity": 5 }
```

#### Full example (ChangePlan)

Request:

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

Response:

- `202 Accepted` (no body)
- Headers include: `Operation-Location: https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations/${OPERATION_ID}?api-version=2018-08-31`

#### Full example (ChangeQuantity)

Request:

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

Response:

- `202 Accepted` (no body)
- Headers include: `Operation-Location: .../operations/${OPERATION_ID}?api-version=2018-08-31`

#### Response headers (202)

| Header | Description |
|---|---|
| `Operation-Location` | `/api/saas/subscriptions/{subscriptionId}/operations/{operationId}?api-version=2018-08-31` |

Official references:
- Change Plan: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#change-the-plan-on-the-subscription
- Change Quantity: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#change-the-quantity-of-licenses-on-the-saas-subscription

### 3.5 Cancel (Unsubscribe) initiated by publisher (not recommended, but possible)

- **DELETE** `/api/saas/subscriptions/{subscriptionId}?api-version=2018-08-31`

#### Full example (Cancel / Unsubscribe)

Request:

```bash
curl -sS -X DELETE \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -i
```

Response:

- `202 Accepted` (common; async; `Operation-Location` in headers)
- `200 OK` (if already Unsubscribed)

> In most scenarios, you should guide customers to cancel in Marketplace. If you implement cancellation, it follows the async operation pattern and you still must deprovision service on your side.

Official reference:
- Cancel: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#cancel-a-subscription

---

## 4. Webhook (subscription change notification endpoint)

You must implement a **Connection webhook URL** (provided in Partner Center technical configuration). Microsoft will send POST notifications to this URL.

### 4.1 Webhook event types & handling strategy

| event (`action`) | Meaning | What you should do |
|---|---|---|
| `ChangePlan` / `ChangeQuantity` | customer initiated change requiring accept/reject | **ACK 200 quickly**, then validate and finally PATCH operation success/failure (Section 5) |
| `Renew` / `Suspend` / `Unsubscribe` | notify-only | ACK 200, then sync state (recommended to call Get Subscription for reconciliation) |
| `Reinstate` | subscription reinstated | ACK 200 and restore if needed; if you cannot accept, handle per docs (e.g., trigger deletion) |

Important notes:

- Webhook must be available 24/7; Microsoft retries (docs mention up to 500 retries within 8 hours)
- Do not strictly deserialize payload (Microsoft may add fields)

Official reference:
- Webhook overview: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook

### 4.2 Webhook payload (schema summary)

Payload fields are mostly consistent across events. Core fields:

| Field | Type | Description |
|---|---|---|
| `id` | string(guid) | **operationId** (used by operations API) |
| `subscriptionId` | string(guid) | subscriptionId |
| `action` | string | `ChangePlan` / `ChangeQuantity` / `Renew` / `Suspend` / `Unsubscribe` / `Reinstate` |
| `planId` | string | target plan (for ChangePlan) |
| `quantity` | number | target quantity (for ChangeQuantity) |
| `status` | string | usually `InProgress` or `Succeeded` (notify-only events) |
| `timeStamp` | string(datetime) | timestamp (UTC) |
| `subscription` | object | subscription snapshot (offerId/planId/quantity/beneficiary/purchaser/status/term etc.) |

#### Full example (ChangePlan payload, from official docs)

Webhook request headers (what you receive):

```http
POST /webhook/marketplace HTTP/1.1
Content-Type: application/json
Authorization: Bearer <jwt>
```

Webhook body:

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

Your response (ACK):

```http
HTTP/1.1 200 OK
```

### 4.3 Webhook security (JWT validation is required)

Microsoft includes `Authorization: Bearer <jwt>` in webhook requests. You must validate this JWT.

Key validations from docs (claims):

- `aud`: must equal the **Entra Application ID** you configured in Partner Center for the webhook audience
- `appid` or `azp`: should equal the **resource ID** used when obtaining publisher token (may appear in `appid` or `azp`)
- `tid`: must equal the **Entra tenant ID** in Partner Center technical configuration

Official reference:
- Securing webhooks: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-webhook#securing-your-webhooks

---

## 5. Operations APIs (manage operations that require ACK)

> Applies to `ChangePlan`, `ChangeQuantity`, and `Reinstate` (docs indicate these produce ACKable operations).

### 5.1 List Outstanding Operations

- **GET** `/api/saas/subscriptions/{subscriptionId}/operations?api-version=2018-08-31`

#### Full example

Request:

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Response (200):

```json
{ "operations": [ { "id": "<guid>", "action": "Reinstate", "status": "InProgress", "planId": "silver", "quantity": 20, "timeStamp": "..." } ] }
```

Official reference:
- List operations: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api#list-outstanding-operations

### 5.2 Get Operation Status (poll)

- **GET** `/api/saas/subscriptions/{subscriptionId}/operations/{operationId}?api-version=2018-08-31`

#### Full example

Request:

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/saas/subscriptions/${SUBSCRIPTION_ID}/operations/${OPERATION_ID}?api-version=2018-08-31" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Response (200 example from official docs):

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

#### Key fields

| Field | Type | Description |
|---|---|---|
| `id` | string(guid) | operationId |
| `action` | string | `ChangePlan` / `ChangeQuantity` / `Reinstate` |
| `status` | string | `NotStarted` / `InProgress` / `Failed` / `Succeeded` / `Conflict` |
| `planId` / `quantity` | string/number | target plan/quantity |
| `errorStatusCode` / `errorMessage` | string | error details on failure |

Official reference:
- Get operation: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api#get-operation-status

### 5.3 Patch Operation (ACK success/failure)

- **PATCH** `/api/saas/subscriptions/{subscriptionId}/operations/{operationId}?api-version=2018-08-31`

#### Full example

Request (after applying change, ACK success):

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

Response:

- `200 OK` (typically no body)

#### Request body

```json
{ "status": "Success" }
```

> `status` allowed values: `Success` / `Failure`

Official reference:
- Patch operation: https://learn.microsoft.com/partner-center/marketplace-offers/pc-saas-fulfillment-operations-api#update-the-status-of-an-operation

---

## 6. Marketplace Metering (metered billing)

> Applies when your SaaS plan defines custom metering dimensions. You aggregate usage and report to Marketplace.

### 6.1 Single usageEvent

- **POST** `/api/usageEvent?api-version=2018-08-31`

#### Full example (usageEvent)

Request:

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

Response (200 Accepted example from official docs):

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

Response (409 Duplicate example from official docs):

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

Response (400 BadArgument example from official docs):

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

#### Request body (SaaS)

| Field | Required | Type | Description |
|---|---:|---|---|
| `resourceId` | Yes | guid | **For SaaS: subscriptionId** |
| `quantity` | Yes | number | > 0; recommended to aggregate hourly |
| `dimension` | Yes | string | custom metering dimension id |
| `effectiveStartTime` | Yes | string(datetime) | UTC; **must be within the last 24 hours** |
| `planId` | Yes | string | planId |

#### Key limits/errors

- For a given `resourceId + dimension + hour`, only one event can be accepted; duplicates return `409 Conflict` (Duplicate)
- Expired (older than 24h) returns `400`

Official reference:
- usageEvent: https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis#metered-billing-single-usage-event

### 6.2 Batch batchUsageEvent (max 25 items)

- **POST** `/api/batchUsageEvent?api-version=2018-08-31`

#### Full example (batchUsageEvent)

Request:

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

#### Request body (SaaS)

```json
{
  "request": [
    { "resourceId": "<guid>", "quantity": 5.0, "dimension": "dim1", "effectiveStartTime": "2018-12-01T08:30:14Z", "planId": "plan1" }
  ]
}
```

#### Response (200)

Returns `result[]`. Each item may be `Accepted` / `Duplicate` / `Expired` / `Error`, etc. You must handle per-item retries/ignores.

Response (200 example from official docs):

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

Official reference:
- batchUsageEvent: https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis#metered-billing-batch-usage-event

### 6.3 Retrieve usageEvents (reconciliation/audit)

- **GET** `/api/usageEvents?api-version=2018-08-31&usageStartDate=<iso8601>[&usageEndDate=...&offerId=...&planId=...&dimension=...&reconStatus=...]`

#### Full example

Request:

```bash
curl -sS -X GET \
  "https://marketplaceapi.microsoft.com/api/usageEvents?api-version=2018-08-31&usageStartDate=2020-12-03" \
  -H "Content-Type: application/json" \
  -H "x-ms-requestid: ${REQUEST_ID}" \
  -H "x-ms-correlationid: ${CORRELATION_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Response (example from official docs):

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

Returns an array containing `usageDate`, `usageResourceId`, `dimension`, `reconStatus`, `submittedQuantity`, `processedQuantity`, etc.

Official reference:
- Retrieve usage events: https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-metering-service-apis#metered-billing-retrieve-usage-events

---

## 7. Minimal end-to-end checklist for your implementation

- Landing Page: receive `token` → Resolve → idempotent provisioning by `subscriptionId` → Activate
- Subscription reconciliation: scheduled Get Subscription / List Subscriptions (optional)
- Webhook: 24/7 receiver + **JWT validation** + quick 200 ACK
- Change handling (ChangePlan/ChangeQuantity): validate operation via Get Operation → apply change in your system → Patch Operation Success/Failure
- Metering (if dimensions exist): hourly aggregation → usageEvent/batchUsageEvent → handle 409 Duplicate / 400 Expired + audit

# Azure Marketplace（Microsoft Commercial Marketplace）SaaS 上架实施工作说明

- English version: [README_EN.md](./README_EN.md)
- API reference (EN): [API_REF_EN.md](./API_REF_EN.md)

> 目标：把“要上架一个 SaaS 应用到 Azure Marketplace”拆解成可执行的工作项、接口清单、数据模型与验收标准，方便团队照着实现并评估工作量。
>
> 适用：不限定技术栈（.NET / Java / Node.js / Python / Go / 任意）。本文以仓库的 [**SaaS Accelerator Repo**](https://github.com/Azure/Commercial-Marketplace-SaaS-Accelerator) 作为参考实现来源，抽象出通用模式。

---

## 1. 你需要实现什么（用一句话说清楚）

当客户在 Azure Marketplace 购买你的 SaaS Offer 后，Marketplace 会把客户重定向到你的“落地页（Landing Page）”；你的服务需要：

- 识别并解析 Marketplace 回传的 token（`marketplace token`），调用 **SaaS Fulfillment API** 解析订阅信息
- 在你的 SaaS 平台里创建/绑定客户租户与订阅（Provision）
- 在合适的时机调用 **Activate** 开始计费（或进入 PendingActivation 走人工/异步审批）
- 处理后续订阅事件：Plan 变更、Quantity 变更、取消订阅等（通常通过 Webhook + 轮询兜底）
- （可选）实现按量计费：向 **Marketplace Metering** 上报使用量（usage events）

SaaS Accelerator 在应用形态上把这些能力拆成了：

- 客户侧门户/落地页（CustomerSite）：接 token、收集参数、触发激活等
- 发布方管理门户（AdminSite）：管理订阅、变更、手动上报 usage、查看日志、配置参数等
- 服务层（Services）与数据访问层（DataAccess）
- （可选）定时触发上报（MeteredTriggerJob / Scheduler Manager）

---

## 2. 概念与关键决策（不做清楚会反复返工）

### 2.1 Offer 形态与计费模型
- **SaaS Offer（Transactable）**：Marketplace 负责订阅与计费入口；你负责交付 SaaS 服务与生命周期管理。
- **计费**常见组合：
  - 纯订阅（按月/按年等）
  - 按用户（Quantity/Seat）
  - 订阅 + 按量（Metered Dimensions）

关键决策：
1) 是否需要 **按量计费（Metered）**？如果需要，你要实现“Usage 归集/去重/重试/审计”。
2) 是否需要 **人工/异步激活**？（即客户购买后先 PendingActivation，再由你方激活）
3) 是否支持 **订阅更新**（Plan/Quantity 变更）？如果支持，要考虑“可接受/拒绝”的业务规则与幂等。

### 2.2 最少必做能力（MVP）
- Landing Page 接入 + Resolve
- 订阅记录落库（或至少可查询/可追溯）
- Activate（或 PendingActivation + 后台激活）
- Unsubscribe / Deprovision
- 安全：密钥管理、RBAC、审计日志

[Python MVP 示例参考](https://github.com/mason1002/ms-mkp-py-mvp)

---

## 3. 参考架构（可替换任何语言与部署形态）

### 3.1 逻辑组件图（推荐）

![arch](./img/marketplace_publisher.png)

### 3.2 运行时边界与职责
- **Marketplace**：发起购买、托管订阅、发起跳转、提供 Fulfillment/Metering API
- **你的 SaaS 平台**：
  - 负责账号/租户与 Marketplace 订阅的映射
  - 负责开通/停用服务、资源配额、策略控制
  - 负责使用量计算并上报（如使用 Metered）
  - 负责审计、告警、合规（安全/隐私/可用性）

---

## 4. Partner Center 侧工作（产品/配置）

> 这部分通常由产品/商务/发布负责人 + 技术负责人共同完成。建议把“Partner Center 配置项”当成工单清单，避免上线前遗漏。

### 4.1 前置：账号与发布权限
- Partner Center 账户、发布者验证、税务/付款资料
- 选择 SaaS Offer 类型、定义 Offer ID / Plan ID（这些会进入你的代码与数据模型）

### 4.2 定义 Plans、计费周期、（可选）Metered Dimensions
- 订阅 Plan：定义名称、周期、价格、是否按用户
- Metered Dimensions（可选）：定义维度名（dimension）、单位、上限等

### 4.3 技术配置（Technical configuration）
你需要提供给 Partner Center 的通常包括：
- **Landing Page URL**（客户购买后重定向的 URL）
- **Webhook / Notification URL**（Marketplace 推送订阅变更事件的入口）
- **Entra ID / AAD 配置**（Tenant ID / App ID / Secret 等，取决于认证方式）

> SaaS Accelerator 的部署脚本会自动创建所需的 App Registration，并把相关值写入配置与数据库；你的实现可以用 IaC/脚本或手工配置，核心是“值一致且可轮换”。

---

## 5. 身份认证与密钥（实现层必须落地）

### 5.0 Marketplace 身份 vs SaaS 身份（务必区分）

- **Marketplace 侧（购买与计费）**：客户在 Azure/Microsoft Commercial Marketplace 完成购买时，使用其组织的 Microsoft Entra ID 身份登录；订阅与计费归属在客户的租户/计费体系之下。
- **你方调用平台 API 的身份（必需）**：你方服务端使用自己注册的 Entra 应用（App Registration），通过 client credentials 获取访问令牌，调用 Fulfillment API / Metering API。
- **SaaS 侧（你产品的用户登录）**：由发布方自行决定与实现（自建账号、Entra/第三方 SSO、B2C 等）。

实现原则：Landing Page URL 上出现的任何参数都不能当作订阅权威身份；订阅的权威标识应以 `marketplace token` 调用 Fulfillment Resolve 后获得的 `subscriptionId`（以及返回的 offer/plan 信息）为准，并将其与 SaaS 内部 tenant/account 做映射与审计。

### 5.1 典型需要的两类应用身份
1) **调用 Marketplace Fulfillment/Metering API 的服务身份**
- 用于你服务端以 client credentials 获取 token
- 权限最小化（仅所需 API）

2) **客户访问 Landing Page / Portal 的登录身份（可选但强烈建议）**
- 用于客户在“Manage Account / 落地页”进入后登录你的 SaaS
- 支持多租户（multi-tenant）与单租户取决于你的业务

### 5.2 密钥管理与轮换
- 所有 client secret / certificate 存在 Key Vault（或等价 KMS），不要放在代码库/明文配置
- 建议：
  - 使用证书替代长期 secret（若团队成熟）
  - 实现轮换流程：新旧并存窗口、回滚策略、监控到期

---

## 6. 必需实现的接口与流程（按生命周期拆解）

> 这一章是“开发工作量”的核心：你可以把每个小节拆成后端/前端/运维的任务卡。

### 6.1 购买后跳转：Landing Page（必须）
**输入**：Marketplace 会将用户浏览器重定向到你在 Partner Center 中配置的 Landing Page URL，并附带用于解析订阅的 `marketplace token`（参数名/承载方式以官方文档与实际跳转为准）。

**关于“自定义参数”（从哪来、有什么用）**

- **来源**：来自你在 Partner Center 填写的 Landing Page URL 本身（例如你预先写入的 query string、路径片段），以及你站点的路由规则。Marketplace 重定向时会在该 URL 基础上追加 `marketplace token`。
- **示例（更直观）**：
  - Partner Center 中配置的 Landing Page URL：
    - `https://saas.contoso.com/landing?env=prod&region=cn&channel=marketplace`
  - 用户完成购买后，Marketplace 实际重定向（示意）：
    - `https://saas.contoso.com/landing?env=prod&region=cn&channel=marketplace&token=<marketplaceToken>`
  - 说明：`env/region/channel` 为你自定义；`token` 为 Marketplace 追加（参数名/承载方式以实际为准）。
- **用途**：用于你的站点做环境/区域/渠道/产品线分流、灰度发布、A/B、日志打标等。例如：`env=prod`、`region=cn`、`channel=marketplace`、`offerAlias=xxx`。
- **边界**：这些自定义参数仅用于“站点行为/上下文”，不能替代订阅身份。订阅的权威标识应以 `marketplace token` 调用 Fulfillment Resolve 获取的 `subscriptionId`（以及返回的 offer/plan 信息）为准。

**实现注意事项**

- 把 URL 上的任何参数都当作不可信输入：仅接受白名单键名与取值范围，避免被用来切换到错误环境/租户。
- 不要在 URL 自定义参数中携带敏感信息（例如内部租户 ID、密钥、数据库标识等）。
- 建议把“自定义参数”落入日志的维度字段（用于排障/统计），但不要写入为订阅的唯一身份。

**你要做的事**：
1) 校验请求基本合法性（HTTPS、来源、参数存在）
2) 用 token 调 Fulfillment **Resolve** 获取：subscriptionId、offerId、planId、购买方信息等
3) 在你的系统中：
   - 创建/绑定客户租户（tenant/account）
   - 创建订阅记录（subscription）
   - 保存 Marketplace 订阅与内部订阅的映射
4) 引导客户：
   - 若需要客户填写额外信息（如管理员邮箱、区域、实例名），在此页面收集
   - 进入“自动开通”或“待激活”路径

建议落库数据（最少）：
- marketplaceSubscriptionId / subscriptionId（GUID）
- offerId、planId、quantity
- purchaser tenant / user 标识（若可获取）
- 状态机：PendingFulfillmentStart / PendingActivation / Active / Suspended / Unsubscribed …
- 时间戳、审计日志

### 6.2 Resolve（必须）
**目的**：把 marketplace token 解析为订阅实体。

实现要点：
- token 只在跳转初期使用，不要当作长期身份凭据
- Resolve 应该是幂等的：同一个 token 多次访问不会造成重复创建

### 6.3 Provision（必须）
**目的**：在你的 SaaS 平台实际开通服务。

你需要定义“开通”的语义：
- 创建租户空间（Tenant）、初始化配置、分配资源、创建默认管理员等
- 失败时要能：重试 / 回滚 / 人工介入

### 6.4 Activate（通常必须；或进入 PendingActivation）
**目的**：触发 Marketplace 开始计费。

两种模式：
- **自动激活**：Provision 成功后立即调用 Fulfillment Activate
- **人工/异步激活**：先标记 PendingActivation，管理员审核或异步任务完成后再 Activate

> SaaS Accelerator 支持通过配置开关决定是否自动激活。

### 6.5 Webhook：订阅更新与取消（强烈建议必须）
Marketplace 可能会在以下事件触发通知：
- Change Plan（升级/降级）
- Change Quantity（按用户 seat 变更）
- Suspend / Reinstate
- Unsubscribe

实现要点：
- **验证 webhook**：签名/令牌校验（按官方机制），避免被伪造请求驱动业务
- **幂等处理**：同一事件可能重放
- **异步化**：Webhook 入口快速 ACK，实际处理进队列/任务
- **兜底轮询**：对关键状态变化，定时调用 Fulfillment 查询状态，避免漏通知

### 6.6 Operations 轮询（用于异步变更）
Plan/Quantity 变更通常是异步操作：
- Fulfillment API 返回 operationId / operation-location
- 你要轮询 operation 状态直到成功/失败

### 6.7 Unsubscribe / Deprovision（必须）
**目标**：客户取消订阅时，正确停止计费与资源使用。

典型动作：
- 在 SaaS 侧禁用服务、回收配额、保留数据（按你的数据保留策略）
- 更新内部状态与审计

---

## 7. 按量计费（Metered）能力（可选，但工作量显著）

### 7.1 你需要实现的“使用量链路”
- 采集：从产品事件/日志/计量点采集原始 usage
- 归集：按 subscription + dimension + 时间窗口聚合
- 去重：同一业务事件不应重复计费
- 上报：调用 Marketplace Metering API
- 审计：记录请求、响应、重试次数、失败原因
- 补偿：失败重试、死信队列、人工重放

### 7.2 上报策略建议
- **小步上报**：按小时/天聚合，避免一次量太大
- **幂等键**：为每次 usage event 构造可追踪的唯一键（例如业务事件 ID）
- **失败处理**：
  - 可重试错误：退避重试
  - 不可重试错误：告警 + 进入人工队列

> SaaS Accelerator 提供了手动上报和 Scheduler Manager（定时固定量）作为参考。

---

## 8. 关键时序图（用于评审/估工期）
[API 参考](./API_REF.md)
### 8.1 购买后首次进入（Resolve + Provision + Activate）

![purchage_sequence](./img/marketplace_saas_purchase.png)
### 8.2 订阅变更（Plan/Quantity）

![change_sequence](./img/marketplace_subscription_update.png)  

### 8.3 按量计费上报（Usage Events）
![metered_sequence](./img/usage_metering_pipeline.png)
---

## 9. 数据模型建议（最少要能审计与排错）

> 你可以用 SQL / NoSQL；但强烈建议具备可查询与审计能力。

最少表/集合建议：
- `Subscriptions`
  - subscriptionId（Marketplace）
  - internalTenantId / accountId
  - offerId / planId / quantity
  - status（枚举 + 时间戳）
- `Plans`（从 Partner Center 同步或手工维护）
- `Operations`（plan/quantity 变更的 operationId、状态、失败原因）
- `UsageEvents`（raw + aggregated + reported state）
- `AuditLogs`（关键动作、调用请求/响应摘要、操作者）

---

## 10. 非功能性要求（上线门槛，别最后一周才补）

### 10.1 安全
- 所有入口必须 HTTPS；强制 HSTS
- Webhook 必须认证/验签
- 最小权限原则：调用 Fulfillment/Metering 的身份与 RBAC
- 租户隔离（Tenant boundary）：数据、权限、日志访问

### 10.2 可观测性
- 统一 correlationId / requestId，贯穿 Landing → 后端 → Marketplace API
- 监控：
  - Resolve/Activate/Webhook 成功率与延迟
  - Metering 上报失败率、重试积压
  - 订阅状态异常（长时间 Pending）

### 10.3 可用性与幂等
- Resolve/Provision/Activate/Webhook 处理必须可重试
- 关键写入采用 Upsert + 唯一索引防止重复订阅

---

## 11. 端到端实施步骤（建议按里程碑拆工期）

### Milestone A：准备与设计（1–2 周，视组织成熟度）
- A1. 明确 Offer/Plan/计费模型（是否 Metered、是否 seat）
- A2. 设计订阅状态机与幂等策略
- A3. 定义接口合同：Landing Page、Webhook、后台管理/API
- A4. 定义数据模型与审计字段

交付物：
- 架构图 + 时序图（本文可直接复用）
- 接口清单与字段定义（OpenAPI/Swagger）
- 数据模型（ERD 或 migration 脚本）

### Milestone B：基础链路（Resolve → Provision → Activate）（2–4 周）
- B1. Landing Page：接 token + 引导客户
- B2. Fulfillment Resolve：拿到 subscriptionId
- B3. Subscription 落库 + 幂等（重复访问/重放）
- B4. Provision 逻辑（创建租户、初始化）
- B5. Activate 或 PendingActivation

验收：
- 可以完成一次测试购买并进入 Active（或 PendingActivation 可人工激活）

### Milestone C：订阅变更与取消（2–3 周）
- C1. Webhook：认证、幂等、入队列
- C2. Plan/Quantity 变更：operation 轮询、接受/拒绝策略
- C3. Unsubscribe：停用/回收/数据保留策略
- C4. 轮询兜底任务（防漏通知）

验收：
- 在测试环境完成 plan 升降级、seat 变更、取消订阅

### Milestone D：Metered（可选，2–6 周，差异很大）
- D1. 维度设计与 Partner Center 配置
- D2. 产品侧 usage 采集点定义
- D3. 归集/去重/重试/审计
- D4. Scheduler（可选）

验收：
- 可稳定上报 usage（含失败重试、告警、审计）

### Milestone E：上线准备（1–3 周）
- E1. 安全审计（密钥、权限、租户隔离、Webhook 防护）
- E2. 监控告警与 Runbook
- E3. 压测/故障演练（重试风暴、Marketplace API 限流、DB 故障）
- E4. 生产发布与回滚方案

---

## 12. 工作量拆分参考（用于评估）

> 下面是“可估算”的粒度。实际人天取决于：是否已有 SaaS 平台、是否已有租户体系、是否需要 Metered、是否需要多区域/高可用。

| 模块 | 主要工作 | 典型复杂度 | 说明 |
|---|---|---:|---|
| Partner Center 配置 | Offer/Plan/技术配置/测试购买 | 中 | 通常非纯开发，但会反复来回核对 |
| 身份与密钥 | App 注册、权限、Key Vault、轮换 | 中 | 经常被忽略，最后会卡上线 |
| Landing + Resolve | 接 token、Resolve、收参、幂等 | 中 | MVP 关键路径 |
| Provision/Deprovision | 开通与回收资源/租户 | 高 | 与产品架构耦合最大 |
| Activate/状态机 | 自动/人工激活、状态迁移 | 中 | 需要强审计 |
| Webhook + 轮询 | 认证、队列化、幂等、补偿 | 高 | 生产稳定性关键 |
| Plan/Quantity 变更 | 操作轮询、接受/拒绝策略 | 中-高 | 业务规则差异大 |
| Metered | 采集/归集/去重/上报/重试 | 高 | 复杂度跨度最大 |
| 监控告警 | 指标、日志、告警、Runbook | 中 | 直接影响上线后运维成本 |

---

## 13. 验收清单（上线前必须全部通过）

### 13.1 功能验收
- [ ] 测试购买后可进入 Landing 并完成 Resolve
- [ ] Provision 幂等：重复进入不会重复创建租户/订阅
- [ ] 可成功 Activate（或进入 PendingActivation 并可人工激活）
- [ ] 可处理 Plan 变更（接受/拒绝都能正确闭环）
- [ ] 可处理 Quantity 变更（若启用 seat）
- [ ] 可处理取消订阅并完成 Deprovision
- [ ] （Metered）usage 上报成功率达标，失败可重试并可审计

### 13.2 非功能验收
- [ ] Webhook 鉴权/验签通过安全评审
- [ ] 密钥不落地在代码/配置文件，轮换有演练
- [ ] 关键链路有可观测性（日志、指标、告警）
- [ ] 具备故障处理 Runbook（Marketplace API 故障、限流、DB 故障）

---

## 14. 与SaaS Accelerator的对应关系（帮助读者快速对照参考实现）
[SaaS Accelerator Repo](https://github.com/Azure/Commercial-Marketplace-SaaS-Accelerator)
- Customer / Landing 参考：`src/CustomerSite`
- Publisher / Admin Portal 参考：`src/AdminSite`
- Fulfillment/Metering 调用封装与业务：`src/Services`
- 数据持久化（订阅/计划/审计）：`src/DataAccess`
- 定时/按量上报示例：`src/MeteredTriggerJob`
- 部署脚本参考：`deployment/Deploy.ps1`、`deployment/Upgrade.ps1`

> 你不需要照搬这些项目结构；但建议保留“入口层（Landing/Webhook）—业务层—持久化—作业/调度—监控”的分层。

---

## 15. 官方参考资料（建议在评审与实现时随时对照）

- SaaS offers 概览与创建流程：
  - https://learn.microsoft.com/azure/marketplace/partner-center-portal/create-new-saas-offer
- SaaS Fulfillment API（v2）：
  - https://learn.microsoft.com/azure/marketplace/partner-center-portal/pc-saas-fulfillment-api-v2
- Marketplace Metering Service API：
  - https://learn.microsoft.com/azure/marketplace/partner-center-portal/marketplace-metering-service-apis
- Metering FAQ（常见限制、时序与重试建议）：
  - https://learn.microsoft.com/azure/marketplace/partner-center-portal/marketplace-metering-service-apis-faq

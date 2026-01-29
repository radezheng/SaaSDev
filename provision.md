# Provision 阶段想在客户订阅内部署资源：可行性与官方推荐做法

本文回答一个常见诉求：

> Marketplace **SaaS Offer（Transactable）** 在你方处理 Provision 时，能否把一套 Azure 资源/服务部署到**客户自己的 Azure 订阅**里？如果可以，应该怎么做？

结论先说清楚：

- **仅使用 SaaS Fulfillment APIs（Resolve/Activate/Operations/Webhook）本身，不能直接“代表客户”在其订阅内创建资源。**Fulfillment 的 token/回调用于订阅生命周期管理，不是 Azure Resource Manager (ARM) 的授权。
- 如果你的产品形态是“需要把资源落在客户订阅内”（典型：在客户订阅创建 AKS/VM/Storage/Function 等），**官方更匹配的 Marketplace 交付形态是 Azure Application / Managed Application（托管应用）**，而不是纯 SaaS 交付。
- 你仍然可以把它做成“**SaaS + 客户侧部署**”的混合形态，但这需要你在 Provision 流程里额外完成 **客户授权** 或 **客户发起部署**（而不是 Marketplace 自动给你权限）。

下面分三种推荐路径说明。

---

## 1. 先澄清：SaaS Offer 的“Provision”在官方语义里是什么？

在 SaaS Fulfillment 的语境中：

- **Resolve**：把购买跳转带来的 `marketplace token` 解析成 `subscriptionId` 等订阅信息（见 [API_REF.md](API_REF.md)）。
- **Provision（你方业务）**：你方在自己的 SaaS 平台做“开通/绑定/初始化/对账/分配配额”等动作。
- **Activate**：你方确认已完成开通后，通知 Marketplace 开始计费。

注意：这套流程**没有**提供任何 ARM 调用所需的客户订阅授权（例如对客户订阅的 `Contributor` 权限，或 ARM token）。因此，

- “在客户订阅内部署资源”不属于 SaaS Fulfillment API 的直接能力范围。

---

## 2. 官方推荐：用 Azure Managed Application（Azure Application Offer）在客户订阅部署

如果你的交付确实需要在客户订阅落资源，Marketplace 官方对齐的做法通常是：

- 发布为 **Azure Application（托管应用 / Managed Application）**：客户购买/部署时由 Azure 平台在客户订阅中创建资源（ARM/Bicep 模板驱动）。
- 你方（发布方）可以通过托管应用机制获得受控的运维访问（通常是托管资源组/委派权限），从而后续升级/运维。

### 2.1 为什么这个形态更“官方/省心”

- **授权链路清晰**：客户在 Azure Portal/Marketplace 的部署体验中显式同意部署到其订阅。
- **可审计**：资源在客户订阅内可见、可控、可计费（按资源自身计费 + Marketplace 计费取决于定价模型）。
- **生命周期可绑定**：客户卸载/取消时，可以更自然地触发资源删除/停用策略。

### 2.2 你需要怎么做（高层步骤）

1) 在 Partner Center 选择/创建 **Azure Application** 类型的 offer/plan。
2) 准备交付模板：
   - ARM template / Bicep（建议）
   - 参数化：region、SKU、命名、网络、身份（Managed Identity/Service Principal）等
3) 选择交付方式：
   - Managed Application（托管应用）/ 其他支持的交付选项（以 Partner Center 当前选项为准）
4) 在你方后端把“客户订阅里的资源实例”与 Marketplace 的订阅（或订单）建立映射：
   - 记录 `subscriptionId`（Marketplace）
   - 记录客户 Azure 订阅、资源组、关键资源 ID（ARM resourceId）
5) 如果你仍然需要 SaaS 控制面（账号/权限/配置/计费维度），可以把它作为控制平面运行在你方订阅中。

> 实操建议：如果你们当前已经确定必须做“客户订阅内落资源”，优先评估是否需要把 Offer 形态从 SaaS 调整为 Azure Application；这会显著降低后续权限/合规/运维摩擦。

---

## 3. 可选混合方案：保留 SaaS Offer，但在 Provision 阶段引导客户“自助部署”

如果你必须保留 SaaS Offer（例如你更看重 SaaS 的订阅体验、按量计费、已有 SaaS 体系），也可以在 Provision 流程里做“客户侧部署”，但核心原则是：

- **部署动作要么由客户发起**，要么由客户显式授权你方身份后由你方发起。

### 3.1 模式 A：客户点击“一键部署”按钮（推荐，最容易合规）

Landing Page / 管理后台在完成 Resolve 后，引导客户点击部署：

- “Deploy to Azure” 按钮跳转 Azure Portal 自定义部署（Template Spec / ARM/Bicep）。
- 客户使用自身权限登录 Azure Portal，选择订阅/资源组，完成部署。

你方做什么：

- 提供模板与参数，生成部署链接。
- 监听部署结果（可通过客户回填、Webhook、或你方轮询部署状态——取决于你采用的集成方式）。
- 部署完成后，记录资源 ID，并继续你方业务开通，最后调用 Activate。

优点：

- 不需要你方拿客户订阅的高权限。
- 部署审计与权限完全在客户侧。

缺点：

- 体验上比“购买即完成”多一步。

### 3.2 模式 B：客户授权你方应用访问其订阅（更自动化，但权限/合规成本高）

流程概述：

1) 客户在你的 Landing/Portal 登录（建议 Entra 登录）。
2) 客户对你方的 Entra 应用完成同意（OAuth2 consent），并在其订阅/资源组给你的服务主体分配 RBAC（最小权限）。
3) 你方使用客户租户上下文获取 ARM token（或使用 On-behalf-of / delegated 权限等模式，具体取决于你的身份架构），然后用 ARM API 发起部署（ARM/Bicep）。

注意事项（必须评估）：

- 权限边界：最小化到资源组级别，而不是订阅级别。
- 安全与合规：你方将获得对客户资源的操作能力，需要明确条款、审计、撤销机制。
- 操作风险：误删/误配会直接影响客户环境。

很多团队最终会发现：如果要走到这种自动化程度，**Managed Application** 往往更合适。

---

## 4. 在 SaaS Provision 里“触发客户订阅部署”的落地建议（不论选哪种方案）

### 4.1 建议的状态机（避免乱序与重复计费）

- `PendingFulfillmentStart`（Resolve 后常见初态）
- `Provisioning`（你方开始开通/引导部署）
- `WaitingForCustomerDeployment`（如果需要客户点击部署）
- `Provisioned`（你方已确认资源就绪/账号初始化完成）
- `Activated`（你方向 Marketplace 调用 Activate 成功）
- `Failed`（失败可重试/人工介入）

原则：

- **只有在你方确认交付完成时才 Activate**，否则会出现“客户付费但服务不可用”的纠纷。
- 所有步骤必须**幂等**：同一个 `subscriptionId` 重放 Resolve/Webhook/用户刷新页面，都不应导致重复部署或重复创建。

### 4.2 你需要落库的关键映射

- Marketplace：`subscriptionId`、`offerId`、`planId`、`quantity`、状态、最近一次 operationId
- 客户 Azure：`tenantId`、`azureSubscriptionId`、`resourceGroup`、关键 `resourceId` 列表
- 部署追踪：部署名称、开始/结束时间、错误码、重试次数

### 4.3 取消订阅/卸载时如何处理

SaaS Offer 的取消（Unsubscribe）只代表 Marketplace 订阅生命周期结束。若你在客户订阅创建了资源，需要明确：

- 是否自动删除？（高风险，需谨慎）
- 是否仅停用/断开连接？
- 是否保留数据（合规/隐私）与保留期限

建议提供“资源清理”操作并留审计。

---

## 5. 快速决策表（你可以直接拿去和产品/架构评审）

- **必须在客户订阅落资源，且希望购买体验最丝滑**：优先选 **Azure Application / Managed Application**。
- **主要是 SaaS（你方托管），只是可选在客户订阅部署连接器/代理**：用 **SaaS + 客户自助部署（Deploy to Azure）**。
- **你方必须自动化在客户订阅部署且需要持续运维**：优先评估 **Managed Application**；如果坚持纯 SaaS，需要客户显式授权 + 强合规治理。

---

## 6. 与本仓库文档的关系

- Fulfillment API 的调用时序与字段见 [API_REF.md](API_REF.md)（Resolve/Activate/Operations/Webhook）。
- 本仓库现有“Provision”章节默认指的是“你方 SaaS 平台开通”，不包含客户订阅资源部署；如果你要做客户订阅部署，请以本文的 2/3 方案补齐。

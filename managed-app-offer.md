# Managed application offer（部署到客户订阅，含 ACI 示例）：发布者要做的事

> 目标：整理“把一套 Azure 资源部署到客户的 Azure 订阅”时，发布者在 **Partner Center + 技术实现** 侧需要完成的工作清单。
>
> 适用：你需要在客户订阅里落地资源（例如 ACI、VNet、Storage、Key Vault 等），并希望通过 Marketplace 提供可交易（transactable）的部署体验。

核心前提：

- Marketplace 的 **SaaS Fulfillment API** 只管订阅生命周期（Resolve/Activate/Webhook/Operations），**不提供**对客户订阅的 ARM 授权。
- “部署到客户订阅”的官方对齐路径通常是 **Azure Application offer 的 Managed application plan**。

官方参考：
- Plan Azure Application offer：https://learn.microsoft.com/partner-center/marketplace-offers/plan-azure-application-offer
- Plan managed application plan：https://learn.microsoft.com/partner-center/marketplace-offers/plan-azure-app-managed-app

---

## 1. 先定义你要交付什么（可验收项）

对齐本仓库的工作拆解方式：先把“客户订阅内那套东西”写成验收项，避免模板/参数反复返工。

- **ACI 工作负载**：镜像、启动命令、环境变量、端口、资源配额（CPU/Memory）
- **镜像分发**：镜像位置（ACR/Docker Hub/其他）、客户订阅内拉取方式（public 或 ACR 凭据/MI）
- **网络**：是否需要 VNet、私网、私有终结点、入站（Application Gateway/Front Door）
- **数据**：Storage/DB 依赖、数据保留与迁移
- **身份**：Managed Identity、访问 Key Vault/Storage 的最小权限
- **运维**：日志（Log Analytics）、告警、升级策略、回滚
- **卸载**：客户取消/卸载时资源如何处理（删除/保留/停用）

---

## 2. Partner Center 侧：发布者要做的事（清单）

### 2.1 创建 Offer
- Partner Center → Marketplace offers → New offer → **Azure Application**
- 定义 `Offer ID`、`Offer alias`（创建后 ID 不可改）

### 2.2 定义 Plan（至少一个）
- Plan 类型选择：**Managed application**（注意：Solution template 不是 Marketplace 交易型）
- 定义 Plan ID/Plan name、受众（public/private）、可用区域

### 2.3 定价与交易模型（需要产品/法务/架构一起定）

Managed application plan 支持 Marketplace 交易，但官方对定价口径有明确建议：

- “按月/计量（metered）的价格只能覆盖 management fee，不应作为 IP/软件费用/基础设施费用的计价方式”。

你需要在评审时先做决策：

- Marketplace 上你到底卖什么：托管服务费？软件订阅费？
  - 如果你要卖“软件订阅费”，很多团队会用 **SaaS offer** 承载订阅计费；Azure Application 负责“部署交付”。

### 2.4 Listing、线索与测试
- **Offer listing**：名称、简介、详细说明、截图/视频、logo、隐私与支持条款
- **Customer leads**：配置线索接收（CRM / HTTPS endpoint / Azure Table 等）
- **Preview audience**：强烈建议先做预览订阅测试再 Live

### 2.5 提交认证与发布
- 按 certification 要求修复反馈
- 任何编辑需要重新发布才会生效

---

## 3. 技术实现侧：部署包与模板要做的事（清单）

### 3.1 准备部署包（Deployment package）

官方要求 Azure Application 部署包为一个 `.zip`，且根目录至少包含：

- `mainTemplate.json`：ARM 模板（通常由 Bicep 编译生成）
- `createUiDefinition.json`：Azure Portal 部署 UI 定义

参考：
- https://learn.microsoft.com/partner-center/marketplace-offers/plan-azure-app-managed-app#deployment-package

建议实践：
- 用 Bicep 开发模板 → CI 生成 `mainTemplate.json`
- 用 ARM template test toolkit（ttk）做静态校验（Marketplace 推荐）

### 3.2 在模板里定义 ACI 与依赖资源

典型会涉及（按你实际架构取舍）：

- `Microsoft.ContainerInstance/containerGroups`（ACI）
- `Microsoft.Network/virtualNetworks` / `subnets`（如需私网）
- `Microsoft.OperationalInsights/workspaces`（日志）
- `Microsoft.KeyVault/vaults`（密钥）
- `Microsoft.Storage/storageAccounts`（持久化）

关键点：

- **镜像拉取**：
  - public image 最简单
  - private ACR 需要设计凭据/身份链路（避免把 ACR 密码硬编码在模板参数里）
- **参数化**：实例命名、region、SKU、网络开关、镜像 tag/版本、管理员邮箱等

### 3.3 设计“升级/回滚/卸载”策略
- 升级：镜像 tag 更新、配置变更（ACI 的滚动/蓝绿能力比 AKS 少，需要更明确的发布策略）
- 回滚：保留上一版本参数与配置，支持快速回退
- 卸载：客户删除 managed app 时资源如何处理（完全删除/保留数据要提前说清）

### 3.4 观测与支持
- 日志：容器 stdout/stderr、应用日志、部署日志
- 告警：容器重启、健康检查失败、依赖不可达
- 支持：提供诊断信息导出方式（便于一线排障）

---

## 4. 与 SaaS offer 的组合（常见产品形态）

- **纯 SaaS（你方托管）**：按 [API_REF.md](API_REF.md) + 本仓库 SaaS 实施说明做 Resolve/Provision/Activate。
- **客户订阅内交付（ACI/多资源）**：优先 Azure Application（Managed application plan）。
- **两者都要（客户可选 SaaS 或自部署）**：通常需要 **两个 offer**（SaaS + Azure Application），并在产品与销售侧把差异讲清楚。

---

## 5. 最小可执行 checklist（建议直接变成工单）

- 选 Offer/Plan：Azure Application → Managed application plan
- 明确交易模型：卖管理费？卖软件订阅费？是否需要 SaaS offer 组合
- 出部署架构草图：资源清单、网络拓扑、数据与身份边界
- 完成 Bicep/ARM：可一键部署到测试订阅
- 完成 `createUiDefinition.json`：参数校验与部署体验
- 打包部署包 `.zip`：并通过静态校验工具
- Partner Center：Offer + Plan + 定价 + Listing + Leads + Preview audience
- 端到端测试：预览订阅部署、升级、卸载、故障注入
- 提交认证：修复 certification 反馈
- 发布与运维：版本节奏、镜像漏洞与依赖治理、支持流程

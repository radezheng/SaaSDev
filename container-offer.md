# Container offer（Azure Container / Kubernetes Application）：发布者要做的事

> 目标：整理“发布 Azure Container offer（Kubernetes Application）”时，发布者在 **Partner Center + 技术打包（CNAB）** 侧要完成的工作。
>
> 适用：你的解决方案是 **Kubernetes 应用**，面向 **Managed AKS** 或 **Arc-enabled Kubernetes** 部署，并希望通过 Microsoft Marketplace 完成交易与部署体验。

官方参考（建议发布前先通读）：
- Plan Container offer：https://learn.microsoft.com/partner-center/marketplace-offers/marketplace-containers
- Create Azure Container offer：https://learn.microsoft.com/partner-center/marketplace-offers/azure-container-offer-setup
- Prepare technical assets：https://learn.microsoft.com/partner-center/marketplace-offers/azure-container-technical-assets-kubernetes

---

## 0. 关键限制（先确认能不能做）

来自官方技术资产文档的限制要点：

- 仅支持 **Linux AMD64** 镜像
- 目标集群：**Managed AKS** 或 **Arc-enabled Kubernetes**，且一个 offer 只能选一种集群类型
- **不支持 single containers**
- Docker container images 已退役（新的 Marketplace 容器 offer 以 **Kubernetes Applications** 形态发布）
- 技术包会复制到 Microsoft-owned registry，并对镜像做漏洞扫描

---

## 1. Partner Center 侧：发布者要做的事（清单）

### 1.1 创建 Offer
- Partner Center → Marketplace offers → + New offer → **Azure Container**
- 填写：
  - **Offer ID**（创建后不可修改）
  - **Offer alias**（仅 Partner Center 内部使用）

### 1.2 配置 Customer leads（可选但强烈建议）
- 连接 CRM / HTTPS endpoint / Azure Table 等，用于接收 Marketplace 线索

### 1.3 填写 Listing（素材与合规）
按官方 listing 要求准备：

- Name、Search results summary、Short description、Description
- Privacy policy link
- Contact information（Support/Engineering 等）
- Media（logo、截图、视频）

并确保满足 Marketplace certification policies。

### 1.4 Preview audience（强烈建议）
- 用 Azure subscription IDs 邀请预览受众（最多可导入 CSV）
- 先做端到端部署测试，再上 Live

### 1.5 Plans and pricing（必须至少一个 plan）
- Container offer 至少需要一个 plan
- 支持的 licensing/billing 模型包括：Free、BYOL、以及多种按集群/节点/Pod/CPU 的预定义计费模型，另有 custom meters（对 Arc 有限制）

---

## 2. 技术实现侧：CNAB 技术包要做的事（清单）

### 2.1 选择并准备技术资产
官方要求以 **CNAB** 打包，CNAB 由以下 artifact 组成：

- Helm chart
- `createUiDefinition.json`
- ARM template（用于部署 extension/可选部署 AKS）
- manifest file（描述包内容与路径）

前置要求：
- 应用必须是 Helm chart-based
- Helm chart 不能包含 `.tgz`（所有文件解包放入仓库）
- chart 中必须包含全部镜像引用与 digest 信息，运行时不能再下载其他 chart/image

### 2.2 准备 ACR，并授权 Microsoft 复制你的 CNAB

发布流程会将 CNAB 从你的 ACR 深拷贝到 Microsoft-owned ACR。官方流程包含：

- 为 Microsoft 的 first-party 应用创建 service principal（app id 固定：`32597670-3e15-4def-8851-614ff48c1efa`）
- 在你的 ACR 上给该 SP 赋予 `acrpull`
- 在同一订阅注册 `Microsoft.PartnerCenterIngestion` resource provider

（命令与详细步骤见官方技术资产文档）

### 2.3 打包与上传：使用 `container-package-app`

官方提供 `container-package-app` 工具（以 Docker 镜像形式提供：`mcr.microsoft.com/container-package-app:latest`），用于：

- 校验 artifacts
- 构建 CNAB
- 上传到你的 ACR

基本流程：
- `cpa verify`：逐项验证
- `cpa buildbundle`：构建并上传

建议把打包过程固化到 CI（Azure Pipelines / GitHub Actions），减少本机环境差异。

---

## 3. 计费模型相关实现（按选择的 licensing option 补齐）

- 选择 *Per core / Per pod / Per node* 等计费模型时，官方要求在 workload YAML 中加入 billing identifier label：
  - `azure-extensions-usage-release-identifier`
- 对 *perCore* 计费模型，需要在容器 manifest 中指定 CPU request（`resources:requests`）
- 对 *custom meters*，需要在 Helm 的 `values.yaml` 中按官方字段约定补充 marketplace/identity/extension 相关字段，部署时由 cluster extensions 替换为实际值

---

## 4. 测试与排障（发布前必须做）

- Helm：在本地或测试集群验证可安装（例如 `helm install --dry-run --debug`）
- UI：用 Azure Portal 的 createUiDefinition sandbox 预览部署 UI
- 按官方提供的 sample repo 对照目录结构与 manifest 写法：
  - https://github.com/Azure-Samples/kubernetes-offer-samples

---

## 5. 最小可执行 checklist（建议直接变成工单）

- 确认适用范围：Kubernetes app（AKS/Arc），Linux AMD64，非 single container
- 准备 Helm chart + UI definition + ARM template + manifest
- 准备 ACR，并完成 Microsoft 复制 CNAB 所需的 SP/RBAC/RP 注册
- 用 `container-package-app` 完成验证与 CNAB 上传（并纳入 CI）
- Partner Center：Offer + Listing + Leads + Plan/Pricing + Preview audience
- 端到端测试：预览受众订阅部署、升级（tag 更新需重新提交 CNAB）、卸载
- 提交认证与修复反馈
- 发布与运维：漏洞扫描、版本管理、支持流程

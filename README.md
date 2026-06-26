# Hermes Agent → GKE 部署方案(评审稿)

把 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 部署到 **GKE Autopilot**,LLM 用 **Gemini**,消息平台用 **Telegram**。

> ⚠️ 本仓库是**方案/评审稿**,不是已执行的部署。目的是供专业人士审阅后再决定是否实施。**不含任何真实密钥**(均为占位符)。

---

## 1. 背景与现状

Hermes Agent 是 Nous Research 的自进化 AI agent(Python,自带 gateway 多平台接入、记忆/技能、cron、子代理)。

**当前已在一台 GCP VM 上以单个 Docker 容器运行**,且已经接通:
- **LLM = Gemini**:provider `google`,模型 `gemini-2.5-flash`,通过 `GEMINI_API_KEY`/`GOOGLE_API_KEY`。
- **平台 = Telegram**:走 `getUpdates` 长轮询(纯出站),已锁定单一允许用户。
- **状态**:全部落在数据目录 `/opt/data`(config、API keys、**SQLite 的记忆/会话/技能**、`channel_directory.json`)。
- 镜像基于公开镜像 `nousresearch/hermes-agent`,配置与密钥靠挂载/环境变量注入(未打进镜像)。

本方案要做的是:**把这个"已经能跑的有状态容器"搬到 GKE 托管**(为了托管化运维 + 后续可能的 CI/CD),而不是从零开发。

## 2. 目标与已定决策

| 项 | 决策 |
|----|------|
| 计算平台 | **GKE Autopilot**(免管节点) |
| 集群 | **新建**;区域 `us-central1`,项目 `gwsjoseph0326` |
| 数据 | **迁移**现有 `/opt/data`(保留记忆 / Telegram 已知会话) |
| 范围 | **只跑 Telegram gateway**(不跑 dashboard、不开 API server) |
| 发布方式 | 先**手动 `kubectl apply`** 跑通(CI/CD 以后再说) |
| 成本 | **已接受** GKE 基础成本 |

## 3. 为什么是 GKE,而不是 Cloud Run

Hermes 主体是**有状态 + 单实例 + 常驻**的:

| Hermes 的性质 | 与 Cloud Run 冲突 | GKE 如何承接 |
|--------------|------------------|-------------|
| SQLite 持久状态在 `/opt/data` | Cloud Run 文件系统临时、随时回收 → 状态丢 | **PVC**(持久卷) |
| Telegram `getUpdates` 单轮询 | Cloud Run 缩容到 0 会杀掉常驻轮询 | **常驻 1 副本** |
| 同时只能 1 个轮询者(否则 409) | 多实例会冲突 | **单副本 + `Recreate` 策略** |
| 出站长连接、非请求驱动 | Cloud Run 以入站 HTTP 计费/保活 | GKE 无此约束 |

详见 [`docs/considerations.md`](docs/considerations.md)。

## 4. 目标架构

```
Telegram (getUpdates 长轮询, 纯出站)
        ▲
        │ outbound
┌───────┴─────────────────────────────────────┐
│ GKE Autopilot 集群 (us-central1)             │
│  namespace: hermes                           │
│   ┌─────────────────────────────────────┐    │
│   │ Deployment hermes (replicas=1,       │    │
│   │   strategy: Recreate)                │    │
│   │   image: nousresearch/hermes-agent   │    │
│   │   args: ["gateway","run"]            │    │
│   │   envFrom: Secret hermes-secrets ────┼──▶ Gemini API (google)
│   │   volumeMount /opt/data ──┐          │    │
│   └───────────────────────────┼──────────┘    │
│   PVC hermes-data (RWO 10Gi) ◀┘               │
│   Secret hermes-secrets (密钥, 不入仓库)       │
└───────────────────────────────────────────────┘
（无 Service / Ingress —— Telegram 是纯出站,不需要入站入口）
```

## 5. 清单文件

`k8s/` 下是可评审的 manifests(均为最小可用 + 安全占位):
- `namespace.yaml`
- `pvc.yaml` —— `/opt/data` 的持久卷(RWO 10Gi)
- `secret.example.yaml` —— **仅占位符**,真实密钥用 `kubectl create secret` 带外创建
- `deployment.yaml` —— 单副本 / `Recreate` / s6 入口 / fsGroup

执行步骤见 [`docs/runbook.md`](docs/runbook.md)。

## 6. 成本(供评审)

| 项 | 估算/月 |
|----|--------|
| Autopilot pod(~0.5 vCPU / 1Gi,24×7) | ~$20 |
| 集群管理费 | ~$73(**每个计费账号有 1 个集群的免费额度可抵** → 若未占用则 ≈$0) |
| PVC(10Gi balanced PD) | ~$1 |
| Gemini token 用量 | 另算(Hermes 上下文大,**输入 token 是成本大头**) |
| **合计(不含 token)** | **~$20–95/月**,取决于免费额度 |

> 对比:现状(VM 上一个容器)≈ 白嫖。上 GKE 的收益是**托管运维**,不是省钱。

## 7. 主要风险

- **迁移期短暂停机**:停 VM 容器 → copy 数据 → GKE 起新 pod,Telegram 会离线分钟级。
- **双轮询冲突(409)**:迁移后 VM 容器**必须保持停用**;GKE 用 `Recreate` 防止更新时两个 pod 并存。
- **数据一致性**:必须**停容器后再拷** SQLite,避免拷到写一半的库。
- **回滚安全**:迁移只**复制**、不删除 VM 上的原数据,失败可 `docker start` 秒回滚(见 runbook)。

## 8. 给评审者的开放问题

见 [`docs/considerations.md`](docs/considerations.md) 末尾「Open questions」——包括是否值得上 GKE、HA/备份、资源规格、密钥管理(K8s Secret vs Secret Manager)、agent 在 pod 内执行命令的安全边界等。

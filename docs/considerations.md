# 部署考量与开放问题

## 逐项考量

### 1. 状态持久化(关键)
Hermes 所有状态在 `/opt/data`:config、API keys、**SQLite**(记忆/会话/技能)、`channel_directory.json`。
- 必须 **PVC**(`ReadWriteOnce`)。
- SQLite 单写 → **只能 1 副本**。
- 用 `Deployment + strategy: Recreate`(或 StatefulSet)。本方案用 Deployment+Recreate,因为更便于把现有数据预先灌入一个已建好的 PVC。

### 2. 单轮询约束(Telegram 409)
Telegram `getUpdates` 同一 bot 同时只能一个轮询者,否则 `409 Conflict`。
- 任何时刻只能有 **1 个 pod** 在跑。
- 滚动更新若新旧 pod 并存会触发 409 → **必须 `Recreate`**(先终止旧 pod 再起新 pod)。
- 迁移后,VM 上原容器必须**保持停用**。

### 3. 密钥管理
需要:`GEMINI_API_KEY`、`GOOGLE_API_KEY`(同值)、`TELEGRAM_BOT_TOKEN`、`TELEGRAM_ALLOWED_USERS`。
- 本方案:**K8s Secret**(`envFrom`),真实值**带外** `kubectl create secret`,不入仓库。
- 可选升级:GCP **Secret Manager + Secrets Store CSI Driver** 或 External Secrets(评审项)。

### 4. 网络
Telegram 走 `getUpdates` = **纯出站**。
- **不需要 Service / Ingress / LoadBalancer / TLS**。
- 仅需集群有出站到公网(Autopilot 默认可出站)。
- 若将来加 Discord/webhook 适配器(入站)才需要 Service+Ingress。

### 5. 不跑的组件
- **dashboard**:存 API key、官方明确"勿裸暴露"。本方案**不部署**;如需调试用 `kubectl port-forward` 临时访问,绝不挂公网。
- **OpenAI 兼容 API server**:默认关闭,本方案不开。

### 6. terminal backend(安全注意)
Hermes 默认 `local` 后端:**工具命令在 pod 内执行**。
- 不需要 DinD / 特权容器。
- 但意味着 agent 可在 pod 内跑命令 → 评审应关注:pod 的最小权限、出站网络管控、是否需要 NetworkPolicy 限制 egress。
- 若要 Docker 沙箱隔离子代理,GKE 上需 DinD/gVisor —— **本方案不启用**。

### 7. 镜像与入口
- 镜像:公开 `nousresearch/hermes-agent`(可直接 pull;也可镜像到 Artifact Registry 防 Docker Hub 限流——评审项)。
- 入口:镜像 `ENTRYPOINT=/init`(s6-overlay PID1,做 chown/usermod/监督树)。**K8s 里只覆盖 `args: ["gateway","run"]`,不要覆盖 `command`**,否则跳过 s6 初始化导致 gateway 异常。
- `/init` 需以 **root** 启动(再 `s6-setuidgid` 降权到 uid 10000)→ **不要设 `runAsNonRoot`**;用 `fsGroup: 10000` 让 PVC 可被降权后的用户写。

### 8. 资源规格
- 起步 `requests: 500m CPU / 1Gi`,`limits: 1 CPU / 2Gi`。
- Hermes 上下文大,内存/CPU 视实际负载调整(评审项:是否够)。
- Autopilot 会按其允许档位对 requests 做圆整。

### 9. 数据迁移
- 现有数据在 VM 的 `~/hermes-gemini-data/`(=容器内 `/opt/data`)。
- 迁移:停容器 → 临时 pod 挂 PVC → `kubectl cp` → 删临时 pod。
- 只复制不删除原数据 → 失败可秒回滚。

### 10. 备份
- PVC 背后是 GCE Persistent Disk → 可用 **PD 快照**定期备份(评审项:备份频率/保留)。

---

## Open questions(给专业评审)

1. **是否值得上 GKE?** 单个有状态 pod,GKE 偏重;收益主要是托管运维 + 未来 CI/CD。是否考虑更轻的方案(继续 VM 容器 / Compute Engine MIG)?
2. **HA 与可用性**:受 SQLite + 单轮询限制,**无法横向扩容/多副本**。可接受单点吗?故障时靠 Deployment 重建(分钟级恢复)+ PD 快照,够吗?
3. **资源规格**:`0.5–1 vCPU / 1–2Gi` 对 Hermes 的实际工作集是否合适?
4. **密钥管理**:用普通 K8s Secret 够吗,还是上 Secret Manager + CSI / External Secrets?
5. **agent 执行安全**:`local` 后端让 agent 在 pod 内跑命令——是否需要 NetworkPolicy 限制 egress、只读根文件系统、seccomp、专用最小权限 ServiceAccount?
6. **镜像供应链**:直接用 Docker Hub `nousresearch/hermes-agent:latest` 是否可接受?是否应固定到 digest 并镜像到 Artifact Registry?
7. **成本**:~$20–95/月(看 Autopilot 免费集群额度是否已被占用)+ Gemini token,是否 OK?
8. **备份/DR**:PVC 快照频率与恢复演练。
9. **滚动更新**:确认 `Recreate` 能避免 Telegram 双轮询;升级流程是否需要更细的编排(先 `kubectl scale --replicas=0` 再更新)?
10. **可观测性**:是否需要把日志/指标接 Cloud Logging/Monitoring,以及 token 用量看板(现有看板在另一个项目)。

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

## Open questions & 评审决议

1. **是否值得上 GKE?**
   * **决议**: 依然以 GKE Autopilot 作为主要方向（用于实验学习/后续 CI/CD，且每个计费账号有 1 个免费集群额度）。同时，将 **GCP VM (COS) + 独立 PD + 快照** 作为 **Plan B** 记录（该方案无集群管理费，且容器启停仅需数秒，磁盘无需跨节点 detach/attach，更轻量）。
   * **规格调整**: 若使用 VM 运行，由于 Hermes 进程（Python + SQLite + LLM 较大上下文）开销，`e2-micro` (1GB) 偏紧，推荐起步使用 **`e2-small` (2GB 内存)**。

2. **HA 与可用性 & 节点升级离线**
   * **决议**: 接受单副本有状态 Pod 的单点局限性。在 GKE Autopilot 自动维护升级节点时，单副本 Pod 会被驱逐重建，造成分钟级离线。可考虑通过配置 GCP 维护窗口（Maintenance Window）将离线时间窗口规避开核心日间时段。

3. **资源规格 (Pod)**
   * **决议**: Pod 设置 `requests: 500m CPU / 1Gi`,`limits: 1 CPU / 2Gi` 级别比较合理，后续可根据 Vertical Pod Autoscaler (VPA) 推荐值进行右调（Right-sizing）。

4. **密钥管理**
   * **决议**: 采用 K8s Secret 配合 `automountServiceAccountToken: false`（彻底切断 Pod 对 K8s API 越权访问）目前已足够。

5. **Agent 执行安全 (Pod 内跑命令)**
   * **决议**:
     * 必须在 Pod spec 中设置 `automountServiceAccountToken: false`。
     * 由于 GKE Autopilot 默认强制启用 Workload Identity，未绑定 GSA 的 Pod 会被 GKE metadata server 自动拦截节点凭证，故 metadata 凭证窃取属于“纵深防御（Defense-in-depth）”范畴，非高危阻塞。
     * 新增 `k8s/networkpolicy.yaml`，使用基于 `ipBlock` 的 egress 白名单，放行公网 DNS (53) 和 HTTPS (443) 流量，同时精确排除 Metadata Server IP (`169.254.169.254/32`)。

6. **镜像供应链**
   * **决议**: 在生产部署中避免直接引用 `:latest`。推荐锁定到特定版本 digest，并镜像到 Google Artifact Registry (GAR)，以规避 Docker Hub 限流及不可预期的镜像更新/漂移。

7. **优雅停机与进程挂死自愈 (新增)**
   * **决议**:
     * **存活探针**: 由于 Gateway 无 HTTP 接口，添加 Exec 探针运行 `pgrep -f "gateway run"` 以监控并自动重启死锁挂死的 python 进程。
     * **优雅停机**: Pod 模版中设置 `terminationGracePeriodSeconds: 30`，确保 s6-overlay 能将 `SIGTERM` 信号正确传递至 python 进程，优雅关闭 Telegram `getUpdates` 轮询，避免滚动更新时发生 409 Conflict。

8. **备份/DR**:PVC 快照频率与恢复演练。
9. **滚动更新**:确认 `Recreate` 能避免 Telegram 双轮询;升级流程是否需要更细的编排(先 `kubectl scale --replicas=0` 再更新)?
10. **可观测性**:是否需要把日志/指标接 Cloud Logging/Monitoring,以及 token 用量看板(现有看板在另一个项目)。

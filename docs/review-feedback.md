# Hermes Agent GKE 部署方案评审报告

这份部署方案整体设计非常严谨、专业，准确地识别了 **SQLite 单写限制**、**Telegram 409 冲突** 以及 **s6-overlay 容器入口初始化** 等核心技术细节。

为了确保实际部署顺利且安全，我们在以下三个维度为您梳理了优化建议：**关键阻塞性问题 (必须修改)**、**安全加固建议 (强烈推荐)** 以及 **架构方案权衡 (供讨论)**。

---

## 1. 关键阻塞性问题 (必须修改)

### ⚠️ 数据迁移过程中的文件权限与属主问题
*   **问题描述**: 
    在 `docs/runbook.md` 第 4 步中，使用 `kubectl cp` 将 VM 的数据拷贝到 `migrator` 临时 pod 的 `/opt/data` 目录下。
    由于 `kubectl cp` 底层使用 `tar`，它会**保留原 VM 文件系统的 UID/GID 和权限限制**。
    
    而在 `k8s/deployment.yaml` 中，实际运行的 Pod 在 s6-overlay 初始化并降权后，以 UID `10000` (hermes 用户) 运行。
    
    如果原 VM 上的文件（如 SQLite `.db` 数据库）属于 VM 的普通用户（如 UID `1000`），且文件权限是 `600` (仅所有者可读写)，则降权后的 hermes 进程 (UID `10000`) 将**无法读写 SQLite 数据库**。这会导致 Pod 启动后抛出 `Permission Denied` 异常，进而陷入 CrashLoopBackOff。
*   **解决方案**:
    在数据拷贝完成、删除 `migrator` 容器前，在 `migrator` 容器中执行 `chown` 和 `chmod` 命令，强制修复挂载卷内所有文件 and 目录的属主与权限。
*   **Runbook 修改建议**:
    在 `docs/runbook.md` 的拷贝步骤后增加两行命令：
    ```bash
    # 拷贝(注意结尾的 . 表示拷目录内容)
    kubectl -n $NS cp ~/hermes-gemini-data/. migrator:/opt/data
    
    # 修正文件属主与权限，确保 UID 10000 拥有完整读写权限
    kubectl -n $NS exec migrator -- chown -R 10000:10000 /opt/data
    kubectl -n $NS exec migrator -- chmod -R ug=rwX,o= /opt/data
    
    kubectl -n $NS exec migrator -- ls -la /opt/data   # 核对
    kubectl -n $NS delete pod migrator
    ```

---

## 2. 安全加固建议 (强烈推荐)

### 🔒 限制 Agent 在 Pod 内本地执行命令的安全边界
在配置中使用 `local` 执行后端意味着 Hermes Agent 的指令或工具链将直接在容器内部执行。如果 Agent 遭到 Prompt 注入或运行了恶意/不受控的命令，它将拥有容器内的全部操作权限。在 Kubernetes 环境下，这带来了以下次生风险：

1.  **Kubernetes API 访问风险**:
    默认情况下，Kubernetes 会将 Pod 对应 ServiceAccount 的 Token 挂载在容器内的 `/var/run/secrets/kubernetes.io/serviceaccount/token`。Agent 如果读取该 Token，就可以调用 API Server 尝试对集群进行越权操作。
    *   **优化建议**: 在 `k8s/deployment.yaml` 的 Pod Spec 中，明确禁用 ServiceAccount Token 自动挂载：
        ```yaml
        spec:
          automountServiceAccountToken: false  # 禁用 Token 挂载，切断对 K8s API 的访问路径
          securityContext:
            fsGroup: 10000
        ```

2.  **GCP Metadata Server 凭证窃取风险**:
    GKE Pod 默认能够直接请求 GCP 元数据服务器 `http://metadata.google.internal` (IP: `169.254.169.254`)。
    如果您的 GKE 节点使用的是默认的 Compute Engine 服务账号（通常拥有很高的 IAM 权限），Agent 就可以通过请求元数据直接获取 GCP 访问凭证，从而控制整个 GCP 项目！
    *   **优化建议 1 (Workload Identity)**: 在集群中启用 Workload Identity，并将部署的 Pod 绑定到一个**没有任何 IAM 权限**的 Google Service Account (GSA) 上。
    *   **优化建议 2 (NetworkPolicy 隔离)**: 使用 Kubernetes NetworkPolicy，限制该 Pod 只能向外访问 Telegram/Gemini API，禁止其访问本地元数据服务器的 IP：
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: hermes-egress-limit
          namespace: hermes
        spec:
          podSelector:
            matchLabels:
              app: hermes
          policyTypes:
          - Egress
          egress:
          # 允许解析 DNS
          - ports:
            - port: 53
              protocol: UDP
            - port: 53
              protocol: TCP
          # 允许公网出站 (Telegram / Gemini API)，但排除 169.254.169.254 (Metadata Server)
        ```

---

## 3. 性能与日常运维优化

### ⚡ 避免挂载卷大文件导致 Pod 启动变慢 (`fsGroupChangePolicy`)
*   **问题描述**: 
    当配置了 `fsGroup: 10000` 时，Kubernetes 的默认行为是在**每次 Pod 启动时**，递归地对挂载卷（即 `/opt/data` 目录）下的所有文件执行一次 `chmod`/`chown` 以匹配 fsGroup。
    随着 Hermes 运行时间变长，SQLite 数据库、历史会话日志以及临时缓存文件会越来越多。这会导致 Pod 启动时的挂载变慢，甚至引发健康检查超时。
*   **优化建议**:
    在 Deployment 的 `securityContext` 中加入 `fsGroupChangePolicy: OnRootMismatch`。这样只有在根目录权限不匹配时才执行递归权限修改，极大地缩短 Pod 的重启时间。
*   **YAML 优化示例**:
    ```yaml
    spec:
      securityContext:
        fsGroup: 10000
        fsGroupChangePolicy: OnRootMismatch  # 避免每次启动递归 chown
    ```

---

## 4. 方案架构探讨 (GKE Autopilot vs 独立 VM)

您在开放问题中提到了「**是否值得上 GKE**」，这确实是一个非常值得商榷的问题。以下是两种方案的对比：

| 维度 | GKE Autopilot 方案 | 独立 VM (COS) + 独立 Persistent Disk 方案 |
| :--- | :--- | :--- |
| **基础成本** | 如果 billing account 中已有一个集群，GKE 集群管理费为 $0；否则需要约 **$73/月** 的固定开销。Pod 费用约 **$20/月**。 | 仅需一个较小规格的 VM (例如 `e2-micro` 或 `e2-small`) 加上独立数据盘，总费用在 **$5 - $15/月**。 |
| **更新停机时间** | 由于使用的是 block storage (GCE PD RWO) 且副本数强制为 1：<br>在滚动更新 (`Recreate`) 时，需要先终止旧 Pod -> Detach 磁盘 -> Attach 磁盘 -> 启动新 Pod。更新过程会带来 **1-2 分钟的离线时间**。 | 容器内重启或 `docker compose down && docker compose up -d` 仅需 **5-10 秒**，磁盘无需跨节点 detach/attach。 |
| **高可用与自愈** | Node 故障或 Pod Crash 时，GKE 会在其他健康节点上自动拉起（自愈能力强）。 | VM 故障或进程挂掉时，需要依赖 `systemd` 或守护进程自启。 |
| **备份运维** | 需要配置 K8s 级别的 VolumeSnapshot，或带外写脚本调度 GCE PD 快照。 | 可以直接在 GCP 侧针对独立 Persistent Disk 配置自动快照策略 (Snapshot Schedule)，更加简单直观。 |

### 💡 架构建议：
1.  如果您目前已有现成的 GKE Autopilot 集群，或者想要搭建 CI/CD 自动化流水线，那么采用 GKE 部署是合适的（只需注意解决上述权限与安全问题）。
2.  如果您是独立运行该 Bot，追求**低成本**、**快速启停（秒级更新）**且**运维简单**，推荐的重构方案是：
    *   在 GCP 上使用 **Container-Optimized OS (COS)** 镜像创建一台 VM。
    *   创建一个**独立的 Persistent Disk (PD)** 并将其挂载 to VM 的特定目录（例如 `/mnt/disks/hermes-data`）。
    *   将 Docker 的持久化数据存放在该 PD 上，这样即使 VM 实例被重建、升级、销毁，您的 SQLite 状态也绝不会丢失。
    *   在 GCP 控制台直接为该 PD 配置 **Snapshot Schedule (定期自动快照)**，完全免去了 GKE 的复杂运维与集群管理成本。

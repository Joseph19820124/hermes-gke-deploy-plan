# 执行 Runbook(评审稿,尚未执行)

> 所有命令均为**拟执行**,供评审。实际执行前请逐条确认。
> 占位:`PROJECT=gwsjoseph0326`、`REGION=us-central1`、`CLUSTER=hermes-cluster`、`NS=hermes`。
> VM 上现有数据目录:`~/hermes-gemini-data/`(= 容器 `/opt/data`),运行中的容器名 `hermes-gemini`。

## 0. 前置:导出现有密钥(在停容器之前!)
从运行中的容器读出要复用的密钥(不写入仓库):
```bash
sudo docker exec hermes-gemini env | grep -E '^(GEMINI_API_KEY|GOOGLE_API_KEY|TELEGRAM_BOT_TOKEN|TELEGRAM_ALLOWED_USERS)='
```
启用 API + 取 GKE 认证插件:
```bash
gcloud services enable container.googleapis.com --project=$PROJECT
gcloud components install gke-gcloud-auth-plugin   # 或 apt 包 google-cloud-cli-gke-gcloud-auth-plugin
```

## 1. 建 Autopilot 集群(~5–10 分钟)
```bash
gcloud container clusters create-auto $CLUSTER \
  --region=$REGION --project=$PROJECT
gcloud container clusters get-credentials $CLUSTER \
  --region=$REGION --project=$PROJECT
```

## 2. 建 namespace + PVC
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/pvc.yaml
```

## 3. ⚠️ 停掉 VM 上的容器(避免双轮询 / 拿一致的 SQLite 快照)
```bash
sudo docker stop hermes-gemini
```
> 从这一步到第 6 步验证完成,Telegram bot 会离线(分钟级)。

## 4. 迁移数据到 PVC
起一个临时 pod 挂上 PVC,把数据拷进去:
```bash
kubectl -n $NS apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: migrator
spec:
  containers:
    - name: shell
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /opt/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: hermes-data
EOF

kubectl -n $NS wait --for=condition=Ready pod/migrator --timeout=120s
# 拷贝(注意结尾的 . 表示拷目录内容)
kubectl -n $NS cp ~/hermes-gemini-data/. migrator:/opt/data
kubectl -n $NS exec migrator -- ls -la /opt/data   # 核对
kubectl -n $NS delete pod migrator
```

## 5. 建 Secret(带外,用第 0 步读到的真实值)
```bash
kubectl -n $NS create secret generic hermes-secrets \
  --from-literal=GEMINI_API_KEY='...' \
  --from-literal=GOOGLE_API_KEY='...' \
  --from-literal=TELEGRAM_BOT_TOKEN='...' \
  --from-literal=TELEGRAM_ALLOWED_USERS='...'
```

## 6. 部署并验证
```bash
kubectl apply -f k8s/deployment.yaml
kubectl -n $NS rollout status deploy/hermes
kubectl -n $NS logs -f deploy/hermes        # 看 gateway 起来、Telegram 连上
```
在 Telegram 给 bot 发条消息,确认能回复。

## 7. 收尾
- 验证 OK → VM 上 `hermes-gemini` 容器**保持停用**(可设不随 Docker 自启;别 `docker start`,否则双轮询 409)。
- 建议:给 PVC 背后的 PD 配置定期快照。

---

## 回滚(任一步失败)
GKE 侧迁移**只复制、未删除** VM 上的原数据,所以可秒回滚:
```bash
# 删除 GKE 上的部署(可选,先停 pod 释放 Telegram 轮询)
kubectl -n $NS delete deploy hermes
# VM 上恢复原容器
sudo docker start hermes-gemini
```
彻底放弃时再删集群:
```bash
gcloud container clusters delete $CLUSTER --region=$REGION --project=$PROJECT
```

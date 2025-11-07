# 部署指南

本文档介绍如何使用GitHub Actions和Helm部署nof1-tracker到Kubernetes集群。

## 目录

- [GitHub Actions自动构建](#github-actions自动构建)
- [Helm Chart部署](#helm-chart部署)
- [配置说明](#配置说明)
- [故障排除](#故障排除)

## GitHub Actions自动构建

项目已配置GitHub Actions工作流，可以自动构建Docker镜像并推送到GitHub Container Registry。

### 工作流触发条件

- 推送到`main`或`master`分支
- 创建版本标签（如`v1.0.0`）
- 针对主分支的Pull Request

### 镜像标签策略

- 分支名称：`main`、`master`等
- PR编号：`pr-123`
- 语义化版本：`v1.0.0`、`v1.0`、`v1`
- Git提交SHA：`a1b2c3d`

### 查看构建状态

1. 访问GitHub仓库的"Actions"页面
2. 点击"Build and Push Docker Image"工作流
3. 查看构建日志和状态

### 手动触发构建

如果需要手动触发构建，可以：

1. 在GitHub仓库中创建新的发布标签：
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```

2. 或者直接推送到主分支：
   ```bash
   git push origin main
   ```

## Helm Chart部署

### 前置条件

- Kubernetes集群（v1.20+）
- Helm 3.x
- kubectl配置正确

### 安装步骤

1. **克隆仓库并进入Helm目录**：
   ```bash
   git clone https://github.com/terryso/nof1-tracker.git
   cd nof1-tracker/helm/nof1-tracker
   ```

2. **创建自定义values文件**：
   ```bash
   cp values.yaml my-values.yaml
   ```

3. **编辑my-values.yaml，配置敏感信息**：
   ```yaml
   # 镜像配置
   image:
     registry: ghcr.io
     repository: terryso/nof1-tracker
     tag: "v1.1.1"  # 使用具体版本标签

   # 密钥配置
   secrets:
     binance:
       apiKey: "your_binance_api_key"
       apiSecret: "your_binance_api_secret"
     telegram:
       botToken: "your_telegram_bot_token"
       chatId: "your_telegram_chat_id"

   # 环境变量
   env:
     LOG_LEVEL: "INFO"
     BINANCE_TESTNET: "false"  # 生产环境设为false
     TELEGRAM_ENABLED: "true"  # 启用Telegram通知
   ```

4. **安装Helm Chart**：
   ```bash
   helm install my-nof1-tracker . -f my-values.yaml
   ```

5. **验证部署**：
   ```bash
   kubectl get pods -l app.kubernetes.io/name=nof1-tracker
   kubectl logs -l app.kubernetes.io/name=nof1-tracker
   ```

### 升级部署

当需要更新配置或应用版本时：

```bash
helm upgrade my-nof1-tracker . -f my-values.yaml
```

### 卸载部署

```bash
helm uninstall my-nof1-tracker
```

## 配置说明

### 镜像配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `image.registry` | `ghcr.io` | 镜像仓库地址 |
| `image.repository` | `terryso/nof1-tracker` | 镜像仓库名称 |
| `image.tag` | `latest` | 镜像标签，建议使用具体版本 |
| `image.pullPolicy` | `IfNotPresent` | 镜像拉取策略 |

### 资源配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `resources.limits.cpu` | `500m` | CPU限制 |
| `resources.limits.memory` | `512Mi` | 内存限制 |
| `resources.requests.cpu` | `250m` | CPU请求 |
| `resources.requests.memory` | `256Mi` | 内存请求 |

### 密钥配置

密钥通过Kubernetes Secret管理，确保敏感信息安全：

- `secrets.binance.apiKey`: Binance API密钥
- `secrets.binance.apiSecret`: Binance API密钥
- `secrets.telegram.botToken`: Telegram机器人令牌（可选）
- `secrets.telegram.chatId`: Telegram聊天ID（可选）

### 环境变量

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `env.LOG_LEVEL` | `INFO` | 日志级别 |
| `env.BINANCE_TESTNET` | `true` | 是否使用测试网 |
| `env.TELEGRAM_ENABLED` | `false` | 是否启用Telegram通知 |

### 持久化存储

如果需要持久化数据，可以启用持久化存储：

```yaml
persistence:
  enabled: true
  storageClass: "standard"  # 根据集群调整
  accessMode: ReadWriteOnce
  size: 1Gi
```

## 高级配置

### 自动扩缩容

启用HPA（水平Pod自动扩缩容）：

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

### 节点选择

将Pod调度到特定节点：

```yaml
nodeSelector:
  node-type: trading
```

### 亲和性和反亲和性

配置Pod亲和性：

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app.kubernetes.io/name
          operator: In
          values:
          - monitoring
      topologyKey: kubernetes.io/hostname
```

## 故障排除

### Pod无法启动

1. 检查Pod状态：
   ```bash
   kubectl describe pod -l app.kubernetes.io/name=nof1-tracker
   ```

2. 查看Pod日志：
   ```bash
   kubectl logs -l app.kubernetes.io/name=nof1-tracker
   ```

3. 常见问题：
   - 镜像拉取失败：检查镜像标签和仓库访问权限
   - 密钥配置错误：验证Secret中的base64编码是否正确
   - 资源不足：调整资源限制和请求

### 密钥配置问题

1. 检查Secret内容：
   ```bash
   kubectl get secret my-nof1-tracker-secret -o yaml
   ```

2. 手动解码验证：
   ```bash
   echo "encoded_value" | base64 -d
   ```

3. 更新Secret：
   ```bash
   kubectl create secret generic my-nof1-tracker-secret \
     --from-literal=BINANCE_API_KEY="your_key" \
     --from-literal=BINANCE_API_SECRET="your_secret" \
     --dry-run=client -o yaml | kubectl apply -f -
   ```

### 网络问题

1. 检查Service配置：
   ```bash
   kubectl get svc my-nof1-tracker
   ```

2. 测试网络连通性：
   ```bash
   kubectl exec -it <pod-name> -- ping binance.com
   ```

## 安全建议

1. **使用RBAC**：限制ServiceAccount权限
2. **网络策略**：限制Pod间通信
3. **Pod安全策略**：启用安全上下文
4. **密钥管理**：考虑使用外部密钥管理系统（如Vault）
5. **镜像扫描**：定期扫描镜像漏洞

## 监控和日志

### 查看日志

```bash
# 实时查看日志
kubectl logs -f -l app.kubernetes.io/name=nof1-tracker

# 查看最近100行日志
kubectl logs -l app.kubernetes.io/name=nof1-tracker --tail=100
```

### 监控指标

考虑集成Prometheus和Grafana进行监控：

1. 在values.yaml中启用监控：
   ```yaml
   monitoring:
     enabled: true
     serviceMonitor:
       enabled: true
   ```

2. 配置Prometheus规则和Grafana仪表板

## 备份和恢复

### 备份

1. 备份Helm Release：
   ```bash
   helm get values my-nof1-tracker > backup-values.yaml
   ```

2. 备份持久化数据：
   ```bash
   kubectl exec -it <pod-name> -- tar czf /tmp/backup.tar.gz /app/data
   kubectl cp <pod-name>:/tmp/backup.tar.gz ./backup.tar.gz
   ```

### 恢复

1. 恢复Helm Release：
   ```bash
   helm install my-nof1-tracker . -f backup-values.yaml
   ```

2. 恢复持久化数据：
   ```bash
   kubectl cp ./backup.tar.gz <pod-name>:/tmp/backup.tar.gz
   kubectl exec -it <pod-name> -- tar xzf /tmp/backup.tar.gz -C /app/data
   ```

## 更多资源

- [Helm官方文档](https://helm.sh/docs/)
- [Kubernetes官方文档](https://kubernetes.io/docs/)
- [nof1-tracker项目主页](https://github.com/terryso/nof1-tracker)

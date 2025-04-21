# 容器化與 Kubernetes 部署

容器化技術（如 Docker）和容器編排工具（如 Kubernetes）是大規模部署和微服務架構中的重要工具。Celery、Kombu 和 RabbitMQ 可以通過容器化實現靈活的部署和管理。本章節將介紹如何將這些工具容器化，並在 Kubernetes 上進行部署，提供具體步驟和最佳實踐。

## 容器化的優勢

容器化為分散式任務處理系統帶來以下優勢：
- **一致性**：開發、測試和生產環境一致，避免環境差異。
- **可移植性**：容器可以在任何支持 Docker 的平台上運行。
- **資源隔離**：每個容器獨立運行，避免資源爭用。
- **快速部署**：通過映像快速啟動和擴展服務。

## Docker 容器化

首先，我們需要為 Celery、RabbitMQ 和相關組件創建 Docker 映像。

### RabbitMQ 容器化

使用官方 RabbitMQ 映像，配置持久化和集群支持：

#### Dockerfile 示例
```dockerfile
FROM rabbitmq:3.9-management

# 啟用管理插件
RUN rabbitmq-plugins enable --offline rabbitmq_management

# 配置持久化目錄
VOLUME /var/lib/rabbitmq

# 暴露端口
EXPOSE 5672 15672
```

#### 啟動容器
```bash
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  -v rabbitmq-data:/var/lib/rabbitmq \
  rabbitmq-image
```

#### 最佳實踐
- 使用卷（Volume）持久化數據，避免容器重啟導致數據丟失。
- 配置環境變數設定用戶和密碼：

```bash
docker run -d --name rabbitmq \
  -e RABBITMQ_DEFAULT_USER=user \
  -e RABBITMQ_DEFAULT_PASS=password \
  rabbitmq-image
```

### Celery Worker 容器化

為 Celery Worker 創建自定義 Docker 映像：

#### Dockerfile 示例
```dockerfile
FROM python:3.9-slim

# 安裝依賴
RUN pip install celery kombu redis

# 複製應用程式代碼
WORKDIR /app
COPY . /app

# 啟動 Worker
CMD ["celery", "-A", "your_module", "worker", "--concurrency=4", "--loglevel=info"]
```

#### 啟動容器
```bash
docker run -d --name celery-worker \
  --link rabbitmq:rabbitmq \
  -e CELERY_BROKER_URL=amqp://user:password@rabbitmq:5672// \
  celery-worker-image
```

#### 最佳實踐
- 使用 `--link` 或自定義網絡連接 RabbitMQ 容器。
- 根據硬體資源調整 `concurrency` 參數。

### 使用 Docker Compose 管理多容器

使用 Docker Compose 管理 RabbitMQ、Celery Worker 和結果後端：

#### docker-compose.yml 示例
```yaml
version: '3'
services:
  rabbitmq:
    image: rabbitmq:3.9-management
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    environment:
      - RABBITMQ_DEFAULT_USER=user
      - RABBITMQ_DEFAULT_PASS=password

  redis:
    image: redis:6.2
    ports:
      - "6379:6379"

  celery-worker:
    build: .
    depends_on:
      - rabbitmq
      - redis
    environment:
      - CELERY_BROKER_URL=amqp://user:password@rabbitmq:5672//
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    command: ["celery", "-A", "your_module", "worker", "--concurrency=4"]

volumes:
  rabbitmq-data:
```

#### 啟動服務
```bash
docker-compose up -d
```

#### 最佳實踐
- 使用 `depends_on` 確保服務啟動順序。
- 定義卷和網絡，確保數據持久化和容器間通信。

## Kubernetes 部署

Kubernetes 提供容器編排功能，適合大規模部署和管理 Celery 與 RabbitMQ。

### RabbitMQ 在 Kubernetes 上部署

使用 StatefulSet 部署 RabbitMQ 集群，確保數據持久化：

#### RabbitMQ StatefulSet 示例
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3.9-management
        ports:
        - containerPort: 5672
        - containerPort: 15672
        env:
        - name: RABBITMQ_ERLANG_COOKIE
          value: "secret-cookie"
        volumeMounts:
        - name: rabbitmq-data
          mountPath: /var/lib/rabbitmq
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
  - port: 5672
    targetPort: 5672
    name: amqp
  - port: 15672
    targetPort: 15672
    name: management
  clusterIP: None  # Headless Service for StatefulSet
```

#### 最佳實踐
- 使用 StatefulSet 確保 RabbitMQ 節點穩定性和順序啟動。
- 配置持久化卷（PVC）儲存數據，避免 Pod 重啟導致數據丟失。

### Celery Worker 在 Kubernetes 上部署

使用 Deployment 部署 Celery Worker，支持自動擴展：

#### Celery Worker Deployment 示例
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: celery-worker
  template:
    metadata:
      labels:
        app: celery-worker
    spec:
      containers:
      - name: celery-worker
        image: your-celery-image:latest
        env:
        - name: CELERY_BROKER_URL
          value: "amqp://user:password@rabbitmq:5672//"
        - name: CELERY_RESULT_BACKEND
          value: "redis://redis:6379/0"
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
          requests:
            cpu: "0.5"
            memory: "256Mi"
        command: ["celery", "-A", "your_module", "worker", "--concurrency=4"]
---
apiVersion: v1
kind: Service
metadata:
  name: celery-worker
spec:
  selector:
    app: celery-worker
  ports:
  - port: 80
    targetPort: 80
```

#### 最佳實踐
- 設定資源限制（`resources`），避免 Worker 過載 Kubernetes 節點。
- 使用 HorizontalPodAutoscaler（HPA）根據 CPU 或隊列長度自動擴展：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: celery-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: celery-worker
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Redis 結果後端在 Kubernetes 上部署

使用 Helm Chart 快速部署 Redis 集群：

#### 部署步驟
1. 安裝 Helm（如果未安裝）：
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```
2. 使用 Helm 安裝 Redis：
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install redis bitnami/redis --set architecture=cluster
   ```

#### 最佳實踐
- 使用 Redis 集群模式支持高可用性和分片。
- 配置服務地址，確保 Celery Worker 能正確連接。

## Kubernetes 進階配置

### 健康檢查與重啟策略

為 RabbitMQ 和 Celery Worker 配置健康檢查，確保服務穩定：

#### 配置示例
在 Celery Worker 的 Deployment 中添加健康檢查：
```yaml
containers:
- name: celery-worker
  image: your-celery-image:latest
  livenessProbe:
    exec:
      command: ["celery", "inspect", "ping"]
    initialDelaySeconds: 30
    periodSeconds: 10
  readinessProbe:
    exec:
      command: ["celery", "inspect", "ping"]
    initialDelaySeconds: 5
    periodSeconds: 5
```

#### 最佳實踐
- 設定合理的檢查間隔和延遲，避免頻繁重啟。
- 結合重啟策略（`restartPolicy`），確保 Pod 在故障時能自動恢復。

### 網絡策略與安全

使用 Kubernetes 網絡策略限制容器間訪問：

#### 網絡策略示例
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: celery-rabbitmq-policy
spec:
  podSelector:
    matchLabels:
      app: celery-worker
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: rabbitmq
    ports:
    - port: 5672
      protocol: TCP
```

#### 最佳實踐
- 限制不必要的網絡訪問，增強系統安全性。
- 使用 Kubernetes Secrets 管理敏感信息（如 RabbitMQ 密碼）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-credentials
type: Opaque
data:
  username: dXNlcg==  # base64 encoded 'user'
  password: cGFzc3dvcmQ=  # base64 encoded 'password'
```

然後在 Deployment 中引用：
```yaml
env:
- name: RABBITMQ_USERNAME
  valueFrom:
    secretKeyRef:
      name: rabbitmq-credentials
      key: username
- name: RABBITMQ_PASSWORD
  valueFrom:
    secretKeyRef:
      name: rabbitmq-credentials
      key: password
```

### 日誌與監控

在 Kubernetes 中收集和分析日誌，監控系統狀態：

#### 最佳實踐
- 使用 EFK（Elasticsearch, Fluentd, Kibana）或 Loki 收集和查詢日誌。
- 整合 Prometheus 和 Grafana 監控 RabbitMQ 和 Celery 的關鍵指標：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: rabbitmq
  endpoints:
  - port: management
    path: /metrics
    interval: 30s
```

- 為 Celery Worker 配置 Flower 進行任務監控，並暴露為 Kubernetes 服務。

### 自動化與 CI/CD 整合

將 Kubernetes 部署整合到 CI/CD 流程中，實現自動化更新：

#### 最佳實踐
- 使用 Helm Chart 打包應用，簡化部署流程。
- 在 CI/CD 管道中運行單元測試和集成測試，確保代碼質量。
- 使用 GitOps 工具（如 ArgoCD）管理 Kubernetes 資源，實現持續部署。

## 總結與注意事項

- **容器化優勢**：使用 Docker 確保環境一致性，快速部署 Celery 和 RabbitMQ。
- **Kubernetes 編排**：利用 Kubernetes 的 StatefulSet 和 Deployment 管理 RabbitMQ 集群和 Celery Worker。
- **進階配置**：配置健康檢查、網絡策略和監控，確保系統穩定性和安全性。
- **自動化部署**：整合 CI/CD 和 GitOps 工具，提升部署效率。

通過容器化和 Kubernetes 部署，您可以構建一個靈活、可擴展且高可用的分散式任務處理系統，應對複雜的生產環境需求。

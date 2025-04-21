# 高可用配置

在生產環境中，確保 Celery、Kombu 和 RabbitMQ 系統的高可用性（High Availability, HA）至關重要，以避免單點故障導致的服務中斷。本章節將介紹如何配置這些工具以實現高可用性，從冗餘設計到故障恢復，提供具體的建議。

## 高可用性基礎

高可用性的目標是確保系統在面對硬體故障、網絡問題或軟體錯誤時仍能正常運行。

### 冗餘設計

冗餘是高可用性的核心，通過多節點部署避免單點故障：

- **Celery Worker**：在多台伺服器上運行多個 Worker。
- **RabbitMQ**：部署集群，確保消息代理可用。
- **結果後端**：使用具有冗餘功能的後端（如 Redis 集群）。

#### 最佳實踐
- 確保每個組件至少有兩個實例，分布在不同硬體或可用區。
- 使用負載均衡器分發流量，避免單一入口點故障。

### 故障檢測與切換

快速檢測故障並切換到備用資源是高可用性的關鍵。

#### 最佳實踐
- 使用監控工具（如 Prometheus）檢測組件狀態，設定告警。
- 配置自動切換機制，例如 RabbitMQ 集群的鏡像隊列。

## RabbitMQ 高可用配置

RabbitMQ 作為消息代理，其可用性直接影響整體系統。

### 集群配置

RabbitMQ 集群允許多個節點協同工作，當某個節點故障時，其他節點可以接管：

#### 設定步驟
1. 在多台伺服器上安裝 RabbitMQ。
2. 配置集群，使節點共享相同的 Erlang Cookie 和 hostname（參考官方文檔）。
3. 啟用鏡像隊列，確保消息在多個節點上備份：

```bash
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

#### 最佳實踐
- 確保集群節點數量為奇數（如 3 或 5），避免腦裂問題。
- 配置客戶端自動重連，避免因單節點故障導致連接中斷。

### 負載均衡

使用負載均衡器（如 HAProxy 或 Nginx）分發客戶端連接到 RabbitMQ 集群：

#### 配置示例（HAProxy）
```
frontend rabbitmq_frontend
    bind *:5672
    default_backend rabbitmq_backend

backend rabbitmq_backend
    balance roundrobin
    server rabbit1 192.168.1.101:5672 check
    server rabbit2 192.168.1.102:5672 check
    server rabbit3 192.168.1.103:5672 check
```

#### 最佳實踐
- 設定健康檢查，確保只將流量導向正常節點。
- 監控負載均衡器性能，避免其成為瓶頸。

## Celery Worker 高可用配置

Celery Worker 負責任務執行，確保其高可用性可以避免任務處理中斷。

### 多 Worker 部署

在多台伺服器上運行 Worker，確保即使部分 Worker 故障，系統仍能處理任務：

```bash
celery -A your_module worker --concurrency=4 -n worker1@host1
celery -A your_module worker --concurrency=4 -n worker2@host2
```

#### 最佳實踐
- 將 Worker 分布在不同可用區或數據中心，降低區域性故障風險。
- 使用監控工具檢查 Worker 狀態，自動重啟故障 Worker。

### 任務重試與錯誤處理

設定任務重試機制，確保任務在 Worker 故障時仍能完成：

```python
@task(retry_backoff=True, retry_jitter=True, max_retries=3)
def unreliable_task():
    # 模擬不可靠任務
    pass
```

#### 最佳實踐
- 為關鍵任務設定重試策略，避免因臨時故障失敗。
- 記錄重試日誌，方便排查問題。

## 結果後端高可用配置

結果後端儲存任務結果，其可用性影響任務狀態查詢。

### 使用高可用後端

選擇支持高可用的結果後端，如 Redis 集群或資料庫集群：

#### Redis 集群設定
1. 部署 Redis 集群，包含主節點和從節點。
2. 配置 Celery 使用 Redis 集群：

```python
app.conf.result_backend = 'redis://:password@host1:6379,host2:6379/0'
```

#### 最佳實踐
- 確保結果後端支持自動故障切換。
- 定期備份結果數據，避免數據丟失。

### 結果過期與清理

設定結果過期時間，避免後端資源耗盡：

```python
app.conf.result_expires = 3600  # 結果1小時後過期
```

#### 最佳實踐
- 自動化清理過期結果，保持後端高效運行。
- 監控後端資源使用情況，及時擴展容量。

## 故障恢復與備份

高可用系統需要完善的故障恢復和備份策略，確保在災難發生時能快速恢復。

### RabbitMQ 數據備份

定期備份 RabbitMQ 的數據和配置，確保在節點故障時能恢復：

#### 備份步驟
1. 備份 RabbitMQ 的數據目錄（通常位於 `/var/lib/rabbitmq`）。
2. 備份配置文件（通常位於 `/etc/rabbitmq`）。

#### 最佳實踐
- 將備份數據儲存在異地，避免單一數據中心故障影響恢復。
- 定期測試備份恢復流程，確保數據完整性。

### Celery 任務恢復

對於關鍵任務，確保任務可以在 Worker 故障後重新執行：

```python
app.conf.task_track_started = True  # 追蹤任務開始狀態
app.conf.task_ignore_result = False  # 保留任務結果
```

#### 最佳實踐
- 使用監控工具檢測未完成任務，自動重新排隊。
- 為長時間任務設定檢查點，允許從中斷點恢復。

### 自動化故障恢復

使用自動化工具（如 Kubernetes）管理組件，實現故障自動恢復：

#### Kubernetes 配置示例
1. 為 Celery Worker 和 RabbitMQ 設定 Pod 重啟策略。
2. 配置健康檢查，確保僅運行健康的實例。

#### 最佳實踐
- 設定合理的重啟延遲，避免頻繁重啟導致系統不穩定。
- 記錄故障恢復日誌，分析問題根因。

## 安全性與高可用性

高可用系統需要考慮安全性，避免因安全問題導致服務中斷。

### 訪問控制

限制對 RabbitMQ 和 Celery 組件的訪問，防止未授權操作：

#### RabbitMQ 訪問控制
- 創建專用用戶，禁用默認 `guest` 帳戶：

```bash
rabbitmqctl delete_user guest
rabbitmqctl add_user myuser mypassword
rabbitmqctl set_permissions -p / myuser ".*" ".*" ".*"
```

#### 最佳實踐
- 使用強密碼和角色分離，限制用戶權限。
- 定期審核訪問日誌，發現異常行為。

### 數據加密

確保消息和連接的安全，使用 SSL/TLS 加密通信：

#### RabbitMQ SSL 配置
1. 生成 SSL 證書並配置 RabbitMQ（參考官方文檔）。
2. 配置 Celery 和 Kombu 使用 SSL 連接：

```python
app.conf.broker_url = 'amqp://user:pass@host:5672//?ssl=true'
```

#### 最佳實踐
- 定期更新證書，避免使用過期證書。
- 確保所有組件支持加密連接，避免明文傳輸。

## 總結與注意事項

- **冗餘與分布**：通過多節點部署和地理分布實現高可用性。
- **自動化恢復**：使用工具和策略實現故障自動檢測和恢復。
- **備份與測試**：定期備份數據並測試恢復流程，確保災難時能快速響應。
- **安全與穩定**：在追求高可用性的同時，確保系統安全，避免新漏洞。

通過以上高可用配置，您可以構建一個穩健的分散式任務處理系統，應對各種故障場景，確保服務連續性。

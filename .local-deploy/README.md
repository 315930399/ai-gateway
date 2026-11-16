# `.local-deploy` — 本地验证覆盖层

对 `ai-gateway/kubernetes/` 基础栈的唯一改动：**让 `ai-gateway-api` 使用本地构建的镜像**
（`ai-gateway-api:entity-scope`），其余全部沿用基础栈（单实例 Redis、MySQL、Kafka 等）。

```bash
cd ai-gateway
kubectl apply -k .local-deploy
```

本地镜像构建：

```bash
cd ai-gateway-api
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o ai-gateway-api.linux .
docker build -t ai-gateway-api:entity-scope .
```

## 为什么 Redis 保持单实例（不再走 Envoy + Redis Cluster）

曾尝试「3 主 Redis Cluster + Envoy redis_proxy」把集群伪装成单实例端点，实测发现：

- 普通命令（GET/SET/INCR）经 Envoy 正常；
- **`EVAL`/`EVALSHA` 经 Envoy 返回 `upstream failure`**，而网关有多处单键 Lua 脚本：
  - 控制面：`model/quotacache/lock.go`（配额重置调度器的分布式锁）、`model/ibatch/redis_adapter.go`、`model/quotacache/redis.go`；
  - 数据面 BFE：`mod_ai_token_auth/batch_quota.go`、`token.go`、`mod_ai_batch/batch_state.go`、`mod_ai_rate_limit/redis_lr_agent.go`。
  
  表现为日志里每分钟一条 `Failed to acquire quota scheduler lock: … i/o timeout`，数据面配额/限流脚本也会失效。

因此 Redis 侧回到基础栈的单实例；若后续要用 Redis Cluster，需要给 `bfe_util/redis_client`
加 cluster-aware 实现（或换支持 EVAL 的 `redis-cluster-proxy`），而不是用 Envoy。

## 回滚

```bash
kubectl apply -k kubernetes   # 直接用基础栈（镜像换回上游 tag）
```

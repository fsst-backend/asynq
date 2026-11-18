# AGENTS.md — Asynq 分布式任务队列库

## 项目概述

**hibiken/asynq v0.25.1** 的源码副本，一个基于 **Redis** 的 Go 分布式任务队列库。在 violet 等项目中作为依赖被引用。不是独立服务，而是一个 **Go library**。

## 技术栈

| 项目 | 值 |
|------|-----|
| **Module 名** | `github.com/hibiken/asynq` |
| **Go 版本** | `go 1.22` |
| **依赖** | Redis (go-redis/v9) |
| **序列化** | Protobuf |
| **版本** | v0.25.1 |
| **在 futren 中** | `replace asynq => ../asynq` (violet go.mod) |

## 目录结构

```
asynq/
├── asynq.go              # 核心类型 (Task/TaskInfo/RedisConnOpt)
├── client.go             # ★ Client 客户端 (任务入队)
├── server.go             # ★ Server 服务端 (任务处理)
├── scheduler.go          # Scheduler 定时调度器
├── periodic_task_manager.go # PeriodicTaskManager 周期任务管理器
├── servemux.go           # ServeMux 任务路由
├── context.go            # Handler 辅助函数
├── inspector.go          # Inspector 队列/任务检查器
│
├── internal/
│   ├── base/             # 基础类型/常量/Redis Key 定义/Broker 接口
│   ├── rdb/              # ★ Redis 核心操作 (Lua 脚本/任务 CRUD)
│   ├── proto/            # Protobuf 任务消息编码
│   ├── log/              # 日志抽象
│   ├── errors/           # 错误哨兵 (ErrDuplicateTask 等)
│   ├── context/          # Context 辅助 (retry/maxRetry/queueName)
│   └── testbroker/       # ★ 测试用内存 Broker
│
├── tools/                # 独立 CLI 工具 (各自有 go.mod)
│   ├── asynq/            # 命令行工具 (Cobra + TUI 仪表盘)
│   └── metrics_exporter/ # Prometheus 指标导出器
│
├── x/                    # 实验性扩展 (各自有 go.mod)
│   ├── metrics/          # Prometheus metrics
│   └── rate/             # 流量控制/限流 (信号量)
│
└── docs/assets/          # 文档图片
```

## 核心组件

### Client (任务生产者)
```go
client := asynq.NewClient(redisConn)
task := asynq.NewTask("email:send", payload)
client.Enqueue(task, asynq.MaxRetry(3), asynq.Timeout(30*time.Second))
```

### Server (任务消费者)
```go
srv := asynq.NewServer(redisConn, asynq.Config{Concurrency: 10})
mux := asynq.NewServeMux()
mux.HandleFunc("email:send", handleEmailTask)
srv.Run(mux)
```

### Worker 组件
| 组件 | 文件 | 职责 |
|------|------|------|
| **processor** | processor.go | worker 协程池，执行 Handler |
| **forwarder** | forwarder.go | 将 scheduled/retry 任务移到 pending |
| **recoverer** | recoverer.go | 恢复崩溃 worker 的活动任务 |
| **heartbeater** | heartbeat.go | 定期续约 server 心跳到 Redis |
| **healthchecker** | healthcheck.go | Redis 连接健康检查 |
| **subscriber** | subscriber.go | 监听 Redis Pub/Sub 取消信号 |
| **syncer** | syncer.go | 同步队列信息到 Redis |
| **janitor** | janitor.go | 清理过期 completed 任务 |
| **aggregator** | aggregator.go | 将同组任务聚合为一个任务 |

## Redis 连接方式

三种模式，通过 `RedisConnOpt` 接口：
- **`RedisClientOpt`** — 单机 Redis
- **`RedisFailoverClientOpt`** — Sentinel 主从 + 故障转移
- **`RedisClusterClientOpt`** — Cluster 模式 (部分 Lua 脚本不兼容)

也支持 `ParseRedisURI("redis://user:pass@host:port/db")` 从 URI 解析。

## 任务选项

| 选项 | 说明 |
|------|------|
| `MaxRetry(n)` | 最大重试次数 |
| `Timeout(d)` | 执行超时 |
| `Deadline(t)` | 截止时间 |
| `Unique(ttl)` | 去重 (TTL 内仅保留一个) |
| `ProcessAt(t)` | 延时执行 |
| `ProcessIn(d)` | 延时执行 (相对时间) |
| `Retention(d)` | 完成后保留时间 |
| `Queue(name)` | 指定队列 |
| `Group(name)` | 任务分组 (聚合) |

## 队列优先级

```go
Config{
    Queues: map[string]int{
        "critical": 6,
        "default":  3,
        "low":      1,
    },
    StrictPriority: true,  // 严格优先级模式
}
```

## 在 futren 中的使用

violet 项目通过 `go.mod` 的 `replace asynq => ../asynq` 引用本地副本。在 violet 的 `ServiceContext` 中初始化 asynq Client/Server 用于异步任务处理。

## 测试

```bash
go test ./...            # 全部测试 (~100+ 测试文件)
go test -bench=. -v      # 性能基准测试
```

使用 `internal/testbroker` (内存 Broker) 模拟 Redis 进行单元测试，不依赖真实 Redis。

## 构建 (tools)

```bash
# CLI 工具
cd tools && go build ./asynq    # 构建 asynq 命令行工具

# Metrics Exporter
cd tools && go build ./metrics_exporter
```

## 配置方式

**全部通过代码配置**，无配置文件。核心配置对象：
- `Config` (server.go:97) — 并发数/队列优先级/重试策略/日志等
- `SchedulerOpts` — 时区/入队回调/心跳间隔
- `RedisConnOpt` — Redis 连接参数

## 文档

- `README.md` — Quickstart + Features 列表
- `doc.go` — GoDoc 包文档
- `CHANGELOG.md` — 578 行详细 changelog
- `CONTRIBUTING.md` / `CODE_OF_CONDUCT.md`

## 禁止事项

- **禁止修改 `asynq.go` 中的核心 API** — 维护与上游兼容性
- **禁止修改 `internal/` 包** — 不对使用者暴露
- Redis Cluster 模式下部分 Lua 脚本不兼容，需注意任务取消/聚合功能限制
- 在 futren 中不要直接使用 `asynq` 包名，应使用 replace 后的路径

# asynq（fsst-backend 分叉）

本仓库（GitHub `fsst-backend/asynq`）是 [hibiken/asynq](https://github.com/hibiken/asynq) 的分叉：基于 Redis 的 Go 分布式任务队列。module 路径保持上游的 `github.com/hibiken/asynq`，由 violet 通过 `replace` 替换为本地检出使用。

## 分叉基线

| 项 | 值 | 依据 |
|----|----|------|
| 上游版本 | **v0.25.1**（上游 2024-12-11 发布） | `internal/base/base.go` 的 `Version = "0.25.1"`、`CHANGELOG.md` 最新条目；与上游 tag `v0.25.1` 逐文件比对，除下述差异外完全一致 |
| 分支 | `master`（唯一分支） | — |
| 历史 | 只有一个提交（导入上游代码 + 本地改动合在一起），上游提交历史未保留；该提交打了 tag `v1.0.0`，该 tag 只是本仓库标记，与上游版本号无关 | `git log` |

## 与上游的差异

只改了 `internal/rdb/` 下的 Redis Lua 脚本调用方式，公开 API、Redis 数据结构与键名都没有变化：

- 上游很多脚本把**键前缀**（任务键前缀 `asynq:{<qname>}:t:`、队列键前缀 `asynq:{<qname>}:`、分组键前缀 `asynq:{<qname>}:g:`）放在 `ARGV` 里传入，在脚本内拼接出要访问的键。
- 本分叉把这些前缀改为通过 `KEYS` 传入，`ARGV` 只保留真正的参数（时间戳、ID、数量等）。
- 涉及文件：`internal/rdb/rdb.go`（出队 `dequeue`、`forward`、读取/删除聚合集、清理过期已完成任务、租约过期列表等）和 `internal/rdb/inspect.go`（`Inspector` 使用的统计、列表、运行/归档/删除任务等脚本）。
- `server.go` 只有注释修正（`asynq.IsPanic` → `asynq.IsPanicError`）。

改动目的：提交说明只写了「将任务 key 前缀和组 key 前缀作为 KEYS 参数传递给 Redis 脚本」，没有写明动机。从改动看，是为了让脚本访问的键都能由 `KEYS` 推出，以适配要求 Lua 脚本声明所访问键的 Redis 部署（上游 README 也注明部分脚本可能不兼容 Redis Cluster）——**具体动机待确认**。

### 已知问题

与上游比对时发现以下脚本参数位置不一致（本次未修改代码），使用相关功能前需先修复：

| 位置 | 问题 | 影响 |
|------|------|------|
| `internal/rdb/inspect.go` `getTaskInfoCmd` | 脚本中写成 `KYES[2]`（拼写错误） | `Inspector.GetTaskInfo` 查询 scheduled / retry 状态的任务时 Lua 报错 |
| `internal/rdb/inspect.go` `archiveTaskCmd` | 调用方传 5 个 KEYS（队列前缀在 `KEYS[4]`、分组前缀在 `KEYS[5]`），脚本却用 `KEYS[5]` 当队列前缀、用 `ARGV[6]`（已不存在）当分组前缀 | `Inspector.ArchiveTask` 对各状态任务均无法正确归档 |
| `internal/rdb/inspect.go` `archiveAllPendingCmd` | 调用方传 3 个 KEYS，脚本使用 `KEYS[4]` | `Inspector.ArchiveAllPendingTasks` 在有 pending 任务时 Lua 报错 |
| `internal/rdb/rdb.go` `DeleteAggregationSet` | 脚本使用 `KEYS[3]`，调用方只传 2 个 KEYS、前缀仍放在 `ARGV` | 使用任务分组聚合（`asynq.Group` + `GroupAggregator`）时删除聚合集报错 |
| `internal/rdb/inspect.go` `runTaskCmd` | 两处错误提示仍拼接旧的 `ARGV[2]` / `ARGV[3]` | 仅在数据异常的报错分支触发，报错信息变成 Lua 错误 |

这些都在 `Inspector`（asynqmon 的查看/归档操作走这里）或分组聚合路径上。入队、出队、重试、定时转发等主流程脚本要么未改动，要么改动后经核对与调用处一致。

## 如何被使用

violet 的 `go.mod`：

```
require github.com/hibiken/asynq v0.25.1

replace github.com/hibiken/asynq => ../asynq
```

- 本仓库必须与 violet **同级检出**，目录名为 `asynq`。violet 各分支（包括生产分支）的 `Jenkinsfile` 在「拉取本地 replace 依赖」阶段都固定把本仓库 `master` 检出到 `../asynq` 再编译，所以**推到 `master` 的改动会直接进入 violet 的下一次构建，包括生产构建**；改动应先在功能分支上验证，推 `master` 需负责人明确授权。
- `replace` 对整个构建生效：violet 引用的 `github.com/hibiken/asynqmon`（Web 监控界面）也会使用本分叉。
- violet 中的用法：各服务用 `asynq.NewClientFromRedisClient` 入队；`stask`、`gtask` 用 `asynq.NewServerFromRedisClient` + `asynq.NewServeMux` 消费任务，用 `asynq.NewSchedulerFromRedisClient` 注册定时任务，并挂载 asynqmon。传入的是 violet 自己构造的 `redis.UniversalClient`（单机 / 集群 / 哨兵）。
- violet 通过 `replace github.com/redis/go-redis/v9 => ./third_party/go-redis` 打了 go-redis 补丁，该替换同样作用于本库依赖的 go-redis。

## 开发与测试

| 命令 | 说明 |
|------|------|
| `go build ./...` | 编译主模块 |
| `go test ./...` | 单元/集成测试，需要可用的 Redis，默认 `localhost:6379`（主包用 db 14，`internal/rdb` 用 db 15，测试会清空所用 db）；可用 `-redis_addr`、`-redis_db`、`-redis_cluster`、`-redis_cluster_addrs` 覆盖 |
| `make proto` | 由 `internal/proto/asynq.proto` 重新生成任务消息的 protobuf 代码 |
| `make lint` | `golangci-lint run` |

仓库内另有两个独立 module，均原样来自上游：

- `x/`（`github.com/hibiken/asynq/x`）：Prometheus 指标采集（`x/metrics`）与基于 Redis 的信号量限流（`x/rate`）。
- `tools/`（`github.com/hibiken/asynq/tools`）：命令行工具 `tools/asynq`（见 [tools/asynq/README.md](tools/asynq/README.md)）和 `tools/metrics_exporter`。

注意：`x/` 与 `tools/` 的 `go.mod` 依赖的是上游发布的 `github.com/hibiken/asynq v0.25.0`，**没有** `replace` 到本仓库根，因此在这里编译出的 CLI / 指标导出器使用的是上游脚本，不包含本分叉的改动。

## 升级上游时注意

1. 先读上游 `CHANGELOG.md` 中 v0.25.1 之后的变更，尤其是 Redis 数据结构或脚本参数的改动——集群中新旧版本 worker 会同时访问同一个 Redis。
2. 以新的上游 tag 为基础，把本分叉的改动重新应用一遍：逐个检查 `internal/rdb/` 中所有 `redis.NewScript`，凡是在脚本里用 `ARGV[n] .. xxx` 拼出键名的，改为把前缀放进 `KEYS`，并同步调整调用处的 `keys` / `argv` 切片和脚本头部注释。上游新增的脚本也要按同样规则处理。
3. 改完后核对每个脚本里 `KEYS[n]` / `ARGV[n]` 的下标与调用处切片的长度、顺序一致（上节「已知问题」都属于这一类错误），并在真实 Redis（含集群模式 `-redis_cluster`）上跑完整 `go test ./...`。
4. 更新本文件的「分叉基线」和「与上游的差异」。
5. 在同级检出的 violet 中 `go build ./...`，并验证 `stask` / `gtask` 的入队与消费。

## 上游用法速览

完整文档见上游 [README](https://github.com/hibiken/asynq#readme)、[Wiki](https://github.com/hibiken/asynq/wiki) 和 [godoc](https://pkg.go.dev/github.com/hibiken/asynq)。下面是最常用的 API（在 violet 中改用 `*FromRedisClient` 构造函数传入已有的 redis 客户端）。

定义任务与处理函数：

```go
const TypeEmailDelivery = "email:deliver"

type EmailDeliveryPayload struct {
    UserID     int
    TemplateID string
}

func NewEmailDeliveryTask(userID int, tmplID string) (*asynq.Task, error) {
    payload, err := json.Marshal(EmailDeliveryPayload{UserID: userID, TemplateID: tmplID})
    if err != nil {
        return nil, err
    }
    // 任务级选项可在创建时给出，入队时可覆盖
    return asynq.NewTask(TypeEmailDelivery, payload, asynq.MaxRetry(5), asynq.Timeout(20*time.Minute)), nil
}

func HandleEmailDeliveryTask(ctx context.Context, t *asynq.Task) error {
    var p EmailDeliveryPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        // 包装 asynq.SkipRetry：不再重试
        return fmt.Errorf("json.Unmarshal failed: %v: %w", err, asynq.SkipRetry)
    }
    // ...
    return nil
}
```

入队（立即执行、延迟执行、指定队列等）：

```go
client := asynq.NewClient(asynq.RedisClientOpt{Addr: "127.0.0.1:6379"})
// 或复用已有客户端：client := asynq.NewClientFromRedisClient(rdb)
defer client.Close()

task, _ := NewEmailDeliveryTask(42, "some:template:id")
info, err := client.Enqueue(task)                               // 立即执行
info, err = client.Enqueue(task, asynq.ProcessIn(24*time.Hour)) // 延迟执行
info, err = client.Enqueue(task, asynq.Queue("critical"), asynq.MaxRetry(10), asynq.Unique(time.Hour))
```

消费：

```go
srv := asynq.NewServer(
    asynq.RedisClientOpt{Addr: "127.0.0.1:6379"},
    asynq.Config{
        Concurrency: 10,
        Queues: map[string]int{"critical": 6, "default": 3, "low": 1}, // 加权优先级
    },
)
mux := asynq.NewServeMux()
mux.HandleFunc(TypeEmailDelivery, HandleEmailDeliveryTask)
if err := srv.Run(mux); err != nil {
    log.Fatal(err)
}
```

定时任务：

```go
scheduler := asynq.NewScheduler(asynq.RedisClientOpt{Addr: "127.0.0.1:6379"}, nil)
entryID, err := scheduler.Register("@every 30s", asynq.NewTask("cleanup", nil))
// ...
if err := scheduler.Run(); err != nil {
    log.Fatal(err)
}
```

监控与运维：Web 界面 [asynqmon](https://github.com/hibiken/asynqmon)；命令行见 [tools/asynq/README.md](tools/asynq/README.md)；程序内可用 `asynq.NewInspector` 查询和操作队列（注意上文「已知问题」）。

## 许可证

MIT，见 [LICENSE](LICENSE)。版权归上游作者 Ken Hibino 及贡献者所有。

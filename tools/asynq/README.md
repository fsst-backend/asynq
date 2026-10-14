# asynq CLI

命令行工具，用于查看和操作 asynq 管理的队列与任务。代码原样来自上游 [hibiken/asynq](https://github.com/hibiken/asynq/tree/master/tools/asynq)。

> 与本分叉的关系：`tools/` 是独立 module（`github.com/hibiken/asynq/tools`），其 `go.mod` 依赖上游发布的 `github.com/hibiken/asynq v0.25.0`，没有 `replace` 到本仓库根。所以无论在本仓库编译还是 `go install` 上游版本，CLI 使用的都是**上游**的 Redis 脚本，不包含本分叉对 `internal/rdb` 的改动。键名与数据结构两者一致，CLI 可以读写同一个 Redis；但如果所连的 Redis 部署拒绝上游脚本的访问方式（本分叉要解决的问题，见根 [README](../../README.md)），CLI 的部分命令会失败。

## 安装

```sh
# 在本仓库内编译
cd tools && go build -o asynq ./asynq

# 或安装上游版本
go install github.com/hibiken/asynq/tools/asynq@latest
```

## 命令

用 `asynq help <command> <subcommand>` 查看详细用法。

| 命令 | 子命令 |
|------|--------|
| `asynq dash` | 终端仪表盘，`--refresh` 设置刷新间隔（默认 8s） |
| `asynq stats` | 汇总统计 |
| `asynq queue` | `list`（`ls`）、`inspect`、`history`、`pause`、`resume`（`unpause`）、`remove`（`rm`、`delete`） |
| `asynq task` | `list`（`ls`）、`inspect`、`cancel`、`archive`、`delete`（`rm`、`remove`）、`run`、`enqueue`、`archiveall`、`deleteall`、`runall` |
| `asynq group` | `list`（`ls`） |
| `asynq cron` | `list`（`ls`）、`history` |
| `asynq server` | `list`（`ls`） |
| `asynq version` | 版本信息 |

## 连接参数（全局 flag）

| flag | 默认值 | 说明 |
|------|--------|------|
| `--config` | `$HOME/.asynq.(yaml\|yml\|json)` | 配置文件，用来设置下列 flag 的默认值 |
| `-u, --uri` | `127.0.0.1:6379` | Redis 地址 |
| `-n, --db` | `0` | Redis db 编号 |
| `-p, --password` | 空 | Redis 密码 |
| `--cluster` | `false` | 以集群模式连接 |
| `--cluster_addrs` | 本机 7000–7005 端口 | 集群节点地址，逗号分隔 |
| `--tls_server` | 空 | TLS 校验用的服务器名 |
| `--insecure` | `false` | 跳过 TLS 证书校验 |

配置文件示例（键名与 flag 同名）：

```yaml
uri: 127.0.0.1:6379
db: 2
password: <redis 密码>
```

对生产 Redis 执行 `pause`、`remove`、`archive`、`delete`、`*all` 等写操作前需负责人确认。

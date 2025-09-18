# One Hub NODE_TYPE 配置分析报告

## 概述

One Hub 项目支持通过 `NODE_TYPE` 环境变量配置节点类型，实现主从架构部署。本文档基于源码分析，详细说明 master 和 slave 模式的区别。

## 配置方式

```bash
# Master 节点（默认）
NODE_TYPE=master
# 或者不设置，默认为 master

# Slave 节点
NODE_TYPE=slave
```

## 核心实现

### 配置解析
```go
// common/config/config.go
IsMasterNode = viper.GetString("node_type") != "slave"
```

当 `NODE_TYPE` 不等于 "slave" 时，节点被识别为 master 节点。

## Master 和 Slave 功能对比

| 功能模块 | Master 节点 | Slave 节点 | 源码位置 |
|---------|------------|-----------|----------|
| **数据库迁移** | ✅ 执行迁移 | ❌ 跳过迁移 | `model/migrate.go` |
| **定时任务** | ✅ 执行所有 cron | ❌ 完全禁用 | `cron/main.go` |
| **渠道缓存同步** | ❌ 不同步 | ✅ 定期同步 | `main.go` |
| **Telegram Bot** | ✅ 启动管理 | ❌ 不启动 | `common/telegram/common.go` |
| **前端服务** | ✅ 直接提供 | 🔄 可重定向 | `router/main.go` |

## 详细功能分析

### 1. 数据库迁移权限

**Master 节点：**
```go
// model/migrate.go
func migrationBefore(db *gorm.DB) error {
    if !config.IsMasterNode {
        logger.SysLog("从库不执行迁移前操作")
        return nil
    }
    // 执行数据库迁移...
}
```

**Slave 节点：**
- 跳过所有数据库结构变更
- 避免多节点同时修改数据库

### 2. 定时任务管理

**Master 节点执行的定时任务：**
```go
// cron/main.go
func InitCron() {
    if !config.IsMasterNode {
        logger.SysLog("Cron is disabled on slave node")
        return
    }
    
    // 每日统计任务
    // 每月账单生成
    // 10分钟统计更新
    // 自动价格更新
}
```

**Slave 节点：**
- 完全禁用定时任务
- 避免重复执行管理任务

### 3. 数据同步机制

**Master 节点：**
```go
// main.go
func SyncChannelCache(frequency int) {
    if config.IsMasterNode {
        logger.SysLog("master node does't synchronize the channel")
        return
    }
    // Master 不执行同步
}
```

**Slave 节点：**
```go
for {
    time.Sleep(time.Duration(frequency) * time.Second)
    logger.SysLog("syncing channels from database")
    model.ChannelGroup.Load()
    model.PricingInstance.Init()
    model.ModelOwnedBysInstance.Load()
}
```

### 4. Telegram Bot 服务

**Master 节点：**
```go
// common/telegram/common.go
func InitTelegramBot() {
    if !config.IsMasterNode {
        return
    }
    // 启动 Telegram Bot
}
```

**Slave 节点：**
- 不启动 Telegram Bot 服务
- 避免多个 Bot 实例冲突

### 5. 前端路由处理

**Master 节点：**
```go
// router/main.go
if config.IsMasterNode && frontendBaseUrl != "" {
    frontendBaseUrl = ""
    logger.SysLog("FRONTEND_BASE_URL is ignored on master node")
}
```

**Slave 节点：**
- 可以使用 `FRONTEND_BASE_URL` 重定向
- 支持前端服务分离部署

## 部署架构建议

### 单节点部署
```yaml
# docker-compose.yml
services:
  one-hub:
    environment:
      - NODE_TYPE=master  # 或不设置
```

### 主从架构部署
```yaml
# Master 节点
services:
  one-hub-master:
    environment:
      - NODE_TYPE=master
    # 负责数据管理和定时任务

  # Slave 节点
  one-hub-slave-1:
    environment:
      - NODE_TYPE=slave
    # 负责 API 请求处理

  one-hub-slave-2:
    environment:
      - NODE_TYPE=slave
    # 负责 API 请求处理
```

## 使用场景

### Master 节点适用于：
- 数据库管理和维护
- 定时任务执行
- Telegram Bot 服务
- 单节点部署

### Slave 节点适用于：
- API 请求处理
- 负载均衡扩展
- 只读服务节点
- 水平扩展场景

## 注意事项

1. **数据库连接**：所有节点需要连接同一个数据库
2. **配置同步**：Slave 节点会定期从数据库同步配置
3. **唯一性**：建议只部署一个 Master 节点
4. **网络访问**：确保 Slave 节点能访问数据库

## 总结

One Hub 的 NODE_TYPE 配置实现了清晰的主从架构：
- **Master 节点**：负责数据管理、定时任务和系统维护
- **Slave 节点**：负责请求处理和服务扩展

这种设计有效避免了多节点部署时的资源冲突，支持系统的水平扩展和高可用部署。

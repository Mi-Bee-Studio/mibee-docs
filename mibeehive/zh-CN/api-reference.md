# MiBeeHive API 参考文档

[English](../en-US/api-reference.md)

## 认证端点

### POST /api/v1/auth/login
**描述**：用户认证和 JWT 令牌生成
**认证**：无
**请求体**：
```json
{
  "username": "admin",
  "password": "password"
}
```
**响应**：
```json
{
  "token": "jwt-token-here",
  "expires_in": 3600
}
```

### GET /api/v1/auth/password-status
**描述**：检查是否需要更改密码
**认证**：需要 JWT
**响应**：
```json
{
  "success": true,
  "data": {
    "must_change": false
  }
}
```

## 文件管理端点

### GET /api/v1/files/{id}/download
**描述**：下载特定文件
**认证**：无
**参数**：
- `id`（路径）：文件 ID
**响应**：文件下载流

### GET /api/v1/files/search
**描述**：搜索文件
**认证**：无
**查询参数**：
- `query`（字符串）：搜索查询
- `type`（字符串）：文件类型过滤（可选）
- `limit`（整数）：结果限制（可选，默认 50）
**响应**：
```json
{
  "data": [
    {
      "id": 1,
      "name": "example.zip",
      "size": 1024,
      "type": "binary",
      "created_at": "2023-01-01T00:00:00Z"
    }
  ]
}
```

### GET /api/v1/files/queue
**描述**：获取下载队列状态
**认证**：无
**响应**：
```json
{
  "data": {
    "pending": 5,
    "active": 2,
    "completed": 100,
    "failed": 3
  }
}
```

### GET /api/v1/files/queue/stats
**描述**：获取下载队列统计信息
**认证**：无
**响应**：
```json
{
  "data": {
    "total_downloaded": 1000,
    "total_size": "10GB",
    "average_speed": "2.5MB/s",
    "success_rate": 95.5
  }
}
```

## 项目管理端点

### GET /api/v1/projects
**描述**：列出所有项目
**认证**：无
**响应**：
```json
{
  "data": [
    {
      "id": 1,
      "name": "GitHub Releases",
      "enabled": true,
      "created_at": "2023-01-01T00:00:00Z",
      "config": {...}
    }
  ]
}
```

### POST /api/v1/projects
**描述**：创建新项目
**认证**：无
**请求体**：
```json
{
  "name": "New Project",
  "enabled": true,
  "config": {...}
}
```

### GET /api/v1/projects/{id}
**描述**：获取项目详情
**认证**：无
**参数**：
- `id`（路径）：项目 ID

### PUT /api/v1/projects/{id}
**描述**：更新项目
**认证**：无
**参数**：
- `id`（路径）：项目 ID
**请求体**：与创建相同

### DELETE /api/v1/projects/{id}
**描述**：删除项目（软删除）
**认证**：无
**参数**：
- `id`（路径）：项目 ID

### GET /api/v1/projects/{id}/files
**描述**：列出项目文件
**认证**：无
**参数**：
- `id`（路径）：项目 ID

## 爬取管理端点

### GET /api/v1/crawl/status
**描述**：获取爬取状态
**认证**：无
**响应**：
```json
{
  "data": {
    "projects": [
      {
        "name": "github",
        "status": "running",
        "last_run": "2023-01-01T00:00:00Z",
        "next_run": "2023-01-02T00:00:00Z"
      }
    ]
  }
}
```

### POST /api/v1/crawl/trigger
**描述**：手动触发爬取
**认证**：无
**请求体**：
```json
{
  "project": "github",
  "force": false
}
```

### GET /api/v1/crawl/logs
**描述**：获取爬取日志
**认证**：无
**查询参数**：
- `project`（字符串）：项目过滤（可选）
- `limit`（整数）：日志限制（可选）
**响应**：
```json
{
  "data": [
    {
      "timestamp": "2023-01-01T00:00:00Z",
      "level": "info",
      "message": "Crawl started",
      "project": "github"
    }
  ]
}
```

## 系统信息端点

### GET /api/v1/system/info
**描述**：获取系统信息
**认证**：无
**响应**：
```json
{
  "data": {
    "version": "1.0.0",
    "uptime": "24h",
    "memory_usage": "128MB",
    "disk_usage": "45%",
    "running_since": "2023-01-01T00:00:00Z"
  }
}
```

### GET /api/v1/system/stats
**描述**：获取当前系统统计信息（CPU、内存、网络）
**认证**：无
**响应**：
```json
{
  "data": {
    "cpu_usage_percent": 23.5,
    "memory_usage_percent": 45.2,
    "memory_total_bytes": 491122688,
    "memory_used_bytes": 222000000,
    "network": {...}
  }
}
```

### GET /api/v1/system/stats/history
**描述**：获取系统统计历史
**认证**：无
**查询参数**：
- `hours`（整数）：历史小时数（可选，默认 24）
**响应**：
```json
{
  "data": [
    {
      "timestamp": "2023-01-01T00:00:00Z",
      "cpu_usage_percent": 23.5,
      "memory_usage_percent": 45.2
    }
  ]
}
```

## 操作系统安装端点

### GET /api/v1/os-install/configs
**描述**：列出操作系统安装配置
**认证**：无
**响应**：
```json
{
  "data": [
    {
      "id": 1,
      "name": "Ubuntu 22.04",
      "enabled": true,
      "format": "preseed",
      "created_at": "2023-01-01T00:00:00Z"
    }
  ]
}
```

## 仪表板端点（需要 JWT）

### GET /api/v1/admin/dashboard/summary
**描述**：获取聚合仪表板摘要，包含所有模块的统计信息
**认证**：需要 JWT
**响应**：
```json
{
  "success": true,
  "data": {
    "system": {
      "version": "1.0.0",
      "uptime": "5d 3h 22m",
      "cpu_usage": 23.5,
      "mem_usage": 45.2,
      "mem_total": 491122688,
      "mem_used": 222000000,
      "disk_total": 61236858880,
      "disk_used": 28456726528,
      "disk_usage_percent": 46.5,
      "containers_enabled": true
    },
    "files": {
      "project_count": 6,
      "total_files": 142,
      "queue_pending": 5,
      "queue_downloading": 1,
      "queue_complete": 130,
      "queue_error": 2
    },
    "deploy": {
      "config_count": 8,
      "iso_count": 12,
      "iso_pending": 3,
      "iso_downloaded": 9
    },
    "share": {
      "file_count": 24,
      "total_bytes": 5368709120,
      "total_size": "5.0 GB"
    },
    "activity": [
      {
        "id": "crawl-42",
        "type": "crawl_success",
        "title": "HashiCorp Terraform",
        "subtitle": "Found 3 versions, downloaded 5 files",
        "timestamp": "2026-05-17T10:30:00Z"
      }
    ]
  }
}
```

## 管理面板端点（需要 JWT）

### 项目管理
- **GET** `/api/v1/admin/projects` - 列出项目
- **POST** `/api/v1/admin/projects` - 创建项目
- **PUT** `/api/v1/admin/projects/{id}` - 更新项目
- **DELETE** `/api/v1/admin/projects/{id}` - 删除项目
- **PUT** `/api/v1/admin/projects/{id}/toggle` - 启用/禁用项目

### 爬取管理
- **GET** `/api/v1/admin/crawl/status` - 获取管理爬取状态
- **POST** `/api/v1/admin/crawl/trigger/{name}` - 触发特定项目
- **POST** `/api/v1/admin/crawl/trigger-all` - 触发所有项目
- **PUT** `/api/v1/admin/crawl/pause/{name}` - 暂停项目
- **PUT** `/api/v1/admin/crawl/resume/{name}` - 恢复项目

### 令牌管理
- **GET** `/api/v1/admin/credentials` - 列出 API 令牌
- **POST** `/api/v1/admin/credentials` - 创建/更新令牌

### 安全管理
- **PUT** `/api/v1/admin/password` - 更改管理员密码

### 监控配置
- **GET** `/api/v1/admin/config/monitor` - 获取磁盘警告/关键阈值
- **PUT** `/api/v1/admin/config/monitor` - 更新磁盘阈值
**PUT 请求体**：
```json
{
  "disk_warning_percent": 80,
  "disk_critical_percent": 95
}
```

### 操作系统安装管理
- **GET** `/api/v1/admin/os-install/configs` - 列出配置
- **POST** `/api/v1/admin/os-install/configs` - 创建配置
- **PUT** `/api/v1/admin/os-install/configs/{id}` - 更新配置
- **DELETE** `/api/v1/admin/os-install/configs/{id}` - 删除配置
- **GET** `/api/v1/admin/os-install/configs/{id}` - 获取配置
- **POST** `/api/v1/admin/os-install/configs/preview` - 预览配置

### ISO 管理
- **GET** `/api/v1/admin/os-install/isos` - 列出 ISO
- **POST** `/api/v1/admin/os-install/iso/download` - 下载 ISO
- **DELETE** `/api/v1/admin/os-install/isos/{name}` - 删除 ISO
- **GET** `/api/v1/admin/os-install/catalog` - 列出 ISO 目录条目
- **POST** `/api/v1/admin/os-install/catalog` - 创建目录条目
- **PUT** `/api/v1/admin/os-install/catalog/{id}` - 更新目录条目
- **DELETE** `/api/v1/admin/os-install/catalog/{id}` - 删除目录条目
- **POST** `/api/v1/admin/os-install/catalog/{id}/check` - 检查最新版本
- **POST** `/api/v1/admin/os-install/catalog/{id}/download` - 触发目录下载
- **POST** `/api/v1/admin/os-install/catalog/check-all` - 检查所有版本
- **GET** `/api/v1/admin/os-install/catalog/queue` - 获取 ISO 下载队列统计
- **POST** `/api/v1/admin/os-install/catalog/download-all` - 队列所有可用的 ISO
- **GET** `/api/v1/admin/os-install/catalog/progress` - 获取 ISO 下载进度

### 容器管理
- **GET** `/api/v1/admin/containers` - 列出容器
- **POST** `/api/v1/admin/containers` - 创建容器
- **GET** `/api/v1/admin/containers/{id}` - 获取容器详情
- **PUT** `/api/v1/admin/containers/{id}` - 更新容器
- **DELETE** `/api/v1/admin/containers/{id}` - 删除容器
- **POST** `/api/v1/admin/containers/{id}/start` - 启动容器
- **POST** `/api/v1/admin/containers/{id}/stop` - 停止容器
- **POST** `/api/v1/admin/containers/{id}/restart` - 重启容器
- **GET** `/api/v1/admin/containers/{id}/logs` - 获取容器日志
- **GET** `/api/v1/admin/containers/{id}/stats` - 获取容器统计

### 镜像管理
- **GET** `/api/v1/admin/images` - 列出 Docker 镜像
- **POST** `/api/v1/admin/images/pull` - 拉取 Docker 镜像
- **DELETE** `/api/v1/admin/images/{id}` - 删除 Docker 镜像

### 应用模板
- **GET** `/api/v1/admin/templates` - 列出应用模板
- **POST** `/api/v1/admin/templates` - 创建应用模板
- **GET** `/api/v1/admin/templates/{id}` - 获取应用模板
- **DELETE** `/api/v1/admin/templates/{id}` - 删除应用模板

### WebDAV 管理
- **GET** `/api/v1/admin/webdav/status` - 获取 WebDAV 状态
- **GET** `/api/v1/admin/webdav/files` - 列出 WebDAV 文件

### 搜索
- **GET** `/api/v1/admin/search` - 在文件和配置中全文搜索
**查询参数**：
- `q`（字符串）：搜索查询
- `type`（字符串）：过滤类型（files、configs、all）

### 日志
- **GET** `/api/v1/admin/logs` - 获取系统日志
**查询参数**：
- `level`（字符串）：日志级别过滤（可选）
- `limit`（整数）：结果限制（可选）
- `source`（字符串）：源过滤（可选）

### 任务
- **GET** `/api/v1/admin/tasks` - 列出后台任务

### 备份
- **GET** `/api/v1/admin/backups` - 列出可用备份
- **POST** `/api/v1/admin/backups/restore` - 从备份档案恢复

## ISO 公开端点

### GET /api/v1/isos
**描述**：列出可用的 ISO 文件（公共，无需认证）
**认证**：无
**响应**：
```json
{
  "data": [
    {
      "id": 1,
      "name": "ubuntu-22.04.iso",
      "distro": "Ubuntu",
      "arch": "amd64",
      "size": "4.7GB",
      "current_url": "https://releases.ubuntu.com/22.04.3/ubuntu-22.04.3-live-server-amd64.iso",
      "download_status": "pending"
    }
  ]
}
```

### GET /api/v1/isos/{name}/download
**描述**：下载 ISO 文件（需要 JWT）
**认证**：通过 Authorization 头或 ?token= 查询参数的 JWT
**参数**：
- `name`（路径）：ISO 名称
**响应**：文件下载流
**示例请求头**：
```
Authorization: Bearer <jwt-token>
```
**示例请求 URL**：
```
/api/v1/isos/ubuntu-22.04/download?token=<jwt-token>
```

## 公共 PXE 端点（无需认证）

### GET /pxe/{format}/{name}
**描述**：提供 PXE 配置文件
**认证**：无
**参数**：
- `format`（路径）：配置格式（preseed、kickstart、autoinstall）
- `name`（路径）：配置名称
**响应**：配置文件内容

## 健康与指标端点

### GET /health
**描述**：健康检查端点
**认证**：无
**响应**：`OK`

### GET /metrics
**描述**：Prometheus 指标端点
**认证**：无
**响应**：Prometheus 格式的指标

## 响应格式

所有 API 端点使用一致的响应格式：

### 成功响应
```json
{
  "success": true,
  "data": {...}
}
```

### 错误响应
```json
{
  "success": false,
  "message": "错误描述"
}
```

### 常见错误代码
- `400 Bad Request`： malformed 请求
- `401 Unauthorized`：无效或缺失 JWT 令牌
- `404 Not Found`：资源未找到
- `500 Internal Server Error`：服务器错误

## 认证

### JWT 令牌
- 管理端点需要在 `Authorization` 头中提供有效的 JWT 令牌
- 令牌格式：`Authorization: Bearer <token>`
- 令牌过期：1 小时（3600 秒）
- 令牌由 `/api/v1/auth/login` 端点提供

### WebDAV 认证
- 需要基础认证
- 匿名用户：只读访问
- 管理员用户：读写访问
- 凭据与 Web 管理面板相同
## 供应端点（公共）

### GET /repo/index
**描述**：可服务工件的 JSON 清单（外部服务器发现工具用）
**认证**：主管通道 `anonymous_read` 时无；`token_read` 时需通道 token（Bearer / Basic 密码栏 / `?token=`）
**响应**：`{"count": N, "items": [{id, project_id, version, filename, size_bytes, checksum, download_url}]}`（30s 缓存）

### GET /repo/files/{id}
**描述**：按文件 ID 下载工件；支持 Range 断点续传（206）、HEAD、sha256 强 ETag 304
**认证**：同上

### GET /apt/{rest...}
**描述**：APT 仓库（`dists/` 元数据 + `pool/` 下载），按需生成 `Packages`/`Release`
**认证**：同上（apt 用法：`deb http://u:<token>@host:9090/apt stable main`）

### GET /simple/{rest...}
**描述**：PyPI Simple（PEP 503）索引 + wheel/sdist 下载，带 sha256 fragment
**认证**：同上（pip 用法：`--index-url http://u:<token>@host:9090/simple/`）

## 通道 token 管理（需要 JWT）

签发/吊销供应面读取凭据。吊销即时生效（缓存同步失效）。

### POST /api/v1/admin/channels/{id}/tokens
**描述**：为通道签发新 token（base58，22 字符 ≈128bit）
**请求体**：`{"name": "edge-fleet"}`（name 缺省 "token"）
**响应**：`{"data": {"id": 1, "channel_id": 1, "name": "edge-fleet", "token": "SVGq9K6t...", "created_at": "..."}}`

### GET /api/v1/admin/channels/{id}/tokens
**描述**：列出通道下全部 token（含使用遥测 `last_used_at`）

### DELETE /api/v1/admin/channels/{id}/tokens/{tokenID}
**描述**：吊销一个 token（通道与 token 双重绑定）

### 通道 auth_mode
- `PUT /api/v1/admin/channels/{id}` 的 `auth_mode` 取值：`anonymous_read`（默认）/ `token_read`；遗留拼写 `public` 归一为 `anonymous_read`，非法值 400

## 蜂后端点（/queen）

`/queen` HTTP 面的认证：hive JWT **或** 静态 token（`AUTH_TOKEN`）任一通过；fleet 面板（`/queen/ui`）自动沿用 hive 登录会话。

### GET /queen/api/v1/health
**认证**：豁免

### GET /queen/api/v1/agents
**描述**：agent 快照列表（在线状态、最近心跳、指标、最近命令回执）

### POST /queen/api/v1/agents/{id}/commands
**描述**：下发命令（QoS 1 经 MQTT）
**请求体**：`{"command_type": "status|reload_config|restart|download_model", "payload": ""}`；`download_model` 支持捷径 `{"model_id": "<id>"}`——服务端解析为 `{url, sha256, file_name, token}`（优先供应面 URL 并嵌入凭据）
**响应**：`202 {"command_id": "...", "topic": "kite/agent/{id}/commands"}`
**护栏**：未知 agent 404；离线 agent 409（命令会丢失）；未知类型 400

### GET /queen/api/v1/events
**描述**：收敛后的事件列表
**查询参数**：`limit` / `node` / `severity` / `since`（unix 时间戳）

### 模型管理
- `GET /queen/api/v1/models` — 分页列表（`page` / `page_size` / `format`）
- `POST /queen/api/v1/models/upload` — multipart `file=@model.gguf`（自动算 sha256 并注册为蜂巢 artifact）
- `GET /queen/api/v1/models/{id}` / `POST`（注册元数据）/ `DELETE`
- `GET /queen/api/v1/models/{id}/download` — 下载；配置守卫后由通道 token 单独保护（agent 单凭据）；带 sha256 强 ETag 与 Range 续传

### 量化任务
- `GET /queen/api/v1/quantize/jobs` — 任务列表
- `POST /queen/api/v1/quantize/jobs/create` — 创建量化任务
- `GET/DELETE /queen/api/v1/quantize/jobs/{id}` — 查询/取消

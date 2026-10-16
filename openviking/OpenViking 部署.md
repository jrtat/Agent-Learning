# OpenViking 部署（+ Windows + Docker）

> 指在windows端，以docker的形式在本地运行openviking的服务端，并以插件形式接入deepseek harness。

## 架构总览

┌─────────────────────────────────────────────────────────┐
│  DSH (web profile)                                      │
│   ├─ 插件 @openviking/dsh-memory-plugin  ← 第3步装       │
│   └─ 读 ~/.openviking/ovcli.conf        ← 第5步写       │
│        (url + user key)                                  │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP (1933)
┌────────────────────▼────────────────────────────────────┐
│  OpenViking 服务端容器 (openviking)                      │
│   ├─ 读 ~/.openviking/ov.conf          ← 第4步写         │
│   │    ├─ server (root_api_key, 0.0.0.0:1933)           │
│   │    ├─ storage (local 向量库)                         │
│   │    ├─ embedding → Cloudflare @cf/baai/bge-m3 (1024)  │
│   │    └─ vlm → 方舟 Agent Plan deepseek-v4.1-flash      │
│   └─ 数据存 ~/.openviking/data/                          │
└─────────────────────────────────────────────────────────┘

两个模型的职责：

| 模型 | 职责 | 缺了会怎样 |
|---|---|---|
| **embedding** | 向量化 → 语义检索 | 存进去了但召不回 |
| **vlm** | 大模型 → 记忆抽取/编译 | 检索能用，但不会自动从对话学习 |

## 前置条件

| 组件 | 用途 |
|---|---|
| WSL2 + Ubuntu | Docker 运行时 |
| Docker Desktop | 跑服务端的容器 |
| Cloudflare | 上有免费 embedding 模型 |
| 任意服务商 | 能提供 VLM 模型 |

> embedding也可在本地跑。

## 第 0 步：准备 3 个密钥

三个密钥互相独立，别混用：

| 密钥 |  用途 |
|---|---|
| embedding 模型 | Cloudflare 控制台（Workers AI 权限） | embedding 向量化 |
| VLM 模型 |  VLM 记忆抽取 |
| root_api_key（自己生成） |  服务端管理员密钥 |

生成 `root_api_key`：

```powershell

-join ((48..57) + (97..122) | Get-Random -Count 40 | ForEach-Object {[char]$_})

```

## 第 1 步：安装 WSL2 + Ubuntu

```powershell
wsl --install -d Ubuntu
```

验证：

```powershell
wsl -l -v
# 应看到 Ubuntu 且 VERSION = 2
```

---

## 第 2 步：安装 Docker Desktop

从 [docker.com](https://www.docker.com/products/docker-desktop/) 下载安装（Windows 11 家庭版自动用 WSL2 后端）。

验证：

```powershell
docker --version
docker run --rm hello-world
```

看到 `Hello from Docker!` 即成功。**别跳过这步**——hello-world 通了才说明整条链是通的。

---

## 第 3 步：安装 DSH 插件

```powershell
dsh plugin --profile web add @openviking/dsh-memory-plugin
```

验证：

```powershell
dsh --profile web --dump-config
# 应看到 openviking-memory-runtime 条目
```

---

## 第 4 步：写服务端配置 `ov.conf`

**文件**：`C:\Users\<你>\.openviking\ov.conf`（新建，**保存为 UTF-8 无 BOM**）

```json
{
  "server": {
    "host": "0.0.0.0",
    "port": 1933,
    "root_api_key": "<第0步生成的随机串>"
  },
  "storage": {
    "workspace": "./data",
    "vectordb": { "name": "context", "backend": "local" },
    "agfs": { "backend": "local" }
  },
  "embedding": {
    "dense": {
      "provider": "openai",
      "api_base": "https://api.cloudflare.com/client/v4/accounts/<AccountID>/ai/v1",
      "api_key": "<Cloudflare Token>",
      "model": "@cf/baai/bge-m3",
      "dimension": 1024,
      "encoding_format": "float"
    }
  },
  "vlm": {
    "provider": "volcengine",
    "api_base": "https://ark.cn-beijing.volces.com/api/plan/v3",
    "api_key": "<方舟 API Key>",
    "model": "deepseek-v4.1-flash",
    "temperature": 0.1,
    "max_retries": 3,
    "max_tokens": 4096,
    "timeout": 120
  }
}
```

关键点：

- `root_api_key` 必须有——容器内绑 `0.0.0.0` 时没有它会拒绝启动
- `embedding.provider` 用 `"openai"`（Cloudflare 无专属 provider，走 OpenAI 兼容通道）；`account_id` 编进 `api_base` 路径里
- `embedding.dimension` 必须 = 1024，与 `@cf/baai/bge-m3` 实际输出一致
- `vlm.provider` 用 `"volcengine"`；**Agent Plan 的 base_url 是 `/api/plan/v3`，不是 `/api/v3`**
- `vlm.model` 用 `deepseek-v4.1-flash`（Agent Plan **不支持豆包模型**，只支持 kimi/deepseek/glm）
- `timeout: 120` 让失败快速暴露，别用默认 600 秒

改完验证 JSON（每次必做）：

```powershell
Get-Content "$env:USERPROFILE\.openviking\ov.conf" -Raw | ConvertFrom-Json | Out-Null; "JSON OK"
```

---

## 第 5 步：启动服务端容器（先于 ovcli.conf）

> 先后依赖：`ovcli.conf` 需要 user key，user key 要服务端起来后才能拿到，所以**先启动容器再写 `ovcli.conf`**。

```powershell
docker run -d --name openviking -p 1933:1933 `
  -v "$env:USERPROFILE\.openviking:/app/.openviking" `
  --restart unless-stopped `
  ghcr.io/volcengine/openviking:latest --without-bot
```

要点：

- `-v` 把宿主机配置目录挂进容器（容器内路径固定 `/app/.openviking`）
- `--without-bot` 关掉不需要的 vikingbot gateway
- PowerShell 续行符是 **`` ` ``**，不是 `\`，后面不能有空格
- 首次拉 1.74GB 镜像，耐心等

验证：

```powershell
docker ps --filter name=openviking        # 应显示 Up (healthy)
docker logs openviking --tail 50          # 看启动日志
curl.exe http://127.0.0.1:1933/health     # 应返回 {"status":"ok","healthy":true}
```

---

## 第 6 步：创建用户、拿 user key

用 root key 调管理接口：

```powershell
$body = @{ user_id = 'jrtat'; role = 'user' } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://127.0.0.1:1933/api/v1/admin/accounts/default/users' `
  -Method POST -Body $body -ContentType 'application/json' `
  -Headers @{ 'X-API-Key' = '你的root_api_key' }
```

返回结果里的 `user_key` 即数据访问密钥。

也可以在浏览器打开 `http://localhost:1933/studio` 用图形界面创建账号和用户。

---

## 第 7 步：写客户端配置 `ovcli.conf`

**文件**：`C:\Users\<你>\.openviking\ovcli.conf`（新建）

```json
{
  "url": "http://localhost:1933",
  "api_key": "<第6步拿到的 user key>"
}
```

关键点：这里填 **user key**，**绝不能填 root_api_key**。root key 只能做管理操作，调数据接口会 403，导致插件写入全部失败。

---

## 第 8 步：重启 DSH 会话

改完 `ovcli.conf` 后重启 DSH 会话，插件才用新凭据连接。

成功标志：

1. 会话开头出现 `<openviking-context>` 注入块
2. 模型拿到 `mcp__openviking__*` 工具
3. `~/.openviking/pending/` 保持空

---

## 第 9 步：验证全链路

```powershell
# 1. 服务端 health
curl.exe http://127.0.0.1:1933/health
```

在 DSH 会话里用 `mcp__openviking__remember` 提交一条消息压真实抽取负载，然后确认三项：

- 任务状态 = **completed**（不是 failed）
- `usage_token_hourly` 出现 VLM token 记录
- `total_memories > 0`

搜一条旧记忆确认召回：

```powershell
$body = @{ query = '你的部署配置' } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://127.0.0.1:1933/api/v1/search/find' `
  -Method POST -Body $body -ContentType 'application/json' `
  -Headers @{ 'X-API-Key' = '你的user_key' }
```

---

## 关键教训（避免重蹈覆辙）

1. **JSON 别漏逗号**——每次改完跑一次 `ConvertFrom-Json` 验证，保存用 UTF-8 无 BOM
2. **`ovcli.conf` 用 user key，不是 root key**（root key 数据接口会 403）
3. **Agent Plan 有专属端点** `/api/plan/v3`，且**不支持豆包模型**（只支持 kimi/deepseek/glm）
4. **embedding 维度必须匹配**——换 embedding 模型前检查现有索引是否需要重建
5. **记忆抽取要求模型支持 tool calling**，选模型时优先确认
6. **验证必须压真实抽取负载**——短提示能通过不代表抽取任务能通过
7. **`ov.conf` 是启动配置**——改完必须 `docker restart openviking`

---

## 两个文件的最终结构

**`ov.conf`**（服务端，4 段）：

```
server（root_api_key + 0.0.0.0:1933）
storage（local 向量库）
embedding.dense（Cloudflare bge-m3，1024 维）
vlm（方舟 Agent Plan deepseek-v4.1-flash）
```

**`ovcli.conf`**（客户端，2 字段）：

```
url（http://localhost:1933）
api_key（user key）
```

---

## 故障排查速查

| 现象 | 排查方向 |
|---|---|
| 容器启动失败 | `docker logs openviking`，多半是 JSON 语法或缺 `root_api_key` |
| `/ready` 显示 embedding 相关字段异常 | 检查 `embedding` 段 provider/api_base/dimension |
| 插件写入失败、pending 积压 | 检查 `ovcli.conf` 是否误填了 root key |
| VLM 任务 600 秒超时 | 检查 `vlm.api_base` 是否用错端点、模型是否该端点支持 |
| 召回为空 | 确认 embedding 正常 + 已触发记忆抽取 |

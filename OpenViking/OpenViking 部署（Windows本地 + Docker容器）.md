# OpenViking 部署

> 指在windows端，以docker的形式在本地运行openviking的服务端，并以插件形式接入deepseek harness。

> data：2026/10/16

## 1. 架构总览

```
┌─────────────────────────────────────────────────────────┐
│  DSH (web profile)                                      │
│   ├─ 插件 @openviking/dsh-memory-plugin  ← 第3步装       │
│   └─ 读 ~/.openviking/ovcli.conf        ← 第5步写        │
│        (url + user key)                                 │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP (1933)
┌────────────────────▼────────────────────────────────────┐
│  OpenViking 服务端容器 (openviking)                      │
│   ├─ 读 ~/.openviking/ov.conf          ← 第4步写         │
│   │    ├─ server (root_api_key, 0.0.0.0:1933)           │
│   │    ├─ storage (local 向量库)                         │
│   │    ├─ embedding → Cloudflare @cf/baai/bge-m3 (1024) │
│   │    └─ vlm → 方舟 Agent Plan deepseek-v4.1-flash      │
│   └─ 数据存 ~/.openviking/data/                          │
└─────────────────────────────────────────────────────────┘
```

两个模型的职责：

| 模型 | 职责 | 缺了会怎样 |
|---|---|---|
| **embedding** | 向量化 → 语义检索 | 存进去了但召不回 |
| **vlm** | 大模型 → 记忆抽取/编译 | 检索能用，但不会自动从对话学习 |

---

## 2. 前置条件

| 组件 | 用途 |
|---|---|
| WSL2 + Ubuntu | Docker 运行时 |
| Docker Desktop | 跑服务端的容器 |
| Cloudflare | 上有免费 embedding 模型 |
| 任意服务商 | 能提供 VLM 模型 |

> embedding也可在本地跑。

---

## 第 0 步：准备 3 个密钥

| 密钥 |  用途 |
|---|---|
| embedding 模型 | Cloudflare 控制台（Workers AI 权限） | embedding 向量化 |
| VLM 模型 |  VLM 记忆抽取 |
| root_api_key（自己生成） |  服务端管理员密钥 |

生成 `root_api_key`：

```powershell

-join ((48..57) + (97..122) | Get-Random -Count 40 | ForEach-Object {[char]$_})

```

> 上述指令的作用：生成一个40位的随机字符串，用来当root_api_key。
> `user key` 是 OpenViking 服务端在「创建用户」时当场签发的一串凭证。
> `root key` 没有任何机构或网站发给你。「你」在 `ov.conf` 里写什么，它就是什么。
> 「你」用`root key`调管理接口 或 打开Web Studio，然后服务端创建一个用户（如 user_id = jrtat），接着服务端签发一个`user_key`并返回给「你」，「你」把它填进`ovcli.conf`中。

---

## 第 1 步：安装 WSL2 + Ubuntu

```powershell
wsl --install -d Ubuntu
```

验证：

```powershell
wsl -l -v
# 应看到 Ubuntu 且 VERSION = 2
```

> 安装Ubuntu需要开（全局）代理。

---

## 第 2 步：安装 Docker Desktop

从 [docker.com](https://www.docker.com/products/docker-desktop/) 下载安装（Windows 11 家庭版自动用 WSL2 后端）。

验证：

```powershell
docker --version
docker run --rm hello-world
```

> 看到 `Hello from Docker!` 即成功。

---

## 第 3 步：安装 DSH 插件

```powershell
(pnpm) dsh plugin --profile web add @openviking/dsh-memory-plugin
```

验证：

```powershell
(pnpm) dsh --profile web --dump-config
# 应看到 openviking-memory-runtime 条目
```

> 如果不能直接使用dsh，可以尝试在前面加pnpm。

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
      "provider": "根据具体情况填写",
      "api_base": "根据具体情况填写",
      "api_key": "根据具体情况填写",
      "model": "根据具体情况填写",
      "dimension": 1024,
      "encoding_format": "float"
    }
  },
  "vlm": {
    "provider": "根据具体情况填写",
    "api_base": "根据具体情况填写",
    "api_key": "根据具体情况填写",
    "model": "根据具体情况填写",
    "temperature": 0.1,
    "max_retries": 3,
    "max_tokens": 4096,
    "timeout": 120
  }
}
```

关键点：

- `root_api_key` 必须有——容器内绑 `0.0.0.0` 时没有它会拒绝启动。

改完验证 JSON（每次必做）：

```powershell
Get-Content "$env:USERPROFILE\.openviking\ov.conf" -Raw | ConvertFrom-Json | Out-Null; "JSON OK"
```

> 这个指令验证json格式是否合法。

---

## 第 5 步：启动服务端容器（先于ovcli.conf）

> 先后依赖：`ovcli.conf` 需要`user key`，`user key`要服务端起来后才能拿到，所以**先启动容器再写 `ovcli.conf`**。

```powershell
docker run -d --name openviking -p 1933:1933 `
  -v "$env:USERPROFILE\.openviking:/app/.openviking" `
  --restart unless-stopped `
  ghcr.io/volcengine/openviking:latest --without-bot
```

> 创建并后台启动一个`OpenViking`服务端容器，把配置目录挂进去、把端口映射出来，并设置成自动重启。

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

用`root key`调管理接口：

```powershell
$body = @{ user_id = '选一个值填进去'; role = 'user' } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://127.0.0.1:1933/api/v1/admin/accounts/default/users' `
  -Method POST -Body $body -ContentType 'application/json' `
  -Headers @{ 'X-API-Key' = '你的root_api_key' }
```

> 用`root key`在`OpenViking`服务端上创建一个新用户，并当场拿到这个用户的`user key`。返回结果里的`user_key`即数据访问密钥。

也可以在浏览器打开`http://localhost:1933/studio`用图形界面创建账号和用户。

---

## 第 7 步：写客户端配置 `ovcli.conf`

**文件**：`C:\Users\<你>\.openviking\ovcli.conf`（新建）

```json
{
  "url": "http://localhost:1933",
  "api_key": "<第6步拿到的 user key>"
}
```

> 关键点：这里填 **`user key`**，**绝不能填 `root_api_key`**。`root key` 只能做管理操作，调数据接口会 403，导致插件写入全部失败。

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

## 附录1：两个文件的最终结构

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

## 附录2：部署好后怎么启动

```powershell

Start-Process "Path\Docker Desktop.exe" // 打开桌面端Docker

docker version // 确认打开成功

docker start openviking // 启动容器，容器名字是前面注册好的

docker ps --filter name=openviking --format '{{.Names}} | {{.Status}}' // 验证服务健康
# 期望：openviking | Up xx seconds (healthy)

curl.exe http://127.0.0.1:1933/health // 验证服务健康
# 期望：{"status":"ok","healthy":true,...}

```

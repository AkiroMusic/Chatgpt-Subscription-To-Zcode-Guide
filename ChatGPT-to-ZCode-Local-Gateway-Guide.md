# ChatGPT Subscription → Local sub2api → ZCode: Reproducible Windows Guide
# ChatGPT 订阅接入 ZCode：Windows 本地部署操作手册

Updated / 更新：2026-11-13  
Audience / 适用读者：希望把自己 ChatGPT 订阅可用的 Codex 模型作为 ZCode 自定义 provider 使用的 Windows 用户。  
Scope / 范围：单人、本机、localhost；Docker Desktop + sub2api + ZCode。无需公网服务器。

> **Read first / 先读结论**：ChatGPT Plus 不等于 OpenAI API 余额。本方案让第三方开源网关保存自己的 Codex OAuth 凭证，再向 ZCode 提供兼容 API。它不是 OpenAI 官方提供的 ZCode 集成，也不能承诺账号零风险。最低第三方接触面的方法仍是使用官方 Codex 客户端；选择本方案前，请自己阅读上游条款和 sub2api 合规提示。不要把凭证交给公共中转站，不要共享账号或绕过限额。

本手册可单独分享。所有账号、密码和 key 都由读者自己创建；不包含作者机器的凭据。先完整阅读一次，再按编号执行。命令块默认在 **Windows PowerShell** 中运行，不是 Git Bash；除 WSL 安装外通常无需管理员权限。

## 1. Architecture and prerequisites / 架构与准备

```text
ZCode 自定义 provider
  → http://127.0.0.1:8080/v1
    → sub2api Docker 容器
      → OpenAI Codex 上游（自己的 ChatGPT OAuth）
    → PostgreSQL + Redis（仅 Docker 内网）
```

准备好：

- Windows 10/11、硬件虚拟化、WSL 2、Docker Desktop 的 Linux containers 后端。
- 当前版 ZCode、Node.js LTS（用于 Codex CLI 和本手册的配置脚本）。
- 自己有权使用的 ChatGPT 账号与 Codex 权益；Plus 的模型列表、额度与可用性以账号当前实际结果为准。
- 本机能访问 GitHub、Docker Hub 和 OpenAI。Windows 浏览器能访问，并不保证 Docker 容器也能访问。
- 至少为 Docker 留出合理的内存和磁盘空间；具体系统要求见 Docker 官方文档。

本地文件约定：

| 路径 | 用途 |
|---|---|
| `$HOME\sub2api-deploy\docker-compose.yml` | 部署定义 |
| `$HOME\sub2api-deploy\.env` | 数据库、管理员等密码，私密文件 |
| `$HOME\.codex\auth.json` | 官方 Codex 登录凭证，按密码保护 |
| `$HOME\.zcode\v2\provider_config.json` | ZCode provider 配置，内含网关 key |
| Docker named volumes | 网关数据、数据库、缓存 |

不要把上述私密文件提交 Git、放进共享目录、发给朋友或粘贴给聊天模型。朋友需要登录自己的账号。

## 2. Install Docker and verify it / 安装并验证 Docker

若未安装 WSL，在管理员 PowerShell 中执行，按提示重启：

```powershell
wsl --install
```

已安装时可更新：

```powershell
wsl --update
wsl --version
```

按 Docker 官方指南安装 Docker Desktop：
https://docs.docker.com/desktop/setup/install/windows-install/

启动 Docker Desktop，使用 WSL 2 backend 和 Linux containers。等待界面显示 Engine running，再执行：

```powershell
docker version
docker compose version
docker info
node --version
npm --version
```

验收：`docker version` 同时显示 Client、Server；`docker info` 正常返回。若提示 `dockerDesktopLinuxEngine` pipe 不存在，先启动 Docker Desktop，无需重装 sub2api。

## 3. Obtain the deployment file / 获取部署文件

首次安装使用全新目录。若已经部署，不要覆盖现有 `.env` 或重复初始化。

以下固定到本次核对过的源码 revision，便于复查。镜像仍使用可变 tag，后文说明版本记录方法。

```powershell
$deployDir = Join-Path $HOME 'sub2api-deploy'
if (Test-Path $deployDir) { throw 'Directory already exists. Inspect it before continuing.' }
New-Item -ItemType Directory -Path $deployDir | Out-Null
Set-Location $deployDir
$revision = '530c5e3a4f9c73ca2b0881191d0ef8e0d82c5484'
Invoke-WebRequest -UseBasicParsing `
  "https://raw.githubusercontent.com/Wei-Shaw/sub2api/$revision/deploy/docker-compose.yml" `
  -OutFile '.\docker-compose.yml'
```

先打开 `docker-compose.yml` 阅读，确认镜像来源和卷定义。不要运行来源不明的安装脚本。

## 4. Create private configuration / 创建私密配置

以下脚本生成独立随机密码，并以 UTF-8 无 BOM 写入 `.env`。不会把密码输出到终端。请填写一个合法的管理员邮箱；它是本地后台登录标识，不需要等同于 OpenAI 账号。

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
if (Test-Path '.env') { throw '.env already exists; do not overwrite it.' }
function New-LocalSecret {
  $rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
  try {
    $bytes = New-Object byte[] 32
    $rng.GetBytes($bytes)
    return ([BitConverter]::ToString($bytes)).Replace('-', '').ToLowerInvariant()
  } finally { $rng.Dispose() }
}
$adminEmail = Read-Host 'Local admin email (valid email format)'
if ($adminEmail -notmatch '^[^\s@]+@[^\s@]+\.[^\s@]+$') { throw 'Use a valid email address.' }
$envText = @"
BIND_HOST=127.0.0.1
SERVER_PORT=8080
POSTGRES_PASSWORD=$(New-LocalSecret)
REDIS_PASSWORD=$(New-LocalSecret)
ADMIN_EMAIL=$adminEmail
ADMIN_PASSWORD=$(New-LocalSecret)
JWT_SECRET=$(New-LocalSecret)
TOTP_ENCRYPTION_KEY=$(New-LocalSecret)
TZ=Asia/Shanghai
RUN_MODE=standard
SECURITY_URL_ALLOWLIST_ALLOW_INSECURE_HTTP=false
SECURITY_URL_ALLOWLIST_ALLOW_PRIVATE_HOSTS=false
"@
[IO.File]::WriteAllText((Join-Path $PWD '.env'), $envText, (New-Object Text.UTF8Encoding($false)))
Remove-Variable envText
notepad .env
```

在本地记下 `ADMIN_EMAIL`、`ADMIN_PASSWORD`，用于登录管理后台。`.env` 本身没有加密，按密码文件保护；建议密码管理器另存副本。

关键说明：

- **必须设 `BIND_HOST=127.0.0.1`**；默认监听所有接口不符合本手册的本机使用范围。
- `POSTGRES_PASSWORD` 首次初始化后，不可仅改 `.env` 就当作完成数据库密码轮换。
- 保留 `JWT_SECRET` 和 `TOTP_ENCRYPTION_KEY`；不要每次启动重新生成。
- Compose 已启用自动 setup；后台初始化行为可能随版本变化。

启动并验证：

```powershell
docker compose --env-file .env -f docker-compose.yml config --quiet
docker compose --env-file .env -f docker-compose.yml pull
docker compose --env-file .env -f docker-compose.yml up -d
docker compose ps
Invoke-RestMethod 'http://127.0.0.1:8080/health'
Start-Process 'http://127.0.0.1:8080'
```

首次拉镜像需要时间。验收：三个服务启动，随后 healthy；app 发布端口应为 `127.0.0.1:8080->8080/tcp`，数据库和 Redis 不向宿主机发布端口。

`config --quiet` 只验证配置；不要把普通 `docker compose config` 输出发到聊天里，其中可能包含展开后的密码。

## 5. Log into official Codex / 登录官方 Codex

按照 OpenAI 官方 Codex 文档安装 CLI，常见 npm 安装方式如下：

```powershell
npm install -g @openai/codex
codex --version
codex login
```

在官方登录页面登录自己的 ChatGPT 账号，选择 ChatGPT 登录方式。先运行一次 `codex` 做简短对话，确认当前订阅确实能使用 Codex。

只检查凭据文件是否存在，不输出其内容：

```powershell
Test-Path (Join-Path $HOME '.codex\auth.json')
```

若使用了自定义 `CODEX_HOME`，以官方客户端实际保存位置为准。不要用浏览器 cookie、ChatGPT sessionToken 或网页抓取替代 Codex OAuth。API-key 登录与 ChatGPT 订阅登录是两种不同计费路径。

## 6. Configure sub2api / 配置网关账号、分组、余额和 key

浏览器打开 http://127.0.0.1:8080 ，使用 `.env` 的管理员邮箱与密码登录。

首次可能要求阅读并确认合规文档。自行阅读后决定是否接受；确认短语、版本以页面当时显示为准，不要从旧教程抄固定短语。

### 6.1 Create an OpenAI group / 创建 OpenAI 分组

后台 Groups / 分组 → 创建：

- 名称示例：`personal-openai`。
- 平台：**OpenAI**。
- 标准 / standard 类型；倍率 **1**（必须大于 0）。
- 初次不要启用复杂模型路由、独占或过滤规则。

### 6.2 Import your Codex session / 导入自己的 Codex 登录

后台 Accounts / 账号 → 创建或导入，寻找 **OpenAI OAuth → Codex session / auth.json** 专用入口。

- 如果页面允许选择文件，选择自己的 `auth.json`。
- 如果只允许文本，在本机文本编辑器打开文件，将内容直接粘贴到本机后台表单。不要粘贴到聊天。
- 平台选择 OpenAI、类型 OAuth，绑定刚创建的 `personal-openai` 分组。
- 给账号起一个不含私密信息的名称。
- 完成后用后台测试功能检查账号状态。

注意：通用“数据导入”要求 sub2api 自己的 accounts/proxies 格式，**不等同于 Codex session 导入**。若找不到专用入口，先核对部署版本，不要随意改写 auth.json。

本次核对版本的专用 API 是 `POST /api/v1/admin/accounts/import/codex-session`，其中 `content` 为原始 JSON 的字符串，另有 `group_ids`、`update_existing` 等字段。普通读者优先使用 UI，避免手工处理管理员 JWT 和凭据。

含 refresh token 的 OAuth 导入可由网关尝试续期，但续期不是永久保证；仅 access token 的导入无法自动续期。不要把“当前过期时间”解释成账号永久可用期限。

### 6.3 Check balance and access / 检查本地额度与权限

标准模式会检查 sub2api 自己的用户余额。管理员初始余额可能为 0；账号导入成功并不代表推理请求必然能通过。

在后台给**实际创建 key 的用户**设置一个适当的本地测试余额，并确认其有权使用 OpenAI 分组。这是本机网关的计量数字，不是向 OpenAI 充值，也不增加 ChatGPT 订阅额度。不要填写夸张到超出数据库字段范围的数值。

### 6.4 Create a gateway API key / 创建网关 key

API Keys / 密钥 → 创建：名称例如 `zcode-local`，绑定 `personal-openai` 分组。

保存生成的 key 到密码管理器。它是 **sub2api 网关 key**，不是 OpenAI Platform key，不是管理员密码，不是 OAuth token。

确认以下三者连通：

```text
网关 key → OpenAI 分组 ← 导入的 OpenAI OAuth 账号
```

## 7. Verify the Responses API first / 先验证网关接口

先不要排查 ZCode。用 PowerShell 在本地输入网关 key；以下代码不会回显 key。

模型名称要从自己的账号可用列表中选择。模型 ID 和可用权限可能随账号、地区及时间变化；先在官方 Codex 或当前网关模型测试页面确认精确名称，再填下面的 `$modelId`。

```powershell
$modelId = Read-Host 'Exact model ID available to your account'
$secureKey = Read-Host 'sub2api gateway API key' -AsSecureString
$ptr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureKey)
try {
  $gatewayKey = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr)
  $headers = @{ Authorization = "Bearer $gatewayKey" }
  $body = @{
    model = $modelId
    input = @(@{ role = 'user'; content = 'Reply exactly OK.' })
    stream = $false
    reasoning = @{ effort = 'medium' }
  } | ConvertTo-Json -Depth 12
  $result = Invoke-RestMethod -Method Post `
    -Uri 'http://127.0.0.1:8080/v1/responses' `
    -Headers $headers -ContentType 'application/json' -Body $body -TimeoutSec 120
  $result | Select-Object status, model, usage
  $result.output | ConvertTo-Json -Depth 8
} finally {
  [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($ptr)
  Remove-Variable gatewayKey, headers, secureKey -ErrorAction SilentlyContinue
}
```

验收：响应 `status` 为 `completed`，output 含预期文本。仅 HTTP 200 不足以判定流式请求完整成功；后续检查应看到 `response.completed`，而非 `response.failed` 或流中断。

如果模型不接受 `medium`，按上游错误提示调整；不要假设所有模型支持相同 effort。

ZCode provider 的 `group` 是 ZCode 本地 UI 分类字段，和 sub2api 后台分组是两个不同概念。示例脚本用 `standard-personal` 作为 ZCode 本地分类名；它不需要与后台分组 `personal-openai` 一致。

优先在 ZCode 自定义 provider UI 中配置：

| 字段 | 值 |
|---|---|
| Provider ID | `gpt-local` |
| 名称 | `GPT (sub2api local gateway)` |
| API 类型 | **OpenAI Responses** / `openai-responses` |
| Base URL | `http://127.0.0.1:8080/v1` |
| API key | 第 6.4 步的网关 key |
| 模型 ID | 第 7 步实测可用的精确 ID |

如果 UI 只显示 Effort On/Off，需要按模型增加 `reasoningLevel` 规则。下面脚本会合并一个 provider，保留其他 provider，并在写入前创建备份。

### 8.1 Safe merge script / 安全合并脚本

1. 保存下面代码为一个新文件，例如桌面的 `configure-zcode-gpt.cjs`，不要覆盖现有脚本。
2. 关闭 ZCode 后，在 PowerShell 运行 `node "$HOME\Desktop\configure-zcode-gpt.cjs"`。桌面若被 OneDrive 重定向，请改用实际保存路径。
3. 输入自己已验证的模型 ID、effort 列表，以及网关 key。key 输入会显示在本地终端，使用私密窗口；不要录屏。脚本不把 key 写进日志或命令行历史，但最终 ZCode 配置会明文保存它。
4. 再打开 ZCode，选择新模型。

```javascript
// configure-zcode-gpt.cjs — Node.js LTS, local execution only
const fs = require('node:fs');
const path = require('node:path');
const readline = require('node:readline/promises');

(async () => {
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  try {
    const modelId = (await rl.question('Verified model ID: ')).trim();
    const levels = (await rl.question('Verified efforts, comma separated (e.g. low,medium,high): '))
      .split(',').map(x => x.trim()).filter(Boolean);
    const key = (await rl.question('Local gateway key (visible locally): ')).trim();
    if (!modelId || !key || !levels.length) throw new Error('Required input is empty');
    if (levels.some(x => !['none','low','medium','high','xhigh','max'].includes(x))) {
      throw new Error('Unexpected effort value');
    }
    const home = process.env.USERPROFILE || process.env.HOME;
    const file = path.join(home, '.zcode', 'v2', 'provider_config.json');
    if (!fs.existsSync(file)) throw new Error('Open ZCode and initialize provider settings first');
    const original = fs.readFileSync(file, 'utf8');
    const doc = JSON.parse(original.replace(/^\uFEFF/, ''));
    if (doc.schemaVersion !== 1 || !doc.config) throw new Error('Different schema; inspect current ZCode docs');
    const c = doc.config;
    if (!Array.isArray(c.providerConfigRules?.providerRules) ||
        !Array.isArray(c.modelConfigRules?.providerModelRules) ||
        !Array.isArray(c.providerOrder)) throw new Error('Unexpected configuration structure');
    if (c.providerConfigRules.providerRules.some(x => x.providerId === 'gpt-local')) {
      throw new Error('gpt-local already exists; edit the existing provider instead of overwriting it');
    }
    c.providerConfigRules.providerRules.push({
      providerId: 'gpt-local', providerName: 'GPT (sub2api local gateway)', enabled: true,
      config: {
        group: 'standard-personal', access: {type:'api-key', apiKey:key},
        api: {type:'openai-responses', baseUrl:'http://127.0.0.1:8080/v1'},
        personalModelIds:[modelId], modelOrder:[modelId]
      }
    });
    c.providerOrder.push('gpt-local');
    c.modelConfigRules.providerModelRules.push({
      providerId:'gpt-local', modelId,
      config: {
        enabled:true,
        optionSpecs: {
          reasoningLevel: {values:levels, map:'{"reasoning":{"effort":reasoningLevel}}'},
          maxOutputTokens: {max:128000, map:'{"max_output_tokens":maxOutputTokens}'}
        }
      }
    });
    const backup = file + '.bak-gpt-local-' + Date.now();
    fs.writeFileSync(backup, original, {flag:'wx'});
    fs.writeFileSync(file, JSON.stringify(doc, null, 2) + '\n');
    console.log('Provider added. Backup: ' + backup);
  } finally { rl.close(); }
})().catch(e => { console.error(e.message); process.exitCode = 1; });
```

**先接一个模型再扩展。**脚本使用 128000 作为输出配置允许上限，只适用于实际支持该值的模型；未知模型应改为其文档允许值。`max` 字段不等于每个请求都会自动发送 128000，也不保证高 effort 永不触及输出上限。

脚本刻意不批量宣称所有模型拥有 1M 上下文、PDF 或 JSON schema 能力。需要时再按 ZCode 当前 schema 为该模型添加经核实的 `properties`。

### 8.2 Capability and effort reference / 能力与挡位参考

本机 2026-11-13 的探测结果，只是当时账号/通道的样本，不是官方永久型号目录：

| 模型 | 当时接受的 effort |
|---|---|
| `gpt-6.1-sol`, `gpt-6`, `gpt-6-astra` | low, medium, high, xhigh, max |
| `gpt-6.1-terra`, `gpt-6-sol`, `gpt-6-luna` | none, low, medium, high, xhigh, max |
| `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.6` | none, low, medium, high, xhigh, max |

`none` 不受支持时，上游会明确返回 unsupported value。不要把 effort 开关的 `disabled` 无条件映射为 `none` 并提供给不支持它的模型。

- 这 10 个模型当时均接受图片输入；简单红蓝图测试中 `gpt-6-luna` 多次识别错误，避免用于关键视觉操作。小样本不能推导整体视觉能力排名。
- 工具调用成功是 computer-use 的前置条件之一，不能等同于真实桌面操作稳定。
- 网关可能配置模型别名映射；名称与实际 upstream model 不一定一一对应。要对比模型能力时检查后台映射和日志。
- PDF、structured output、完整 context window 未在本次最终检查中独立验证。不要仅因为配置里写了 `true` 就宣称支持。

## 9. Acceptance checklist / 首次验收

按顺序完成，失败时停在该层排查：

1. **Docker**：三个容器 healthy，app 只发布到 127.0.0.1。
2. **OAuth 账号**：后台账号可用、绑定 OpenAI group；余额和权限通过。
3. **网关 API**：第 7 步非流式请求 `completed`。
4. **ZCode 文本**：新 session 选 `gpt-local/<模型>`，要求回复固定短句。
5. **Effort**：菜单显示配置的细挡位，切换后无 unsupported value。
6. **Tools**：让模型创建或读取一个专用临时测试文件，确认调用工具并读回结果；勿拿真实业务数据做测试。
7. **图片**：上传自己生成的非敏感测试图，要求描述；确认不是只有请求被接受。
8. **Computer-use（需要时）**：独立测试应用中输入短句并截图核对，避免从敏感桌面开始。
9. **维护**：知道如何启动 Docker、看日志、恢复 provider 备份。

## 10. MCP management and reconnect loops / MCP 管理与重连

ZCode Settings → MCP 可以手动启停服务器，开关状态会写入对应配置。按当前任务只启用需要的 MCP，并在会话中确认工具是否可用；不同版本的工具热更新行为可能不同，最稳妥是新开 session 后验收。

**建议只开当前任务需要的 MCP。**不用时关闭重型服务器；需要时手动打开，再确认当前会话是否已获得工具。不同版本的热更新行为可能不同，最稳妥是新开 session 后验收，不能断言所有旧 session 都不会更新。

User 配置通常位于 `~/.zcode/cli/config.json`，workspace 配置通常位于 `<项目>/.zcode/config.json`。需要按项目隔离时才考虑迁移；迁移必须保留备份，检查同名覆盖和插件重复来源，不能把“已建 workspace”当作自动完成作用域迁移。

**真实故障样本**：全量挂载 311 个 tools 时，terra 对同一请求持续 502；去掉工具集成功，缩短历史但保留全工具仍失败。特定 270 项子集成功、280+ 子集失败，sol 对同一大工具集成功。说明此通道的大工具集兼容性有问题；实验没有分离工具数量、schema 大小和组合效应，不能宣布官方统一“270 个硬上限”。

缓解：关闭不用的 MCP、减少 schema，或换已验证可用模型。不要静默截断 tools 或删历史 function_call/output；这可能损坏调用配对。

`upstream_status:502` 与 `pool=1, excluded=1` 只说明这次请求失败且单账号池无备用，**不能单凭它判断 OpenAI 全局故障**。先对比同模型短请求、减少工具集的请求和其他模型。

## 11. Context, compact and output budgets / 上下文、压缩与输出预算

- 输入上下文限制与输出 token 限制是两回事；reasoning token 也可能消耗输出预算。
- 自定义 provider 的 `contextWindow` 是客户端元数据，不能扩大上游真实窗口。不能把本机配置的 `1050000` 无条件复制给所有型号。
- ZCode 的自动压缩是客户端功能，是否启用、触发阈值、使用的总结模型和重试行为应按当前版本设置/日志验证。本次检查没有逼近真实上限触发自动 compact，因此不保证“到了必然自动压缩”。
- 长任务前可在输入框 `/` 菜单确认有无 compact 命令，并在需要时主动压缩；重要状态保存在项目文档，必要时新会话继续。
- 看到 `The model's response exceeded the output token maximum`：先降低 effort / 缩小单次任务，检查实际发出的 `max_output_tokens` 及型号上限。提高允许上限可缓解部分情况，但不是永久修复保证。
- compact 失败时检查其独立请求的错误；不要推定它一定携带完整 tools，或一定用当前模型。

## 12. Daily startup and shutdown / 日常使用

每天：

1. 打开 Docker Desktop，等待 engine ready；可在其设置启用登录 Windows 时自动启动。
2. 通常无需重新登录 Codex，Compose 的 restart 策略会让服务随 Docker 恢复；必要时执行：

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose up -d
docker compose ps
```

3. 打开 ZCode，选择 `gpt-local` 模型即可。无须每次开着官方 Codex 窗口或管理网页。

临时停止网关且保留数据：

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose stop
# 恢复
docker compose up -d
```

关闭 Docker Desktop 会同时停止网关；正在用 GPT 的会话会失败。

## 13. Troubleshooting / 排障表

| 现象 | 优先检查 | 处理 |
|---|---|---|
| localhost:8080 连不上 | Docker engine、容器、端口占用 | 启动 Docker，再 compose up -d；端口冲突时同步修改宿主端口和 ZCode base URL |
| 登录邮箱被拒绝 | 邮箱格式 | 用合法 email 格式；不要用 `local@localhost` |
| 合规确认失败 | 当前文档和确认短语 | 在页面阅读后按当前提示确认，不硬编码旧短语 |
| 401 | 用错 key、OAuth 到期/撤销 | 区分网关认证与上游认证；必要时 Codex 重新登录，再专用入口更新导入 |
| 无 OpenAI account / group 不匹配 | key、账号是否绑定同一 OpenAI group | 修正绑定；查看账号是否暂停/失效 |
| 余额不足 / 403 | 本地用户余额、分组权限 | 配置本地测试余额，不等同于增加 Plus 额度 |
| model_not_found | 名称/权益/映射/退役 | 从当前账号实测可用列表选，不照搬旧型号 |
| unsupported reasoning.effort | 每型号允许值 | 删除不支持的挡位，按错误提示配置 |
| 一直重连 / 502 | 精确请求、工具集、网络、上游 | 做短请求对照，关不用的 MCP；不要只根据状态码归因 |
| 200 但中途失败 | SSE error、response.failed、EOF | 检查完整结束事件和 usage，不能仅看 HTTP 状态 |
| 输出上限错误 | effort、实际输出 cap、任务长度 | 降 effort、拆任务、核实允许上限与真实发送值 |
| 429 / rate limit | 当前订阅限额 | 等待恢复或换其他已授权 provider；不靠多账号轮换绕限额 |
| 网页能用但 Docker 上游不通 | 容器 DNS/网络/代理 | 区分容器和宿主机网络；容器访问宿主机通常用 host.docker.internal |
| effort 仍只有 On/Off | API 类型、每模型 optionSpecs | 用 Responses 并配置 reasoningLevel，确认选的是同一 provider/model |

本地查看日志：

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose logs --since 10m --tail 150 sub2api
```

日志可能包含请求内容、账号标识或初始化凭据；先本地检查并脱敏，再分享错误摘要。不要把 auth.json、完整 .env、provider_config 或完整抓包发给他人。临时抓包会保存私密会话，分析完应按自己授权的范围清理。

## 14. Backup, upgrade and rollback / 备份、升级与回滚

重要：provider 配置备份也含 key。与原件同等保护。

- 备份 `.env`、Compose 文件和 ZCode provider 配置到私密位置。
- 网关账号等数据在 Docker volumes 中，仅保存 Compose/.env 不是完整数据库备份。重要状态请额外按 PostgreSQL 官方流程备份并验证恢复。
- 记录实际镜像标识，以便复查版本：

```powershell
docker image inspect weishaw/sub2api:latest --format '{{json .RepoDigests}}'
docker compose images
```

本手册固定了 Compose 源码 revision，但 `latest`、`18-alpine`、`8-alpine` 镜像标签会变化；严格复现需记录/固定 digest。不要盲目降级数据库或用旧应用打开新 schema。

升级前阅读 release notes 并完成备份，之后才运行：

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose pull
docker compose up -d
docker compose ps
```

回滚 provider：关闭 ZCode，恢复第 8 步备份，重新打开并验收。只想暂时不用时，在 UI 禁用 `gpt-local` 即可。

停用部署：`docker compose stop` 或 `docker compose down` 均可保留 named volumes。**不要加 `-v`**；`down -v` 会删除持久化数据，只能在明确决定永久销毁且完成备份后执行。

## 15. 复现验收记录 / Reproducibility and acceptance

不同机器、账号、模型与软件版本的结果会不同。不要把作者某次账号的模型目录、MCP 开关状态或请求结果当成你的验收结果。请在自己的环境逐项完成第 9 节，并记录：部署文件 revision、镜像 digest、ZCode 与网关版本、实际可用模型 ID、测试时间、Responses 最终事件及工具调用结果。避免记录或公开任何 OAuth token、网关 key、管理员密码或原始会话内容。

本手册中的模型与 tools 观察是一次有限样本，用于提示可能的兼容性问题；它们不是服务承诺，也不能证明所有账号/区域/订阅都可用。PDF、结构化输出、真实上下文上限、自动 compact 和真实 computer-use 应在自己的环境单独验证。

## 16. References / 资料来源

- sub2api repository: https://github.com/Wei-Shaw/sub2api
- Verified Compose revision: https://github.com/Wei-Shaw/sub2api/blob/530c5e3a4f9c73ca2b0881191d0ef8e0d82c5484/deploy/docker-compose.yml
- Docker Desktop Windows: https://docs.docker.com/desktop/setup/install/windows-install/
- Codex CLI: https://github.com/openai/codex
- OpenAI Codex docs: https://developers.openai.com/codex/
- OpenAI Terms of Use: https://openai.com/policies/terms-of-use/
- ZCode configuration paths and fields: checked against the locally installed version's configuration and bundled zcode-guide; use the current client schema if the version changes.

# ChatGPT Subscription → Local sub2api → ZCode: Reproducible Windows Guide

Updated: 2026-11-13  
Audience: Windows users who want to use Codex models available through their own ChatGPT subscription as a custom provider in ZCode.  
Scope: Single user, local machine, localhost; Docker Desktop + sub2api + ZCode. No public server required.

> **Read first**: ChatGPT Plus is not the same as OpenAI API credit. This approach lets a third-party open-source gateway store your own Codex OAuth credentials and provide a compatible API to ZCode. It is not an official OpenAI-provided ZCode integration, and it cannot guarantee zero account risk. Using the official Codex client remains the approach with the least third-party exposure. Before choosing this approach, read the upstream terms and sub2api compliance notice yourself. Do not give credentials to a public relay, share accounts, or bypass limits.

This guide can be shared on its own. Readers create all accounts, passwords, and keys themselves; no credentials from the author's machine are included. Read the entire guide once before following the numbered steps. Unless otherwise indicated, run command blocks in **Windows PowerShell**, not Git Bash. Administrator privileges are generally unnecessary, except when installing WSL.

## 1. Architecture and prerequisites

```text
ZCode 自定义 provider
  → http://127.0.0.1:8080/v1
    → sub2api Docker 容器
      → OpenAI Codex 上游（自己的 ChatGPT OAuth）
    → PostgreSQL + Redis（仅 Docker 内网）
```

Prepare:

- Windows 10/11, hardware virtualization, WSL 2, and Docker Desktop with the Linux containers backend.
- The current version of ZCode and Node.js LTS (for the Codex CLI and this guide's configuration script).
- A ChatGPT account and Codex entitlement that you are authorized to use. The available models, usage limits, and availability for Plus depend on the account's current results.
- Local access to GitHub, Docker Hub, and OpenAI. Access from a Windows browser does not guarantee that Docker containers can also connect.
- Reasonable memory and disk space for Docker; see Docker's official documentation for specific system requirements.

Local file conventions:

| Path | Purpose |
|---|---|
| `$HOME\sub2api-deploy\docker-compose.yml` | Deployment definition |
| `$HOME\sub2api-deploy\.env` | Database, administrator, and other passwords; private file |
| `$HOME\.codex\auth.json` | Official Codex login credentials; protect like a password |
| `$HOME\.zcode\v2\provider_config.json` | ZCode provider configuration, including the gateway key |
| Docker named volumes | Gateway data, database, and cache |

Do not commit the private files above to Git, put them in shared directories, send them to friends, or paste them into chat models. Friends need to log in with their own accounts.

## 2. Install Docker and verify it

If WSL is not installed, run this in an administrator PowerShell window and restart when prompted:

```powershell
wsl --install
```

If it is already installed, you can update it:

```powershell
wsl --update
wsl --version
```

Install Docker Desktop by following Docker's official guide:
https://docs.docker.com/desktop/setup/install/windows-install/

Start Docker Desktop and use the WSL 2 backend with Linux containers. Wait for the interface to show Engine running, then run:

```powershell
docker version
docker compose version
docker info
node --version
npm --version
```

Acceptance check: `docker version` shows both Client and Server, and `docker info` returns normally. If you see an error saying the `dockerDesktopLinuxEngine` pipe does not exist, start Docker Desktop first; there is no need to reinstall sub2api.

## 3. Obtain the deployment file

For a first installation, use a new, empty directory. If you have already deployed it, do not overwrite the existing `.env` or initialize it again.

The following pins the source to the revision checked for this guide, making it easier to review. The image still uses a mutable tag; the version-recording method is described later.

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

Open and read `docker-compose.yml` first. Check the image sources and volume definitions. Do not run installation scripts from unknown sources.

## 4. Create private configuration

The script below generates separate random passwords and writes `.env` as UTF-8 without a BOM. It does not print passwords to the terminal. Enter a valid administrator email address; it is the local admin console login identifier and does not need to match your OpenAI account.

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

Record `ADMIN_EMAIL` and `ADMIN_PASSWORD` locally; you will use them to log in to the admin console. `.env` is not encrypted, so protect it like a password file. Consider storing a copy in a password manager.

Key notes:

- **You must set `BIND_HOST=127.0.0.1`**; listening on all interfaces by default does not meet this guide's local-only scope.
- After the first initialization, changing `POSTGRES_PASSWORD` in `.env` alone does not complete a database password rotation.
- Keep `JWT_SECRET` and `TOTP_ENCRYPTION_KEY`; do not generate new ones on every startup.
- Compose enables automatic setup; the admin initialization behavior may change between versions.

Start and verify:

```powershell
docker compose --env-file .env -f docker-compose.yml config --quiet
docker compose --env-file .env -f docker-compose.yml pull
docker compose --env-file .env -f docker-compose.yml up -d
docker compose ps
Invoke-RestMethod 'http://127.0.0.1:8080/health'
Start-Process 'http://127.0.0.1:8080'
```

The first image pull may take some time. Acceptance check: all three services start and then become healthy; the app's published port should be `127.0.0.1:8080->8080/tcp`, and the database and Redis should not publish ports to the host.

`config --quiet` only validates the configuration. Do not share the output of ordinary `docker compose config` in chat; it may contain expanded passwords.

## 5. Log into official Codex

Install the CLI by following OpenAI's official Codex documentation. A common npm installation method is:

```powershell
npm install -g @openai/codex
codex --version
codex login
```

Sign in to your own ChatGPT account on the official login page, choosing the ChatGPT sign-in option. Run `codex` once for a brief conversation to confirm that your current subscription can use Codex.

Check only whether the credentials file exists; do not display its contents:

```powershell
Test-Path (Join-Path $HOME '.codex\auth.json')
```

If you use a custom `CODEX_HOME`, use the location where the official client actually saves the file. Do not substitute browser cookies, the ChatGPT `sessionToken`, or web scraping for Codex OAuth. API-key login and ChatGPT subscription login are different billing paths.

## 6. Configure sub2api: gateway account, group, balance, and key

Open http://127.0.0.1:8080 in a browser and sign in with the administrator email and password from `.env`.

You may be asked to read and acknowledge a compliance document the first time. Read it yourself and decide whether to accept it. The confirmation phrase and document version depend on what the page shows at the time; do not copy a fixed phrase from an old guide.

### 6.1 Create an OpenAI group

In the admin console, go to Groups → Create:

- Example name: `personal-openai`.
- Platform: **OpenAI**.
- Standard type; multiplier **1** (must be greater than 0).
- Do not enable complex model routing, exclusivity, or filtering rules initially.

### 6.2 Import your Codex session

In the admin console, go to Accounts → Create or Import, and look for the dedicated **OpenAI OAuth → Codex session / auth.json** entry point.

- If the page allows file selection, select your own `auth.json`.
- If it accepts only text, open the file in a local text editor and paste its contents directly into the local admin console form. Do not paste it into chat.
- Select OpenAI as the platform and OAuth as the type, then bind the account to the `personal-openai` group you created.
- Give the account a name that contains no private information.
- When finished, use the admin console's test function to check the account status.

Note: Generic “data import” expects sub2api's own accounts/proxies format; it is **not the same as importing a Codex session**. If you cannot find a dedicated entry point, verify the deployment version first. Do not rewrite `auth.json` arbitrarily.

The dedicated API in the version checked for this guide is `POST /api/v1/admin/accounts/import/codex-session`. Its `content` field is a string containing the original JSON, and it also has fields such as `group_ids` and `update_existing`. Readers should prefer the UI to avoid manually handling the admin JWT and credentials.

An OAuth import containing a refresh token lets the gateway attempt renewal, but renewal is not guaranteed forever. An import containing only an access token cannot renew automatically. Do not interpret the “current expiration time” as a guarantee of permanent account availability.

### 6.3 Check balance and access

Standard mode checks the balance of the user managed by sub2api. The initial administrator balance may be 0; a successful account import does not guarantee that inference requests will pass.

In the admin console, set an appropriate local test balance for the **user who actually created the key**, and confirm that this user is allowed to use the OpenAI group. This is a metering value for the local gateway. It does not add funds to OpenAI or increase your ChatGPT subscription allowance. Do not enter an implausibly large value that exceeds the database field's range.

### 6.4 Create a gateway API key

Go to API Keys → Create. Use a name such as `zcode-local` and bind it to the `personal-openai` group.

Save the generated key in a password manager. It is a **sub2api gateway key**, not an OpenAI Platform key, an admin password, or an OAuth token.

Confirm that these three items are connected:

```text
Gateway key → OpenAI group ← Imported OpenAI OAuth account
```

## 7. Verify the Responses API first

Do not troubleshoot ZCode yet. Enter the gateway key locally in PowerShell; the code below will not echo the key.

Choose an exact model ID from the models available to your own account. Model IDs and entitlements can vary by account, region, and time; verify the exact name in the official Codex client or the current gateway model test page before entering `$modelId` below.

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

Acceptance check: the response `status` is `completed` and `output` contains the expected text. HTTP 200 alone is not enough to determine whether a streaming request completed successfully; later checks should show `response.completed`, not `response.failed` or a broken stream.

If the model does not accept `medium`, adjust it according to the upstream error message. Do not assume all models support the same effort values.

## 8. Add a native provider to ZCode's model list

In ZCode provider configuration, `group` is a local UI categorization field. It is distinct from the sub2api admin group. The sample script uses `standard-personal` as the ZCode-local category; it does not need to match the admin group `personal-openai`.

Prefer configuring this in ZCode's custom provider UI:

| Field | Value |
|---|---|
| Provider ID | `gpt-local` |
| Name | `GPT (sub2api local gateway)` |
| API type | **OpenAI Responses** / `openai-responses` |
| Base URL | `http://127.0.0.1:8080/v1` |
| API key | Gateway key from step 6.4 |
| Model ID | Exact ID verified as available in step 7 |

If the UI shows only Effort On/Off, add `reasoningLevel` rules for the model. The script below merges one provider, preserves other providers, and creates a backup before writing.

### 8.1 Safe merge script

1. Save the code below as a new file, for example `configure-zcode-gpt.cjs` on the desktop. Do not overwrite an existing script.
2. Close ZCode and run `node "$HOME\Desktop\configure-zcode-gpt.cjs"` in PowerShell. If your desktop is redirected to OneDrive, use the actual file path.
3. Enter your verified model ID, effort values, and gateway key. Key input is visible in the local terminal, so use a private environment; do not record the screen. The script does not write the key to logs or command-line history, but the final ZCode configuration stores it in plaintext.
4. Reopen ZCode and select the new model.

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

**Connect one model before expanding.** The script uses 128000 as the configured maximum output allowance; this is suitable only for models that actually support that value. For an unknown model, change it to a value allowed by the model's documentation. The `max` field does not mean that every request automatically sends 128000, and it does not guarantee that high effort will never hit the output limit.

The script deliberately does not claim in bulk that all models have a 1M context window or support PDF or JSON schema. When needed, add verified `properties` for that model according to the current ZCode schema.

### 8.2 Capability and effort reference

These are results probed on one local machine on 2026-11-13. They are samples for that account/channel at that time, not a permanent official model catalog:

| Model | Effort values accepted at the time |
|---|---|
| `gpt-6.1-sol`, `gpt-6`, `gpt-6-astra` | low, medium, high, xhigh, max |
| `gpt-6.1-terra`, `gpt-6-sol`, `gpt-6-luna` | none, low, medium, high, xhigh, max |
| `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.6` | none, low, medium, high, xhigh, max |

If `none` is unsupported, the upstream returns an explicit unsupported-value error. Do not map the effort toggle's `disabled` value to `none` unconditionally and expose it to models that do not support it.

- All 10 models accepted image input at the time. In simple red/blue image tests, `gpt-6-luna` misidentified the image several times; avoid it for critical visual operations. A small sample cannot establish an overall visual capability ranking.
- Successful tool calls are one prerequisite for computer use; they do not establish reliable real desktop operation.
- The gateway may have model alias mappings configured; names may not correspond one-to-one with actual upstream models. When comparing model capabilities, check the admin console mappings and logs.
- PDF, structured output, and the full context window were not independently verified in the final checks for this guide. Do not claim support solely because the configuration says `true`.

## 9. Initial acceptance checklist

Complete these in order. If a check fails, stop and troubleshoot at that layer:

1. **Docker**: All three containers are healthy, and the app publishes only to 127.0.0.1.
2. **OAuth account**: The account is usable in the admin console and bound to the OpenAI group; balance and permissions are valid.
3. **Gateway API**: The non-streaming request in step 7 returns `completed`.
4. **ZCode text**: In a new session, select `gpt-local/<model>` and ask it to reply with a fixed short phrase.
5. **Effort**: The menu shows the configured granular values, and switching values does not produce an unsupported-value error.
6. **Tools**: Ask the model to create or read a dedicated temporary test file, then confirm that it called the tool and read back the result. Do not test with real business data.
7. **Image**: Upload a non-sensitive test image you created and ask the model to describe it; confirm that more than just request acceptance occurred.
8. **Computer use (if needed)**: In an isolated test application, enter a short phrase and verify it with a screenshot. Do not start from a sensitive desktop.
9. **Maintenance**: Know how to start Docker, view logs, and restore the provider backup.

## 10. MCP management and reconnect loops

In ZCode Settings → MCP, you can manually start and stop servers; the UI toggle writes the `enabled` value to the configuration. On this machine, the disabled state was verified to persist; MSSQL can be left enabled separately.

**Enable only the MCP servers needed for the current task.** Turn off heavyweight servers when not in use. Turn them on manually when needed, then confirm whether the current session has received the tools. Hot-reload behavior may differ across versions; opening a new session and verifying is the safest approach. Do not assert that every existing session will fail to update.

User configuration is usually at `~/.zcode/cli/config.json`, while workspace configuration is usually at `<project>/.zcode/config.json`. Consider migration only when project isolation is needed. Preserve a backup during migration, check for name collisions and duplicate plugin sources, and do not assume that creating a workspace automatically migrates configuration scope.

**Real failure sample**: With all 311 tools mounted, terra repeatedly returned 502 for the same request. Removing the tool set made it succeed, while shortening the history but keeping all tools still failed. A specific subset of 270 items succeeded and subsets of 280+ failed; sol succeeded with the same large tool set. This indicates a compatibility problem with large tool sets on this channel. The experiment did not isolate the effects of tool count, schema size, and tool combinations, so it does not establish a universal official “270-tool hard limit.”

Mitigations: turn off unused MCP servers, reduce the schema, or switch to a model verified to work. Do not silently truncate tools or delete historical function_call/output entries; this can break call pairing.

`upstream_status:502` and `pool=1, excluded=1` only indicate that this request failed and that the single-account pool had no fallback. **They do not, by themselves, establish a global OpenAI outage.** First compare a short request with the same model, a request with fewer tools, and a different model.

## 11. Context, compaction, and output budgets

- Input context limits and output token limits are different; reasoning tokens may also consume the output budget.
- `contextWindow` for a custom provider is client metadata; it cannot enlarge the actual upstream window. Do not copy the locally configured value `1050000` to every model unconditionally.
- ZCode's automatic compaction is a client feature. Verify whether it is enabled, its trigger threshold, the summarization model it uses, and its retry behavior against the current version's settings and logs. The checks for this guide did not approach the real limit closely enough to trigger automatic compaction, so they do not guarantee that “it will always compact automatically when the limit is reached.”
- Before a long task, check whether a compact command is available in the `/` menu in the input box, and compact manually when needed. Keep important state in project documentation and continue in a new session if necessary.
- If you see `The model's response exceeded the output token maximum`, first lower effort or reduce the size of the individual task, then check the actual `max_output_tokens` sent and the model's limit. Raising the configured maximum may mitigate some cases, but it is not a guaranteed permanent fix.
- If compaction fails, inspect the error from its separate request. Do not assume it always includes the full tools set or always uses the current model.

## 12. Daily startup and shutdown

Each day:

1. Open Docker Desktop and wait for the engine to be ready. You can enable automatic startup when signing in to Windows in its settings.
2. You usually do not need to log into Codex again. The Compose restart policy should bring services back when Docker starts. If necessary, run:

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose up -d
docker compose ps
```

3. Open ZCode and select a `gpt-local` model. You do not need to keep the official Codex window or admin console open each time.

Temporarily stop the gateway while preserving data:

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose stop
# Restart the services
docker compose up -d
```

Closing Docker Desktop also stops the gateway; sessions currently using GPT will fail.

## 13. Troubleshooting table

| Symptom | Check first | Action |
|---|---|---|
| Cannot connect to localhost:8080 | Docker engine, containers, port conflict | Start Docker, then run `compose up -d`; if the port conflicts, change both the host port and the ZCode base URL |
| Login email rejected | Email format | Use a valid email format; do not use `local@localhost` |
| Compliance acknowledgment fails | Current document and confirmation phrase | Read the page and follow its current prompt; do not hard-code an old phrase |
| 401 | Wrong key, expired/revoked OAuth | Distinguish gateway authentication from upstream authentication; if needed, log into Codex again and update the import through the dedicated entry point |
| No OpenAI account / group mismatch | Key and whether the account is bound to the same OpenAI group | Correct the binding; check whether the account is paused or invalid |
| Insufficient balance / 403 | Local user balance and group permissions | Configure a local test balance; this does not increase the Plus allowance |
| model_not_found | Name, entitlement, mapping, or retirement | Choose from the models verified as available to the current account; do not copy old model names |
| unsupported reasoning.effort | Values allowed for each model | Remove unsupported values and configure according to the error message |
| Repeated reconnects / 502 | Exact request, tool set, network, upstream | Compare with a short request and turn off unused MCP servers; do not attribute the cause based only on the status code |
| 200 but fails midstream | SSE error, response.failed, EOF | Check the complete terminal event and usage; do not rely only on the HTTP status |
| Output-limit error | Effort, actual output cap, task length | Lower effort, split the task, and verify the allowed maximum and actual transmitted value |
| 429 / rate limit | Current subscription limit | Wait for the limit to reset or use another authorized provider; do not rotate multiple accounts to evade limits |
| Works in browser but Docker cannot reach upstream | Container DNS/network/proxy | Distinguish container networking from host networking; containers usually use `host.docker.internal` to reach the host |
| Effort still shows only On/Off | API type and per-model `optionSpecs` | Use Responses and configure `reasoningLevel`; confirm that the same provider/model is selected |

View local logs:

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose logs --since 10m --tail 150 sub2api
```

Logs may contain request content, account identifiers, or initialization credentials. Inspect and redact them locally before sharing an error summary. Do not send anyone `auth.json`, the complete `.env`, `provider_config`, or a complete packet capture. A temporary packet capture stores private session data; clean it up within the scope you have authorized after analysis.

## 14. Backup, upgrade, and rollback

Important: provider configuration backups also contain the key. Protect them as carefully as the original.

- Back up `.env`, the Compose file, and the ZCode provider configuration to a private location.
- Gateway account data and other state are stored in Docker volumes. Saving only Compose and `.env` is not a complete database backup. For important state, also back up PostgreSQL using its official procedure and verify that it can be restored.
- Record the actual image identifiers so the versions can be reviewed later:

```powershell
docker image inspect weishaw/sub2api:latest --format '{{json .RepoDigests}}'
docker compose images
```

This guide pins the Compose source revision, but the `latest`, `18-alpine`, and `8-alpine` image tags can change; for strict reproducibility, record or pin the digest. Do not blindly downgrade the database or open a new schema with an old application.

Read the release notes and complete a backup before upgrading. Then run:

```powershell
Set-Location (Join-Path $HOME 'sub2api-deploy')
docker compose pull
docker compose up -d
docker compose ps
```

To roll back the provider: close ZCode, restore the backup from step 8, reopen ZCode, and verify. To stop using it temporarily, disable `gpt-local` in the UI.

To stop the deployment, either `docker compose stop` or `docker compose down` will preserve named volumes. **Do not add `-v`**; `down -v` deletes persistent data and should be used only after explicitly deciding to destroy it permanently and completing a backup.

## 15. Reproducibility and acceptance

Results vary by machine, account, model, and software version. Do not treat one account's model list, MCP toggle state, or request result as your own acceptance result. Complete section 9 in your environment and record the deployment-file revision, image digest, ZCode and gateway versions, exact available model IDs, test time, terminal Responses event, and tool-call outcome. Do not record or publish OAuth tokens, gateway keys, admin passwords, or raw session content.

The model and tool observations in this guide are limited samples intended to flag possible compatibility issues. They are not service guarantees and do not prove that every account, region, or subscription has access. PDF, structured output, the actual context limit, automatic compaction, and real computer use should be verified independently in your own environment.

## 16. References

- sub2api repository: https://github.com/Wei-Shaw/sub2api
- Verified Compose revision: https://github.com/Wei-Shaw/sub2api/blob/530c5e3a4f9c73ca2b0881191d0ef8e0d82c5484/deploy/docker-compose.yml
- Docker Desktop for Windows: https://docs.docker.com/desktop/setup/install/windows-install/
- Codex CLI: https://github.com/openai/codex
- OpenAI Codex documentation: https://developers.openai.com/codex/
- OpenAI Terms of Use: https://openai.com/policies/terms-of-use/
- ZCode configuration paths and fields: checked against the configuration of the locally installed version and its bundled zcode-guide documentation; when the version changes, use the current client schema.

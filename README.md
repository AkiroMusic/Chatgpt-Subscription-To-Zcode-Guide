# ChatGPT Subscription to ZCode: Local Gateway Guide
# ChatGPT 订阅接入 ZCode：本地网关操作手册

A reproducible Windows guide for running your own local sub2api gateway and connecting it to ZCode with an OpenAI Responses provider.

面向 Windows 用户的可复现指南：运行自己的本地 sub2api 网关，并通过 OpenAI Responses 自定义 provider 接入 ZCode。

**Important / 重要提示:** This is a third-party integration, not an OpenAI-supported ZCode feature. It stores your own Codex OAuth credentials in software you operate. Read the upstream terms and the gateway's compliance notice before proceeding. Do not share credentials, accounts, or subscription access; do not use the setup to bypass limits.

这是第三方集成，不是 OpenAI 官方提供的 ZCode 功能。它会将你自己的 Codex OAuth 凭据保存在你运行的软件中。开始前请阅读上游服务条款及网关合规提示。不要共享账号、凭据或订阅额度，也不要用本方案绕过限额。

**Languages / 语言版本:** [English guide](ChatGPT-to-ZCode-Local-Gateway-Guide.en.md) | [简体中文指南](ChatGPT-to-ZCode-Local-Gateway-Guide.md)

## Architecture / 架构概览

```text
ZCode → http://127.0.0.1:8080/v1 → local sub2api / 本地 sub2api → your Codex OAuth / 你自己的 Codex OAuth
                                      ├─ PostgreSQL (Docker volume)
                                      └─ Redis (Docker volume)
```

## Contents / 文档内容

- [English full guide](ChatGPT-to-ZCode-Local-Gateway-Guide.en.md)
- [简体中文完整手册](ChatGPT-to-ZCode-Local-Gateway-Guide.md)

Both guides cover Docker Desktop, local-only deployment, Codex login, account/group/key setup, ZCode provider configuration, verification, MCP and reconnect troubleshooting, context/output-budget caveats, maintenance, backup, and rollback.

两份手册均涵盖 Docker Desktop、本机限定部署、Codex 登录、账号/分组/key 设置、ZCode provider 配置、验证、MCP 与重连排障、上下文和输出预算注意事项、维护、备份及回滚。

## Quick start / 快速步骤

1. Install Docker Desktop for Windows and Node.js LTS. / 安装 Windows 版 Docker Desktop 和 Node.js LTS。
2. Follow the detailed guide to deploy sub2api with `BIND_HOST=127.0.0.1`. / 按完整手册部署 sub2api，并设置 `BIND_HOST=127.0.0.1`。
3. Sign in to Codex using your own eligible ChatGPT account. / 使用自己有权使用的 ChatGPT 账号登录 Codex。
4. Import your own Codex session through sub2api's dedicated flow. / 通过 sub2api 专用入口导入自己的 Codex session。
5. Create an OpenAI group and gateway API key in the local admin console. / 在本地管理后台创建 OpenAI 分组和网关 API key。
6. Configure ZCode with API type `openai-responses`, base URL `http://127.0.0.1:8080/v1`, and the gateway key. / 在 ZCode 配置 API 类型 `openai-responses`、Base URL 和网关 key。
7. Complete the verification checklist before real use. / 完成验收清单后再用于真实工作。

This is only an outline. Read the full guide first; model availability, effort values, quotas, and UI labels vary by account and software version.

以上只是概要，请先阅读完整手册。模型可用性、推理挡位、额度和 UI 标签会随账号及软件版本变化。

## Scope and limitations / 范围与限制

This is a single-user, localhost-oriented setup guide, not a hosted service or a promise that the upstream permits every use of subscription OAuth through a third-party gateway. Subscription quotas and model access remain controlled by the upstream provider.

本指南面向单人、本机 localhost 部署；不是托管服务，也不保证上游允许订阅 OAuth 经第三方网关进行所有形式的使用。订阅额度和模型权限仍由上游服务控制。

Model and large-tool-set observations are dated, limited samples. They are not universal official model documentation or a guaranteed numeric tools limit. Verify behavior with your own account and current software versions. Automatic compaction, full context limits, PDF, structured output, and real computer-use require separate validation.

模型和大工具集观察是有日期的有限样本，不是普遍适用的官方型号文档，也不构成固定 tools 数上限。请用自己的账号和当前软件版本实测。自动 compact、完整上下文上限、PDF、结构化输出及真实 computer-use 都需要单独验证。

## Safety / 安全要求

- Never publish `auth.json`, `.env`, gateway keys, admin passwords, ZCode provider configuration, or raw session logs. / 绝不要发布这些私密文件和凭据。
- Bind the service to `127.0.0.1`; do not expose the admin console or API publicly without a separately reviewed secure deployment design. / 将服务绑定到 `127.0.0.1`，不要直接公开管理后台或 API。
- Keep Docker volumes and secrets backed up privately. Avoid `docker compose down -v` unless you intend to permanently delete gateway data. / 私密备份 Docker volumes 和密钥；除非确认永久删除，否则不要运行 `down -v`。
- Use your own account. Do not upload another person's OAuth credentials or share gateway access. / 只使用自己的账号，不上传他人的 OAuth 凭据或共享网关访问权限。

## References / 参考资料

- [sub2api](https://github.com/Wei-Shaw/sub2api)
- [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [OpenAI Codex documentation](https://developers.openai.com/codex/)
- [OpenAI Terms of Use](https://openai.com/policies/terms-of-use/)

# mist-relay-template

一个可自行部署到 Cloudflare Workers 的 Anthropic API 透传模板，基于
[`wusaki0723/mist`](https://github.com/wusaki0723/mist) 修改。

本仓库不包含任何部署者的账号信息、Worker 地址或密钥。每位使用者都需要使用自己的
Claude 订阅、Cloudflare 账户和独立 secrets。

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/plOrz/mist-relay-template)

## 部署前须知

- 这是非官方社区项目，不由 Anthropic 或 Cloudflare 提供支持。
- Worker 会把 Claude 长期 OAuth token 保存为 Cloudflare Secret，并代理 API 请求。
- 源码默认会向 system prompt 注入 Claude Code 前缀。此行为可能不符合上游服务条款，
  并可能导致账号受限；不能接受风险时请勿使用。
- Worker 源码不持久化或主动记录请求正文，但 Cloudflare 仍会处理流量，并可能按照其
  平台政策保留基础设施级元数据。
- `PROXY_API_KEY` 是公开 Worker 与 Claude 账号之间的唯一访问保护，请使用足够长的随机值。

## 1. 获取长期 token

需要有效的 Claude 订阅和 Claude Code：

```bash
claude setup-token
```

在浏览器中完成授权后，终端会显示一个以 `sk-ant-oat01-` 开头的 token。它是敏感凭据，
不要发送给任何人，也不要提交到 GitHub。

## 2. 部署 Worker

点击上方 **Deploy to Cloudflare** 按钮，使用自己的 GitHub 与 Cloudflare 账户完成部署。

如果一键部署导入失败，可使用 Wrangler：

```bash
git clone https://github.com/plOrz/mist-relay-template.git
cd mist-relay-template
npm ci
npx wrangler login
npm run deploy
```

## 3. 添加 secrets

在 Cloudflare Dashboard 中打开：

**Workers & Pages → mist → Settings → Variables and Secrets → Add**

添加以下两个值，类型均选择 **Secret**：

| 名称 | 值 |
| --- | --- |
| `CLAUDE_OAUTH_TOKEN` | 第 1 步生成的 setup-token |
| `PROXY_API_KEY` | 自己生成的长随机字符串 |

可用下面的命令生成代理密钥：

```bash
openssl rand -hex 32
```

也可以使用 Wrangler 交互式写入，输入内容不会提交到仓库：

```bash
npx wrangler secret put CLAUDE_OAUTH_TOKEN
npx wrangler secret put PROXY_API_KEY
```

## 4. 验证

```bash
curl https://<your-worker>.<your-subdomain>.workers.dev/health
```

成功配置后应看到：

```json
{
  "ok": true,
  "hasToken": true,
  "looksLikeSetupToken": true
}
```

健康检查只返回布尔状态，不显示任何 token 片段。

## 5. 连接 Claude Code

```bash
export ANTHROPIC_BASE_URL="https://<your-worker>.<your-subdomain>.workers.dev"
export ANTHROPIC_API_KEY="<your-PROXY_API_KEY>"
claude
```

`ANTHROPIC_BASE_URL` 不要添加 `/v1` 后缀。

## 更新 token

setup-token 过期或失效后，重新生成并替换 Cloudflare Secret：

```bash
claude setup-token
npx wrangler secret put CLAUDE_OAUTH_TOKEN
```

## 与上游相比的修改

- 修复 `package.json` 与 lockfile 的依赖版本不一致问题，使 `npm ci` 可复现。
- `/health` 不再返回 token 前缀。
- README 与仓库元数据通用化，不包含个人部署信息。

## License

本项目基于 [`wusaki0723/mist`](https://github.com/wusaki0723/mist)，依照
[MIT License](LICENSE) 分发。

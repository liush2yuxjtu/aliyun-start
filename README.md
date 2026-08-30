# aliyun-start

**One** entry-point agent skill (Claude Code / Codex) for everything **Alibaba Cloud (阿里云)** —
deploy & manage ECS / 轻量应用服务器 (SWAS), run commands without SSH (命令助手), open firewall ports,
get HTTPS **without buying a domain or 备案** (Caddy + sslip.io + TLS-ALPN-01), and diagnose networking.

It is a thin **front door** over the official
[`aliyun/alibabacloud-aiops-skills`](https://github.com/aliyun/alibabacloud-aiops-skills) bundle:
instead of installing all ~120 skills, this one skill loads your credentials, ensures the `aliyun` CLI +
SWAS plugin, routes your intent to the right action, and pulls individual bundle skills **on demand**.

## Install

### Pi

```bash
pi install git:github.com/liush2yuxjtu/aliyun-start
mkdir -p ~/.config/aliyun-start
cp ~/.pi/agent/git/github.com/liush2yuxjtu/aliyun-start/.env.example ~/.config/aliyun-start/.env
chmod 600 ~/.config/aliyun-start/.env   # then fill in your AccessKey
```

### Claude Code

```bash
git clone https://github.com/liush2yuxjtu/aliyun-start ~/.claude/skills/aliyun-start
mkdir -p ~/.config/aliyun-start
cp ~/.claude/skills/aliyun-start/.env.example ~/.config/aliyun-start/.env
chmod 600 ~/.config/aliyun-start/.env   # then fill in your AccessKey
```

Existing Claude installs using `~/.claude/skills/aliyun-start/.env` migrate automatically on first use.
Then invoke with `/skill:aliyun-start`, `/aliyun-start`, or natural language such as "部署到阿里云".

## Credentials

Credentials live at `~/.config/aliyun-start/.env`, outside package checkouts, so updates cannot delete them
and the repository never tracks them. Use a RAM user (not the root account) with
`AliyunSWASFullAccess` + `AliyunBSSReadOnlyAccess`. Disable/rotate the key in the RAM console when done.

## What it knows (baked-in gotchas)

- SWAS CLI needs **both** `--region <r>` (endpoint) **and** `--biz-region-id <r>` (request) or `NotFoundInstance`.
- SWAS plugin uses **kebab-case** subcommands; firewall `--port` is a **single port**.
- **命令助手 (run-command)** runs shell on the box with no SSH.
- Free-trial / 宝塔 images are **Alibaba Cloud Linux 3** (dnf, no apt/docker).
- Mainland-proof deploy: ship a node-linux-x64 binary + Next standalone, run via systemd.
- HTTPS with no domain/备案: Caddy + `<dashed-ip>.sslip.io` + TLS-ALPN-01 on 443.

## License

MIT

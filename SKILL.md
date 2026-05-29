---
name: aliyun-start
description: >
  ONE entry point for everything Alibaba Cloud (阿里云). Use when the user wants to deploy/manage
  Aliyun servers (ECS / 轻量应用服务器 SWAS), check status, SSH in, open firewall ports, get HTTPS,
  diagnose networking/reachability, or run any operation backed by the official Alibaba Cloud AIOps
  skills bundle (github.com/aliyun/alibabacloud-aiops-skills). Triggers: "/aliyun-start", "aliyun start",
  "部署到阿里云", "阿里云运维", "开/管服务器", "deploy to aliyun", "aliyun deploy", "云服务器", "轻量服务器",
  "ecs", "swas", "网络可达性", "给服务器加 HTTPS". This skill loads credentials from its own .env and
  routes the request to the right capability from the bundle. Do NOT install the 120 individual bundle
  skills; this single skill is the front door.
---

# aliyun-start — single front door to Alibaba Cloud

This is the **only** Aliyun skill. It (1) loads credentials, (2) ensures the `aliyun` CLI + SWAS plugin,
(3) routes the user's intent to the right action, pulling deeper instructions from the official bundle
on demand instead of installing 120 separate skills.

## Step 0 — load credentials (always run first)

```bash
SKILL_DIR="$HOME/.claude/skills/aliyun-start"
set -a; . "$SKILL_DIR/.env"; set +a
export ALIBABA_CLOUD_ACCESS_KEY_ID ALIBABA_CLOUD_ACCESS_KEY_SECRET
AL="$HOME/.local/bin/aliyun"; command -v aliyun >/dev/null && AL=aliyun
```

`.env` holds the AccessKey (gitignored). Never print the secret. If `.env` is missing, ask the user for
`ALIBABA_CLOUD_ACCESS_KEY_ID` + `_SECRET` (RAM user with `AliyunSWASFullAccess` + `AliyunBSSReadOnlyAccess`),
write them to `$SKILL_DIR/.env` (chmod 600), and remind them to disable the key when done.

## Step 1 — ensure tooling (idempotent)

```bash
# aliyun CLI (no brew needed)
if ! "$AL" version >/dev/null 2>&1; then
  curl -fsSL -o /tmp/acli.tgz "https://aliyuncli.alicdn.com/aliyun-cli-macosx-latest-arm64.tgz"
  tar xzf /tmp/acli.tgz -C /tmp && mkdir -p "$HOME/.local/bin" && cp /tmp/aliyun "$HOME/.local/bin/aliyun"
fi
# profile from .env
"$AL" configure set --profile "$ALIYUN_PROFILE" --mode AK --region "$ALIYUN_REGION" \
  --access-key-id "$ALIBABA_CLOUD_ACCESS_KEY_ID" --access-key-secret "$ALIBABA_CLOUD_ACCESS_KEY_SECRET" >/dev/null
# SWAS (轻量) plugin → kebab-case subcommands
"$AL" plugin install --names aliyun-cli-swas-open >/dev/null 2>&1 || true
"$AL" sts GetCallerIdentity --profile "$ALIYUN_PROFILE"   # verify
```

## ⚠️ Hard-won gotchas (read before any SWAS call)

- **Endpoint region must match the instance region.** Pass BOTH `--region <r>` (endpoint) AND
  `--biz-region-id <r>` (request) or you get `NotFoundInstance` against the wrong (hangzhou) endpoint.
- SWAS plugin uses **kebab-case**: `list-instances`, `run-command`, `create-firewall-rule`,
  `describe-command-invocations`. Region param is `--biz-region-id`, not `--region-id`.
- **Firewall `--port` is a single port** like `3000` (range form `a/b` is rejected for single ports).
- **命令助手 (run-command)** lets you run shell on the box with **no SSH** — great for injecting an SSH key
  or one-off ops. Poll output via `describe-command-invocations --invoke-id <id>` (Output is base64).
- Free-trial / 宝塔 images are **Alibaba Cloud Linux 3** (dnf/yum, NO apt, NO docker preinstalled),
  and free-trial 轻量 cannot pick Ubuntu — don't assume Debian/apt.

## Routing — pick the matching playbook

| User wants | Do this |
|---|---|
| **Status / list servers** | `"$AL" swas-open list-instances --region $SWAS_REGION --biz-region-id $SWAS_REGION` (or `ecs DescribeInstances`). Balance: `bssopenapi QueryAccountBalance`. |
| **Run a command on the box (no SSH)** | `"$AL" swas-open run-command --region $R --biz-region-id $R --instance-id $SWAS_INSTANCE_ID --name c --type RunShellScript --command-content '<sh>'` → poll `describe-command-invocations`. |
| **Get SSH access** | run-command to append your pubkey to `/root/.ssh/authorized_keys`; then `create-firewall-rule` port 22; `ssh -i $SSH_KEY root@$SERVER_IP`. |
| **Open a port** | `"$AL" swas-open create-firewall-rule --region $R --biz-region-id $R --instance-id $SWAS_INSTANCE_ID --rule-protocol TCP --port <N> --remark <x>` |
| **Deploy a Next.js/Node app (mainland-proof)** | Build standalone locally (`DOCKER_BUILD=1 pnpm build`), ship a node-linux-x64 binary + `.next/standalone`+static+public via `tar | ssh`, run with a `systemd` unit (`Restart=always`). Avoids slow on-box npm/docker pulls. |
| **HTTPS without a domain or 备案** | Caddy static binary + hostname `<dashed-ip>.sslip.io` + Caddyfile with `tls { issuer acme { disable_http_challenge } }` and global `auto_https disable_redirects` → Let's Encrypt **TLS-ALPN-01** on 443 only (dodges mainland port-80/备案). Open firewall 443. |
| **Buy / provision a new server** | Cheapest cash via API is normal 包月 (e.g. SWAS 2C2G); promo ¥9.9/free-trial are **web-only** (coupon) — send the user to `https://www.aliyun.com/daily-act/ecs/activity_selection` / `https://free.aliyun.com/`. API `create-instances` auto-charges balance, so CONFIRM spec+cost first. |
| **Network reachability / why can't I reach it** | Pull the bundle skill `alibabacloud-network-reachability-analysis` (see below); also test from a China vantage (the box itself) — a local Mac behind a VPN often can't reach mainland:443. |
| **Anything else (RDS, OSS, WAF, domain, 备案 query, ECS appmanager deploy, …)** | Pull the specific bundle skill on demand (next section). |

## Using the official bundle on demand (do NOT bulk-install)

The bundle has ~120 skills under categories: computing, storage, netcdn, security, database, middleware,
developertools, migrationom, etc. Fetch just the one you need as reference, then drive `aliyun` CLI:

```bash
# discover
curl -s "https://raw.githubusercontent.com/aliyun/alibabacloud-aiops-skills/main/skills/<path>/SKILL.md"
# high-value paths:
#  computing/computenest/alibabacloud-ecs-code-deploy        (aliyun appmanager deploy to ECS)
#  developertools/solutions/alibabacloud-terraform-code-generation
#  developertools/solutions/alibabacloud-cli-guidance
#  netcdn/netana/alibabacloud-network-reachability-analysis
#  doweb/companyreg/alibabacloud-icpba-sucessdata-query      (ICP 备案 query)
#  doweb/domain/alibabacloud-domain-manage
#  database/rds/alibabacloud-rds-copilot · database/kvstore/alibabacloud-tair-devtoolset
#  security/cloudfw/* · security/waf/* · computing/ecs/alibabacloud-ecs-diagnose
```

If the user explicitly wants a bundle skill installed as its own Claude skill, you may
`npx skills add aliyun/alibabacloud-aiops-skills --skill <name>` — but default to the on-demand
reference approach so this stays the single front door.

## Current known deployment (this account)

`.env` is pre-seeded with a live box: **上海 / `$SERVER_IP` / `https://$SERVER_HOSTNAME`** (SWAS free trial,
2C4G, Alibaba Cloud Linux 3, expires 2026-06-29), running the **sub3api** gateway via systemd `sub3api.service`,
HTTPS via Caddy `caddy.service`. SSH: `ssh -i $SSH_KEY root@$SERVER_IP`.

## Safety

- Provisioning a paid resource or `create-instances` spends real money — confirm spec + cost first.
- Never echo the AccessKey secret. After the engagement, advise rotating/disabling the key in the RAM console.

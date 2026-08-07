# GitHub Org：NOTIFY 配置指南

将下列项配置在 **GitHub Organization**（推荐）Settings → Secrets and variables → Actions。

## 必填

| 名称 | 类型 | 来源 | 示例值 |
|---|---|---|---|
| `NOTIFY_WORKER_URL` | Org **Variable** | notify-worker 部署后的公网根 URL（非密钥） | `https://notify-worker.your-account.workers.dev` |
| `NOTIFY_GHA_TOKEN` | Org **Secret** | 与 notify-worker 侧 `npx wrangler secret put NOTIFY_GHA_TOKEN` 填入的**同值** | 随机长字符串 |

## 配置步骤

### 1. 生成 GHA token

```bash
openssl rand -hex 32
```

### 2. 写入 notify-worker

```bash
cd notify-worker
npx wrangler secret put NOTIFY_GHA_TOKEN
# 粘贴上一步生成的值
npx wrangler deploy
```

### 3. 写入 GitHub Org

1. Variables：`https://github.com/organizations/<org>/settings/variables/actions`  
   → New organization variable → `NOTIFY_WORKER_URL`
2. Secrets：`https://github.com/organizations/<org>/settings/secrets/actions`  
   → New organization secret → `NOTIFY_GHA_TOKEN`（与 wrangler 同值）

Org Variable / Secret 可被组织内仓库 workflow 通过 `${{ vars.NOTIFY_WORKER_URL }}` / `${{ secrets.NOTIFY_GHA_TOKEN }}` 引用（需授予仓库访问权限）。

### 4. 消费方 workflow env 约定

Action 运行时通过 **env**（非 with）传入：

```yaml
env:
  NOTIFY_WORKER_URL: ${{ vars.NOTIFY_WORKER_URL }}
  NOTIFY_AUTH_TOKEN: ${{ secrets.NOTIFY_GHA_TOKEN }}
```

| 运行时 env | 对应 GitHub 配置 | 说明 |
|---|---|---|
| `NOTIFY_WORKER_URL` | Variable `NOTIFY_WORKER_URL` | Worker 根 URL |
| `NOTIFY_AUTH_TOKEN` | Secret `NOTIFY_GHA_TOKEN` | Action 固定读取此 env 名 |

## 轮换 token

1. 生成新 token → `wrangler secret put NOTIFY_GHA_TOKEN`
2. 更新 Org `NOTIFY_GHA_TOKEN`
3. 旧 token 立即失效；`NOTIFY_AUTH_TOKEN`（Worker 间）不受影响

## 不需要配置的 Secret

- `RESEND_API_KEY` — 仅 notify-worker 持有
- `NOTIFY_AUTH_TOKEN` — 仅 Cloudflare Worker Service Binding 消费方使用，**不要**放入 GitHub Secrets

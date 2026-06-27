# action-notify-email

GitHub Composite Action：作为 [notify-worker](../notify-worker/) 在 GitHub 平台的 HTTP 代理层，统一 CI 邮件通知，**不在各仓库配置 Resend Key**。

## 快速接入

### 1. Org / 仓库 Secrets

在 GitHub Organization（推荐）或单个仓库 Settings → Secrets and variables → Actions 中配置：

| Secret | 说明 |
|---|---|
| `NOTIFY_WORKER_URL` | notify-worker 公开地址（不含路径），如 `https://notify-worker.<account>.workers.dev` |
| `NOTIFY_GHA_TOKEN` | 与 notify-worker 侧 `NOTIFY_GHA_TOKEN` wrangler secret **同值** |

> Worker 部署与 token 说明见 [notify-worker/README.md](../notify-worker/README.md)。  
> Org Secrets 配置见 [docs/ORG_SECRETS.md](docs/ORG_SECRETS.md)。

### 2. Workflow 引用

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ... your steps ...

      - name: Notify on failure
        if: failure()
        uses: ONGOING-Z/action-notify-email@v1
        with:
          subject: "CI 失败: ${{ github.repository }}"
          body: |
            Workflow: ${{ github.workflow }}
            Branch: ${{ github.ref_name }}
            Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          dedup-key: "${{ github.workflow }}-${{ github.sha }}"
        env:
          NOTIFY_WORKER_URL: ${{ secrets.NOTIFY_WORKER_URL }}
          NOTIFY_AUTH_TOKEN: ${{ secrets.NOTIFY_GHA_TOKEN }}
```

`NOTIFY_AUTH_TOKEN` 是 Action 运行时 env 名（固定）；值来自仓库/Org 的 `NOTIFY_GHA_TOKEN` secret。

## Inputs

| 名称 | 必填 | 默认 | 说明 |
|---|---|---|---|
| `subject` | 是 | — | 邮件标题 |
| `body` | 是 | — | 纯文本正文 |
| `to` | 否 | — | 收件人；缺省用 notify-worker `DEFAULT_TO` |
| `dedup-key` | 否 | — | KV 去重键，建议 `workflow-sha` |
| `fail-on-error` | 否 | `true` | 发信失败是否 fail job |

## Outputs

| 名称 | 说明 |
|---|---|
| `ok` | `true` / `false` |
| `skipped` | 去重跳过时 `true` |
| `id` | Resend message id |
| `error` | 错误信息 |

## 行为

- `POST {NOTIFY_WORKER_URL}/v1/send`，Bearer `NOTIFY_AUTH_TOKEN`
- Header `X-Notify-Source: github-actions`
- 失败时 300ms 后重试 1 次（与 orchestrator notify step 一致）
- 日志不输出 token

## 发布

```bash
git tag v1.0.0
git push origin v1.0.0
git tag -f v1 && git push origin v1 -f   # 主版本指针
```

消费方引用 `@v1` 即可跟踪同主版本更新。

## 安全提示

- 不要在 `body` 中粘贴 secrets 或完整 CI 日志（可能含敏感信息）
- matrix job 请共用 `dedup-key`，避免重复发信
- 吊销 GHA 访问时只需 rotate notify-worker 的 `NOTIFY_GHA_TOKEN`，不影响 Worker 间 `NOTIFY_AUTH_TOKEN`

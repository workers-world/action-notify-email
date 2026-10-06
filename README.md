# action-notify-email

GitHub Composite Action：作为 [notify-worker](../notify-worker/) 在 GitHub 平台的 HTTP 代理层，统一 CI 邮件通知，**不在各仓库配置 Resend Key**。

## 快速接入

### 1. Org / 仓库配置

在 GitHub Organization（推荐）Settings → Secrets and variables → Actions 中配置：

| 名称 | 类型 | 说明 |
|---|---|---|
| `NOTIFY_WORKER_URL` | **Variable** | notify-worker 公开地址（不含路径），如 `https://notify-worker.<account>.workers.dev` |
| `NOTIFY_GHA_TOKEN` | **Secret** | 与 notify-worker 侧 `NOTIFY_GHA_TOKEN` wrangler secret **同值** |

> Worker 部署与 token 说明见 [notify-worker/README.md](../notify-worker/README.md)。  
> Org 配置见 [docs/ORG_SECRETS.md](docs/ORG_SECRETS.md)。

### 2. Workflow 引用

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ... your steps ...

      - name: Notify on failure
        if: failure()
        uses: workers-world/action-notify-email@v1
        with:
          subject: "CI 失败: ${{ github.repository }}"
          body: |
            Workflow: ${{ github.workflow }}
            Branch: ${{ github.ref_name }}
            Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          # 推荐同时传 html，邮件客户端中链接可直接点击
          html: |
            <div style="font-family:system-ui,sans-serif;line-height:1.5">
              <p><strong>CI 失败</strong></p>
              <ul>
                <li>Workflow: ${{ github.workflow }}</li>
                <li>Branch: ${{ github.ref_name }}</li>
                <li>Run: <a href="${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}">查看 workflow run</a></li>
              </ul>
            </div>
          dedup-key: "${{ github.workflow }}-${{ github.sha }}"
        env:
          NOTIFY_WORKER_URL: ${{ vars.NOTIFY_WORKER_URL }}
          NOTIFY_AUTH_TOKEN: ${{ secrets.NOTIFY_GHA_TOKEN }}
```

`NOTIFY_AUTH_TOKEN` 是 Action 运行时 env 名（固定）；值来自仓库/Org 的 `NOTIFY_GHA_TOKEN` secret。

仅传 `body`（无 `html`）时，notify-worker 会把正文中的 `http(s)://` URL **自动 linkify** 成可点击链接；显式 `html` 不会被覆盖。

## Inputs

| 名称 | 必填 | 默认 | 说明 |
|---|---|---|---|
| `subject` | 是 | — | 邮件标题 |
| `body` | 否* | — | 纯文本正文 |
| `body-file` | 否* | — | 从文件读取纯文本正文；与 `body` 同时存在时优先 `body-file` |
| `html` | 否* | — | HTML 正文（大体积 digest 请改用 `html-file`，避免 Actions 日志打印 `with:` 且绕过 shell ARG_MAX） |
| `html-file` | 否* | — | 从文件读取 HTML 正文；与 `html` 同时存在时优先 `html-file` |
| `to` | 否 | — | 收件人；缺省用 notify-worker `DEFAULT_TO` |
| `dedup-key` | 否 | — | KV 去重键，建议 `workflow-sha` |
| `fail-on-error` | 否 | `true` | 发信失败是否 fail job |

\* `body` / `body-file` 与 `html` / `html-file` 至少提供一个。

大 HTML（如 weekly digest）建议先写入文件再传 `html-file`，例如：

```yaml
- run: printf '%s' "$HTML" > digest.html
  env:
    HTML: ${{ steps.build.outputs.html }}
- uses: workers-world/action-notify-email@v1
  with:
    subject: Weekly digest
    html-file: digest.html
    dedup-key: digest-${{ github.sha }}
```

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
- 运行环境时区固定为 `Asia/Shanghai`（`TZ` env），日志时间戳为上海时间
- 失败时 300ms 后重试 1 次（与 orchestrator notify step 一致）
- 日志不输出 token
- `NOTIFY_WORKER_URL` / `NOTIFY_AUTH_TOKEN` 为空时始终打 `::error::`；若 `fail-on-error: false` 另打 `::warning::` 后 exit 0（Release PR `notify-blocked` 另有前置 Guard 步骤，配置缺失直接 fail job）
- reusable workflow **勿**再 `secrets: NOTIFY_WORKER_URL`（已改为 Variable）；caller 用 `secrets: inherit` 仅继承 `NOTIFY_GHA_TOKEN`

## 发布

**不要手工打 tag。** 发布全自动：

1. 功能 PR 合入当前 `dev_*` → Validate 绿 → Promote 开/合 Release PR（`dev_*` → `master`）
2. push `master` 触发 [`release-tag.yml`](.github/workflows/release-tag.yml)：在最新 `vX.Y.Z` 上 patch+1 打不可变 tag，并把主版本指针 `v1` 前移到同一提交
3. `vX.Y.Z` tag 触发 [`gh-release-on-tag.yml`](.github/workflows/gh-release-on-tag.yml) 建 GitHub Release 页

minor / major：Actions → **Release action tag** → Run workflow（branch 选 `master`），填写 `version`（如 `1.2.0`）。

消费方引用 `@v1` 即可跟踪同主版本更新。

## 安全提示

- 不要在 `body` 中粘贴 secrets 或完整 CI 日志（可能含敏感信息）
- matrix job 请共用 `dedup-key`，避免重复发信
- 吊销 GHA 访问时只需 rotate notify-worker 的 `NOTIFY_GHA_TOKEN`，不影响 Worker 间 `NOTIFY_AUTH_TOKEN`

# 使用外部 Cron 触发 GitHub Actions

项目的 `run.py` 是一次性命令行程序，没有 HTTP 服务。外部 Cron（例如
[cron-job.org](https://cron-job.org/)）应当调用 GitHub 的
`workflow_dispatch` API，由 GitHub Actions 负责安装 Playwright、读取 Secrets 并执行发送任务。

```text
cron-job.org -> GitHub workflow_dispatch API -> .github/workflows/send.yml -> python run.py
```

## 1. 确认工作流触发方式

`.github/workflows/send.yml` 已保留 `workflow_dispatch`，因此可以通过 API 触发。
仓库不应同时启用 GitHub Actions 的 `schedule` 和外部 Cron，否则同一天可能执行两次。

## 2. 创建专用 GitHub Token

在 GitHub 的 [Fine-grained personal access tokens](https://github.com/settings/personal-access-tokens/new)
页面创建 Token：

- Repository access：只选择 `wuya11111/douyin-auto-fire`；
- Repository permissions：`Actions: Read and write`；
- 有效期建议设置为 6 个月或 1 年，并设置到期提醒。

Token 只用于触发工作流，不要提交到仓库或写进 URL。

## 3. 配置 cron-job.org

新建任务，配置以下内容：

请求 URL：

```text
https://api.github.com/repos/wuya11111/douyin-auto-fire/actions/workflows/send.yml/dispatches
```

请求方法：`POST`

请求头：

```text
Accept: application/vnd.github+json
Authorization: Bearer github_pat_你的Token
X-GitHub-Api-Version: 2022-11-28
Content-Type: application/json
User-Agent: cron-job-douyin-auto
```

请求体：

```json
{
  "ref": "main",
  "inputs": {
    "dry_run": "false"
  }
}
```

在 cron-job.org 中将时区设置为 `Asia/Shanghai`，例如：

```text
每天 08:30：30 8 * * *
每天 20:00：0 20 * * *
```

如果使用 UTC 时区，北京时间需要减 8 小时。

## 4. 测试触发

第一次建议将请求体中的 `dry_run` 改成 `true`。发送请求后，GitHub API 成功时返回
HTTP `204 No Content`，随后可以在仓库的 **Actions** 页面看到一条
`workflow_dispatch` 运行记录。

确认 Dry Run 成功后，再改回 `false` 执行真实发送。

也可以使用以下命令测试（将 Token 替换为实际值）：

```bash
curl -X POST \
  -H 'Accept: application/vnd.github+json' \
  -H 'Authorization: Bearer github_pat_你的Token' \
  -H 'X-GitHub-Api-Version: 2022-11-28' \
  -H 'Content-Type: application/json' \
  -H 'User-Agent: cron-job-douyin-auto' \
  'https://api.github.com/repos/wuya11111/douyin-auto-fire/actions/workflows/send.yml/dispatches' \
  -d '{"ref":"main","inputs":{"dry_run":"true"}}'
```

## 5. Token 到期与安全

Token 到期后，Cron 请求会返回 `401 Unauthorized`，工作流不会运行。续期步骤为：

1. 创建一个具有相同仓库和权限的新 Token；
2. 先在 cron-job.org 中更新 `Authorization` 请求头；
3. 测试并确认返回 `204`；
4. 再撤销旧 Token。

建议关闭 cron-job.org 的失败自动重试，避免一次网络错误触发多个工作流。

GitHub Actions Runner 仍可能因排队出现延迟。如果必须精确到某一分钟执行，请将项目部署到 VPS，使用项目自带的 `systemd timer` 直接运行 `python run.py`。

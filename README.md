# anyrouter status page

一个可迁移的最小状态页项目：

- 静态页面：`docs/`
- 探测脚本：`scripts/check_anyrouter.py`
- GitHub Actions 模板：`.github/workflows/status-check.yml`

## 功能

- 每次探测发送一个最小的、按当前 Claude Code CLI 关键字段对齐的请求到 anyrouter
- `max_tokens=1`
- 记录：
  - 当前 HTTP status code
  - 是否成功吐出文本
  - 最近错误消息
  - 最近探测耗时
- 当前线上部署由 Cloudflare Worker 定时触发 GitHub Action
- 触发频率：每 10 分钟一次
- 只保留最近 7 天，按小时聚合

## 本地运行

1. 复制配置文件：

   ```bash
   cp .env.example .env
   ```

2. 填入：

   - `ANYROUTER_API_BASE`
   - `ANYROUTER_API_KEY`
   - `ANYROUTER_MODEL`（推荐：`claude-opus-4-7[1m]`）

3. 安装依赖：

   ```bash
   pip install -r requirements.txt
   ```

4. 执行探测：

   ```bash
   python scripts/check_anyrouter.py
   ```

5. 打开 `docs/index.html` 预览页面，或用任意静态服务器托管 `docs/`。

## 部署到新仓库

推荐把本目录内容提升到新仓库根目录，最终结构类似：

- `.github/workflows/status-check.yml`
- `docs/`
- `scripts/`
- `requirements.txt`

然后：

1. GitHub Pages 指向 `docs/`
2. 仓库 Secrets 配置：
   - `ANYROUTER_API_BASE`
   - `ANYROUTER_API_KEY`
   - `ANYROUTER_MODEL`
3. 启用 Actions
4. 使用 Cloudflare Worker 定时触发该 workflow 的 `workflow_dispatch`
   - 当前线上配置为每 10 分钟触发一次

## 说明

- GitHub 只会识别仓库根目录下的 `.github/workflows/`。当前目录里的 workflow 是迁移模板，默认按“迁移后位于仓库根目录”来写。
- 当前仓库的定时调度由 Cloudflare Worker 负责，GitHub Actions 侧通过 `workflow_dispatch` 接收触发。
- 探测失败时脚本仍会写入状态文件；只有缺少配置或写文件失败时才会退出非零。

## Opus 4.7[1m] 兼容说明

2026-04-17 本地抓包对比后确认，`claude-opus-4-7[1m]` 不能用的原因不是“模型名本身错了”，而是**旧探测脚本模拟的 Claude Code 请求已经落后于当前真实 CLI 请求格式**。

旧脚本的问题主要有：

- 还在发送旧的最小请求体，例如 `temperature=0`
- `anthropic-beta` 组合偏旧
- 缺少当前 Claude Code CLI 会带的关键结构，例如：
  - `thinking: {"type": "adaptive"}`
  - `output_config: {"effort": "medium"}`
  - `context_management`
  - Claude Code 风格的 billing/system envelope

在当前 anyrouter / new-api 链路上，这些差异会导致两种现象：

- 用旧格式探测 `claude-opus-4-7[1m]` 时，网关可能直接返回 `500 new_api_panic`
- 用真实 Claude Code CLI 请求同一模型时，可以正常返回 `200`

因此脚本已经改成按当前真实 Claude Code CLI 的关键字段来发探测请求：

- 默认示例模型更新为 `claude-opus-4-7[1m]`
- `[1m]` 后缀只作为“启用 1M context beta header”的便捷写法；实际发出的 `model` 会去掉 `[1m]`
- 不再发送 `temperature`
- 发送：
  - `thinking: {"type": "adaptive"}`
  - `output_config: {"effort": "medium"}`
  - `context_management`
  - Claude Code 风格的 system / metadata / headers
- 仍保持探测足够便宜：
  - `max_tokens=1`
  - `tools=[]`
  - `stream=false`

如果你后面再切换 Claude 新模型，优先建议以**真实 Claude Code CLI 抓包结果**为准，而不是只改模型名。

## 🚩 友情链接

感谢 **LinuxDo** 社区的支持！

[![LinuxDo](https://img.shields.io/badge/社区-LinuxDo-blue?style=for-the-badge)](https://linux.do/)

## 致敬

- [lsdefine/GenericAgent](https://github.com/lsdefine/GenericAgent)
  - 本项目中模拟 Claude Code CLI 发送请求的方式参考了这个项目。

# 会话进度日志

## 2026-11-13
- 确认仓库状态：v0.20.0 detached HEAD，工作区干净。
- 发现 analysis/ 目录不存在，参照文件缺失 → 决策 D1。
- 创建规划文件。
- 确认 v0.20.0 结构：chat completion 位于 vllm/entrypoints/openai/chat_completion/（serving.py 1818 行、protocol.py 906 行、api_router.py、stream_harmony.py）；v1 引擎位于 vllm/v1/engine/ 与 vllm/v1/core/sched/scheduler.py（2304 行）、gpu_model_runner.py（6996 行）。
- 派出 8 个并行 Explore 代理：HTTP 层 / 协议+chat template / AsyncLLM+IPC / Scheduler+KV / Worker+模型执行 / 输出处理+响应 / 启动初始化 / 分支补充路径。
- 搭建 /tmp/vllm-doc-gen/ 下的 HTML 生成器框架。
- 确认工具链：google-chrome 可用（/usr/bin/google-chrome），node v24.15.0，python 3.12.3。
- 完成模板 template.html + build.py + check.js（/tmp/vllm-doc-gen/）。
- 代理 1（HTTP 层）完成并已记入 findings.md。
- 全部 8 个研究代理完成，发现已归档 findings.md（82 行密集锚点）。
- 开始编写请求数据 JSON（分 4 个分段文件：http/protocol/engine/sched/worker/output）。
- 请求链 HTML 生成完成：analysis/vllm-chat-completions-call-chain.html（122 KB，11 阶段 72 步骤，480 条三入口源码链接）。
- 启动时序 HTML 生成完成：analysis/vllm-startup-call-timeline.html（42 KB，7 阶段 20 步骤）。
- headless Chrome 验证全部通过：渲染、搜索、筛选、展开、TOC、桌面/移动端截图、深色主题。
- 源码引用审计：201 个引用全部文件存在、行号不越界。
- 开始提交推送。

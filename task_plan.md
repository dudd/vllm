# 任务计划：vLLM /v1/chat/completions 源码级调用链分析 → 离线 HTML 文档

## 目标
1. 分析 vLLM v0.20.0 中 POST /v1/chat/completions 的完整源码调用链。
2. 生成可直接离线打开的独立 HTML 分析文档（请求链时序 + 阶段导航 + 搜索/筛选/主题）。
3. 重新审计主链，补充分支（Harmony/GPT-OSS、beam search、抢占、spec decode、KV connector 等）。
4. 制作启动调用时序文档（原参照文件 analysis/vllm-startup-sequence-full.html 不存在，需自行设计统一视觉结构）。
5. headless Chrome 验证 HTML 质量。
6. 推送到 origin git@github.com:dudd/vllm.git 的 v0.20.0-claude 分支。

## 约束
- 不修改 vLLM 源码；不打开任何可见 GUI。
- 页面无外部 CDN 依赖，支持浅色/深色主题、响应式。
- 源码链接提供 Zed、Cursor、原始文件三种入口。
- 每步标注调用类型（sync/async/threadpool/IPC/GPU/SSE）。
- 提交信息注明 AI 协助来源。

## 阶段
- [x] 阶段 0：环境确认（v0.20.0 detached HEAD，工作区干净；analysis/ 目录不存在）
- [ ] 阶段 1：主调用链源码分析（HTTP→协议→chat→processor→engine→scheduler→worker→output→response）
- [ ] 阶段 2：分支与补充审计（Harmony、beam search、抢占、spec decode、KV connector、abort、stats 等）
- [ ] 阶段 3：启动时序链分析
- [ ] 阶段 4：生成请求链 HTML 文档
- [ ] 阶段 5：生成启动时序 HTML 文档
- [ ] 阶段 6：headless Chrome 验证并修复
- [ ] 阶段 7：创建 v0.20.0-claude 分支、提交、推送、核对远端

## 关键决策
- D1：参照文件 analysis/vllm-startup-sequence-full.html 不存在 → 两份文档采用同一套自行设计的视觉体系，保持信息结构一致（编号逻辑、下一跳、深链接三入口）。
- D2：基线代码 = 当前 checkout 的 v0.20.0 标签，所有行号以此为准。
- D3：文档放在 analysis/ 目录下（新建）。

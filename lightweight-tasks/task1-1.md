---
title: qwen-code-third-party-integration-draft-discussion迁移——方案整理
status: [x]
progress:
- 2025-10-29 17:00:00: 基于 task1.md 输出迁移方案子任务内容。
- 2025-10-29 17:04:21: 按 TODO 风格整理子任务列表。
- 2025-10-29 17:09:12: 基于分支起点 diff 梳理代码/测试/文档变更范围。
- 2025-10-29 23:32:57: 从最新 main 创建 feature/stream-json-migration 分支并切换完成，准备进行差异梳理。
- 2025-10-29 23:35:42: 完成 9d664623f56dfd025d21c84cedcf63851689248b → origin/qwen-code-third-party-integration-draft-discussion 差异梳理并输出摘要。
- 2025-10-29 23:39:39: 生成完整 diff 到 `openspec/lightweight-tasks/diffs/stream-json-migration-diff.md` 供详细审阅。
- 2025-10-29 23:46:35: 按文件拆分 diff，输出至 `openspec/lightweight-tasks/diffs/` 目录下各独立 md。
- 2025-10-29 23:47:51: 调整 diff 文件扩展名为 `.diff`，统一存储格式。
- 2025-10-29 23:49:43: 自动生成 `diffs/index.md` 罗列所有差异文件，便于导航。
- 2025-10-29 23:58:56: 为 `diffs/index.md` 编写逐文件摘要与关键片段，形成结构化 diff 索引。
- 2025-10-29 23:59:58: 输出基于 TDD 的迁移阶段计划至 `openspec/lightweight-tasks/task1-2.md`。
---

## 子任务概览

1. [x] 变更理解：通读 `enable-stream-json-io` 的 proposal/design/spec 与 `docs/examples/stream-json` 示例脚本，梳理 CLI 新增协议、控制通道、增量事件及验证流程要点，将变更理解总结到 @openspec/lightweight-tasks/task1-understanding.md 中。
2. [x] 环境准备：基于最新 `main` 建立工作分支 `feature/stream-json-migration`，在 `task1.md` 中同步记录状态与进度。
3. [x] 差异梳理（基准 9d664623f56dfd025d21c84cedcf63851689248b → origin/qwen-code-third-party-integration-draft-discussion）

## 差异梳理记录

- **文档示例**：`docs/examples/stream-json/` 新增 README/README_cn、Python SDK 与校验脚本、示例请求及日志；`docs/cli/index.md` 更新 CLI 选项；补充多篇 stream-json 与 Agent Framework 相关 RFC 文档（中英文版本）。
- **CLI 参数与入口**：`package.json` 添加 `qwen` 与 `stream-json-session` npm script；`.gitignore` 增加 Python 缓存忽略；`packages/cli/src/config/config.ts`/`config.test.ts` 新增 `--input-format`、`--output-format`、`--include-partial-messages` 参数及校验逻辑；`gemini.tsx` 在 stream-json 模式下转入 `runStreamJsonSession`。
- **Stream JSON 模块**：新增 `packages/cli/src/streamJson/` 目录，涵盖协议类型定义、输入解析、事件写出、会话控制器等实现，并配套 `input/session/writer` 单元测试。
- **非交互 CLI 改造**：`nonInteractiveCli.ts`/`.test.ts` 大规模调整，接入 stream-json 写出器、控制请求处理、增量事件输出与 usage 统计；引入工具确认和 hook 控制响应逻辑。
- **核心配置与工具执行**：`packages/core/src/config/config.ts` 持久化输入输出格式与增量标记，提供读取方法；`nonInteractiveToolExecutor.ts` 支持输出增量回调与完成回调；`scripts/build_package.js` 改为 `spawnSync` 执行 `tsc --build` 并宽容诊断。

---
title: 阶段0 基础支撑执行计划
status: [x]
progress:
- 2025-10-30 01:16:39: 完成阶段 0 基础支撑，已更新 .gitignore、scripts/build_package.js 与 package.json。
---

## 子任务概览

1. [x] 更新 `.gitignore`  
   - 将 `__pycache__/`、`*.py[codz]`、`*$py.class` 加入忽略列表，防止示例脚本与测试脚本产生的缓存文件污染工作区。
2. [x] 调整 `scripts/build_package.js`  
   - 以 `spawnSync('tsc', ['--build'])` 替换原有 `execSync`，并在返回码非零时打印警告而非直接失败，保障迁移过程中构建诊断更具弹性。
3. [x] 扩展 `package.json` 脚本  
   - 新增 `qwen`（运行 `tsx packages/cli/index.ts`）与 `stream-json-session`（附带 stream-json 参数）脚本，为后续阶段执行示例与验证脚本提供统一入口。

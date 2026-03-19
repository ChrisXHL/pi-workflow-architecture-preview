# Session Notes

## 用户原始目标
把 workflow 插件画个架构图，并进一步做成可打开查看的 `preview.html` 可视化版。

## 本次我对系统的理解
目标项目是 `/Users/chris/pi-workflow-board` 中的 `workflow-engine.ts` 扩展。
它实现的是一个：
- 文件驱动的 workflow runtime
- 串行 phase runner
- 隔离 worker 执行模型
- 带审批门和 TUI 可视化的工作流面板

## 本次依据的主要源码
- `pi-workflow-board/.pi/extensions/workflow-engine.ts`
- `pi-workflow-board/README.md`

## 关键结论
1. 真正的核心在 `workflow-engine.ts`，README 只是对外说明。
2. 状态事实来源是 `workflow/` 目录下的文件，而不是 UI。
3. 变量池 `vars.json` 是上下游节点 prompt 注入总线。
4. 单节点通过隔离的 `pi --no-session` worker 执行。
5. 当前模型更像串行 phase runner，而不是通用 DAG 调度器。

## 当前未确定项
1. 用户说“workflow 插件”，这里按 `pi-workflow-board` MVP scaffold 理解，而不是其他同名项目。
2. 目前没有额外读取全局 `workflow-creator` / `article-worker` skill 的内部实现，只把它们视为节点 skill 依赖方。
3. 没有把 `.disabled` 版本单独画入图中，因为其逻辑与当前启用文件基本同构。

## 本次假设
1. 用户更关心“里面的逻辑怎么流转”，所以用了三张视图：总体架构、单节点执行、状态模型。
2. 重点解释运行时，而不是 UI 样式细节或未来规划。
3. 预览页用于讲解与复核，因此采用 stage tabs 组织多视角而不是一张超大杂糅图。

## 后续可继续迭代的方向
1. 再加一张“命令到状态迁移”的状态机图。
2. 再加一张“默认 article-pipeline 样例”的业务流图。
3. 如果你愿意，也可以把这版再精简成一页适合对外分享的老板版图。

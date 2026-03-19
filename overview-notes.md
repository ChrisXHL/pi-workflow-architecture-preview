# 业务输入驱动结构图说明

这次不再从 `/wf-next` 这种系统命令开始看，而是先从一个真实业务案例开始：

**目标：** 产出一篇可发布文章  
**输入：** 主题、素材、受众、约束  
**输出：** final article

## 为什么这样画

如果只从 runtime 视角开始，用户看到的是：
- command
- engine
- state.json
- worker

但看不出来：
- 这条 workflow 在业务上到底解决什么问题
- 每个阶段的业务价值是什么
- 为什么会有 review / approval 这个门

所以这一版强制拆成两层：

### 上层：业务阶段流
- init：先产出计划
- research：产出调研材料
- draft：产出初稿
- review：产出审阅意见并等待人工审批
- final：产出终稿

### 下层：runtime 承载层
- spec/input：把业务任务正式定义成 workflow
- workflow-engine：按顺序编排 phase
- state/jobs/vars/artifacts：保存运行事实与业务产物
- isolated worker：执行单节点
- UI：展示状态并接入人工干预

## 最关键的理解

上层回答：**这条业务流程是怎么走的**  
下层回答：**这条业务流程是怎么被系统稳定跑出来的**

## 系统定位

这套插件当前更像：
- 文件驱动的串行 workflow runtime
- 带人工审核门的 phase runner
- 单节点隔离 worker 编排器

而不是：
- 通用 DAG 调度平台
- 并行拓扑引擎
- 大规模任务队列系统

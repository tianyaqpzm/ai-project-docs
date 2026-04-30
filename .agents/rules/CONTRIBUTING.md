---
trigger: always_on
---

# 全局项目规范
大模型生成的 Implementation Plan(实施计划) 必须采用中文
## 文档架构与路径定义（文档使用中文）
  文档架构与路径定义所有跨工程的逻辑必须遵循“全局大脑”原则，文档统一存放于工作区根目录。
|文档类型|输出路径(自根目录起)|描述|
|---|---|---|
|API定义|/docs/api-contracts/{ProjectName-FeatureName-Version}.yaml|记录子工程的业务API接口文档,需包含API接口说明、入参出参、异常情况，OpenAPI 3.0.0 (YAML 格式)命名举例：文件名称MsJavaBiz-PregramerRecipes-v1.yaml,API名称/rest/dark/v1/knowledge/topics|
|特性定义|/docs/features/{feature-name}.md|记录特性的业务逻辑、涉及的子工程、接口变动|
|问题复盘|/docs/incidents/{date}-{issue}.md|记录线上故障或重大 Bug 的根因分析（RCA）|
|架构决策|/docs/architecture/{id}-decision.md|记录技术选型、架构调整的决策过程（ADR）|
|影响评估|/docs/impact-logs.md|影响评估汇总记录每次重大修改对其他工程的潜在影响，按时间维度排序，最新的在最前面|


## 约束
核心研发流程（使用中文记录在案），使用antigravity 配置awesome-skills需按照下面规则：
文档先行：
 * 通过@brainstorming 或@feature-spec 在编写任何代码前先输出需求或特性文档
 * 在完成一个功能模块，自动提取GIT修改或当前上下文，生成人类可读的变更记录

根因分析（RCA）机制：
 * 通过@debugging-strategies，在遇到复杂问题按步骤进行排查，留下排查思路
 * @root-cause-analysis(或@bug-report):问题解决后，总结复盘

修改影响控制：
 * @impact-analysis 或@architecture-review: 修改核心代码前 需评估风险
 * @test-driven-development(TDD): 利用TDD技能，强制AI先写测试用例，再写代码，代码提交前保证用例的通过率

根据问题根因判断尽可能放在一个文档中（文件名要更具有问题的区分性），避免流水账,提炼经验（尽可能精简）到对应工程下的 ai-code-ws.md 文件

## 注意事项：
 * 完善、追加内容时避免覆盖上一行内容
 * 实施计划 采用中文描述
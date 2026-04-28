---
trigger: always_on
---

# 全局编程规范 (Global Coding Standards)

AI 在生成代码时应遵循以下逻辑，旨在保证代码的低耦合、高内聚。所有子工程必须遵循以下统一的规范，以保持代码库的一致性。

## 一、 代码编程规范 (Coding Standards)

### 1. 命名约定 (Naming Conventions)
* **语义化**: 变量名必须反映其用途（如 `is_user_active` 优于 `flag`）。
* **格式一致性**: 
  * Python: 函数与变量用 `snake_case`，类名用 `PascalCase`。
  * Java/TS: 变量与方法用 `camelCase`，类名用 `PascalCase`。
* **常量**: 全大写加下划线，例如 `MAX_RETRY_COUNT`。

### 2. 函数与方法 (Functions)
* **单一职责 (SRP)**: 一个函数只做一件事。
* **长度限制**: 单个函数逻辑代码建议不超过 50 行，文件不建议超过 300 行，如果超过，考虑拆分。
* **参数控制**: 函数参数原则上不超过 5 个。超过时应封装为对象或配置字典。
* **尽早返回 (Early Return)**: 优先处理异常边界情况，减少 `if-else` 嵌套深度。

### 3. 注释与文档 (Documentation)
* **代码自解释**: 良好的命名应减少注释需求。注释应解释“为什么”这么做，而非“做了什么”。
* **标准格式**: Python 必须包含 Docstring，Java 必须包含 Javadoc，TS 符合 JSDoc。
* **类型注解**: 强类型语言（Java/TypeScript）严禁大量使用 `Any`/`Object`；Python 建议标注类型提示（Type Hints）。

### 4. 错误处理 (Error Handling)
* **禁止静默失败**: 严禁空的 `try-except` 或 `try-catch` 块。
* **精准捕获**: 只捕获预期的异常，不建议直接捕获基类 `Exception`。
* **防御性编程**: 对所有外部输入（API、用户输入）进行合法性校验。

## 二、 架构规范 (Architectural Standards)
用于指导 AI 如何组织文件结构和处理组件间的通信。

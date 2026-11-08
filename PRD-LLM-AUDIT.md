# Cross Platform Agent 代码推理审计 PRD

## 1. 文档信息

- 文档名称：Cross Platform Agent 代码推理审计 PRD
- 文档版本：v1.0
- 文档状态：初稿
- 适用范围：`cross-platform-agent` 及其上层审计编排服务

---

## 2. 背景

当前 `cross-platform-agent` 已具备以下能力：

- 通过 REST / WebSocket 接收结构化请求
- 调用系统 `git` 客户端完成 Git 操作
- 通过 `command.run` 在白名单范围内执行本机命令
- 在 `git.commit` / `git.pull` / `git.push` 失败时尝试弹出本机提示框

项目当前定位明确：

- Agent 是本地命令执行层
- Git 核心能力继续依赖系统 Git 客户端
- Agent 不负责自然语言理解和任务规划

随着后续接入大模型的需求增加，系统需要支持在 `git.commit`、`git.push` 等关键操作前，对待提交代码进行推理审计，例如：

- 安全风险检查
- 代码正确性检查
- 敏感信息泄露检查
- 潜在破坏性变更识别

为了保持 Agent 的“薄执行层”定位，需要将大模型调用与审计策略放在 Agent 外部，由独立的审计编排服务负责。

---

## 3. 产品目标

本期目标：

1. 在不破坏 Agent 薄执行层定位的前提下，引入代码推理审计能力
2. 支持在 `commit` / `push` 前由上层服务调用大模型完成代码审计
3. 支持将审计结果用于提示、人工确认或阻断执行
4. 保持跨平台兼容 Windows / Linux / macOS
5. 保持底层 Git 能力仍然使用本机 `git` 客户端

非目标：

1. 不在本期将大模型能力直接内嵌到 Rust Agent 中
2. 不在本期让 Agent 承担工作流编排职责
3. 不在本期做完整 CI/CD 平台
4. 不在本期支持任意自然语言指令直接进入命令执行层

---

## 4. 核心设计原则

### 4.1 Agent 保持薄

Rust Agent 只负责：

- 参数校验
- 路径白名单校验
- 命令白名单校验
- Git / 文件 / 命令执行
- 执行结果返回

Rust Agent 不负责：

- Prompt 组装
- 大模型 API 调用
- 审计策略判定
- 多步流程编排
- 人工审批流

### 4.2 Git 能力继续依赖系统 Git

所有 Git 相关能力继续通过系统 `git` 命令实现，不引入 libgit2 作为核心实现路径。

### 4.3 LLM 审计上移

代码推理审计能力放入外部“审计编排服务”中，由该服务：

- 从 Agent 获取 diff / 文件 / Git 状态
- 调用大模型接口
- 生成结构化审计结果
- 决定是否继续调用 `git.commit` / `git.push`

### 4.4 默认最小上传原则

送给大模型的内容必须最小化，优先上传：

- 变更 diff
- 变更文件列表
- 必要的少量上下文文件

默认不上传整个仓库。

---

## 5. 目标用户与使用场景

### 5.1 目标用户

- 本地开发者工具使用者
- IDE / Web 管理端调用方
- 上层 Agent / MCP / Workflow 编排系统
- 对提交前安全审计有要求的团队

### 5.2 核心场景

#### 场景 A：提交前审计

用户在本地改完代码后，调用上层服务发起“提交前审计”：

1. 上层服务从 Agent 获取 staged diff
2. 调用大模型做推理审计
3. 若通过，则执行 `git.commit`
4. 若存在高风险问题，则提示或阻断

#### 场景 B：推送前审计

用户在准备推送到远端分支前，调用“推送前审计”：

1. 上层服务获取待推送 diff
2. 调用模型判断风险
3. 若通过则执行 `git.push`
4. 若不通过则阻断或要求人工确认

#### 场景 C：仅审计不执行

仅对变更代码进行分析，输出审计建议，不触发 Git 写操作。

---

## 6. 总体架构

推荐架构如下：

```text
Web / IDE / Tool
        |
        v
Audit Orchestrator
   |          |
   |          +--> LLM Provider(OpenAI / 本地模型 / 其他)
   |
   +--> cross-platform-agent
             |
             +--> git / filesystem / allowed commands
```

模块职责：

### 6.1 Rust Agent

- 提供底层 REST / WebSocket 能力
- 提供 Git 操作能力
- 提供最小文件读取能力
- 提供受控命令执行能力

### 6.2 Audit Orchestrator

- 聚合上层请求
- 组装审计上下文
- 调用大模型
- 解析模型输出
- 基于策略决定是否执行后续 Git 动作
- 记录审计日志

### 6.3 LLM Provider

- 提供代码理解和风险判断能力
- 返回结构化审计结果

---

## 7. 功能范围

## 7.1 本期必须支持

### A. Agent 侧新增最小基础能力

1. `git.diff`
2. `git.changed_files`
3. `fs.read_text`
4. `git.show_file`

说明：

- 这些能力仍是薄封装
- 底层实现继续调用系统 Git 或标准文件读取
- 不引入审计策略

### B. 编排服务侧高层工作流

1. `commit_with_audit`
2. `push_with_audit`
3. `review_diff`

### C. 审计策略模式

1. `off`
2. `advisory`
3. `manual_approval`
4. `blocking`

### D. 审计结果结构化输出

至少包含：

- verdict
- summary
- findings
- severity
- file
- line
- suggestion

## 7.2 本期不做

1. 不支持任意 shell 指令推理
2. 不支持直接上传整个仓库给模型
3. 不支持在 Agent 内部内嵌多模型路由
4. 不支持在 Agent 内部维护会话记忆
5. 不支持复杂多人审批流

---

## 8. 功能需求明细

## 8.1 Agent 新增能力需求

### 8.1.1 `git.diff`

目的：

- 获取指定范围的 diff 文本

典型用途：

- staged 代码审计
- push 前差异审计

建议参数：

```json
{
  "action": "git.diff",
  "params": {
    "repo_path": "/absolute/path/to/repo",
    "staged": true,
    "base_ref": "origin/main",
    "head_ref": "HEAD",
    "context_lines": 3
  }
}
```

规则：

- `staged=true` 时返回 staged diff
- 指定 `base_ref/head_ref` 时返回 revision range diff
- 必须受路径白名单控制

### 8.1.2 `git.changed_files`

目的：

- 返回变更文件列表

典型用途：

- 控制模型上下文
- 选择需要补充读取的关键文件

建议参数：

```json
{
  "action": "git.changed_files",
  "params": {
    "repo_path": "/absolute/path/to/repo",
    "staged": true,
    "base_ref": "origin/main",
    "head_ref": "HEAD"
  }
}
```

### 8.1.3 `fs.read_text`

目的：

- 读取文本文件内容

典型用途：

- 为模型补充少量上下文文件

建议参数：

```json
{
  "action": "fs.read_text",
  "params": {
    "path": "/absolute/path/to/file",
    "max_bytes": 65536
  }
}
```

规则：

- 仅允许读取文本文件
- 必须位于 `AGENT_ALLOWED_ROOTS`
- 需限制最大大小

### 8.1.4 `git.show_file`

目的：

- 读取某个 revision 下的文件内容

典型用途：

- 比较修改前后的上下文
- 读取 `HEAD` 中的老版本文件

建议参数：

```json
{
  "action": "git.show_file",
  "params": {
    "repo_path": "/absolute/path/to/repo",
    "revision": "HEAD",
    "file_path": "src/main.rs"
  }
}
```

---

## 8.2 编排服务高层接口需求

### 8.2.1 `commit_with_audit`

目标：

- 在执行 `git.commit` 前先完成 LLM 审计

输入：

```json
{
  "repo_path": "/path/to/repo",
  "commit_message": "fix: handle null config",
  "audit_policy": {
    "mode": "blocking",
    "severity_threshold": "high",
    "profile": "security,correctness"
  },
  "model": {
    "provider": "openai",
    "model": "gpt-5-mini"
  }
}
```

输出：

```json
{
  "status": "blocked",
  "commit_executed": false,
  "audit": {
    "verdict": "block",
    "summary": "发现 1 个高风险问题",
    "findings": [
      {
        "severity": "high",
        "category": "security",
        "file": "src/auth.rs",
        "line": 42,
        "title": "可能缺少权限校验",
        "suggestion": "在写操作前校验当前用户权限"
      }
    ]
  }
}
```

处理流程：

1. 读取 staged diff
2. 读取变更文件列表
3. 按需补充上下文文件
4. 调用大模型
5. 生成结构化审计结果
6. 根据策略决定是否执行 `git.commit`

### 8.2.2 `push_with_audit`

目标：

- 在执行 `git.push` 前审计待推送代码

输入示例：

```json
{
  "repo_path": "/path/to/repo",
  "remote": "origin",
  "branch": "main",
  "audit_range": {
    "base_ref": "origin/main",
    "head_ref": "HEAD"
  },
  "audit_policy": {
    "mode": "manual_approval",
    "severity_threshold": "medium",
    "profile": "security,secrets,breaking-change"
  }
}
```

处理流程：

1. 读取待推送 diff
2. 审计差异
3. 返回结果
4. 若符合策略则执行 `git.push`

### 8.2.3 `review_diff`

目标：

- 仅审计，不执行 Git 写操作

适用场景：

- PR 创建前审计
- IDE 内提示
- 代码生成后自动复核

---

## 9. 审计策略要求

## 9.1 策略模式

### `off`

- 不做审计
- 直接执行 Git 动作

### `advisory`

- 审计结果仅提示
- 不阻断执行

### `manual_approval`

- 有风险时要求人工确认
- 确认后才继续执行

### `blocking`

- 达到风险阈值时阻断执行

## 9.2 默认策略建议

- `git.commit`：
  - 默认 `advisory` 或 `manual_approval`

- `git.push` 到主分支：
  - 默认 `blocking`

- 发布或生产类分支：
  - 默认 `blocking`

---

## 10. 模型输入输出要求

## 10.1 输入要求

模型输入应以最小必要原则构造，优先包含：

- diff 文本
- 变更文件列表
- 提交信息
- 少量上下文文件
- 可选的语言或框架信息

需限制：

- 单次最大文件数
- 单文件最大字符数
- 最大总 token 预算

需跳过：

- 二进制文件
- 超大文件
- `node_modules`
- `dist`
- `target`
- 构建产物

## 10.2 输出要求

模型输出必须是结构化 JSON，不接受仅自然语言自由输出。

建议结构：

```json
{
  "verdict": "pass | warn | block",
  "summary": "string",
  "findings": [
    {
      "severity": "low | medium | high | critical",
      "category": "security | correctness | secret | performance | maintainability",
      "file": "string",
      "line": 0,
      "title": "string",
      "evidence": "string",
      "suggestion": "string"
    }
  ]
}
```

---

## 11. 异常与降级策略

## 11.1 模型调用失败

### 对 `commit`

建议默认：

- 记录失败原因
- 提示“审计未完成”
- 可配置是否允许继续提交

### 对 `push`

建议默认：

- 主分支默认阻断
- 非主分支可配置为提示后继续

## 11.2 Agent 调用失败

例如：

- 仓库路径非法
- Git 命令失败
- 文件超限
- 命令超时

要求：

- 必须保留底层原始错误
- 编排服务需返回结构化失败原因

## 11.3 模型结果不可解析

要求：

- 视为审计失败
- 不直接默认为通过

---

## 12. 安全与合规要求

1. 大模型 API Key 不放入本地 Rust Agent
2. 审计上传内容必须最小化
3. 编排服务需记录上传了哪些文件 / diff
4. 对敏感仓库应支持仅本地模型或禁用外发
5. Agent 继续保持路径白名单与命令白名单约束
6. 不允许通过审计接口绕过 `command.run` 限制

---

## 13. 用户体验要求

1. 当审计阻断 `commit` / `push` 时，调用方应获得明确原因
2. 发现问题时，结果应尽量定位到文件和行号
3. 返回结果需区分：
   - 审计通过
   - 审计警告
   - 审计阻断
   - 审计失败
4. 对 `manual_approval` 模式，应支持上层 UI 进行确认交互

---

## 14. 验收标准

## 14.1 Agent 层验收

1. 新增动作 `git.diff`、`git.changed_files`、`fs.read_text`、`git.show_file`
2. 新动作全部遵守路径白名单
3. Git 核心实现继续使用系统 `git`
4. 不引入大模型 SDK 到 Agent 核心路径

## 14.2 编排层验收

1. 支持 `commit_with_audit`
2. 支持 `push_with_audit`
3. 支持 `review_diff`
4. 审计结果为结构化 JSON
5. 支持 `advisory` / `manual_approval` / `blocking`

## 14.3 体验验收

1. 提交前审计可在 1 次调用链中完成
2. 当发现高风险问题时可阻断执行
3. 阻断结果中包含摘要与问题列表
4. 模型调用失败时有清晰降级策略

---

## 15. 里程碑建议

### M1：补齐 Agent 最小能力

- `git.diff`
- `git.changed_files`
- `fs.read_text`
- `git.show_file`

### M2：完成编排服务最小闭环

- `review_diff`
- `commit_with_audit`

### M3：完成推送前审计

- `push_with_audit`
- 主分支阻断策略

### M4：增强能力

- secret 扫描
- 审计日志持久化
- 本地模型支持
- 多模型 Provider 支持

---

## 16. 结论

本项目的最佳演进路径不是把大模型能力直接塞进 Rust Agent，而是保持 Agent 作为“本地受控执行层”，将代码推理审计能力放到外部编排服务中。

这样可以同时满足：

- 架构边界清晰
- 跨平台一致性
- Git 依赖系统客户端
- 大模型能力可插拔
- 安全与审计策略可演进

该方案兼顾了当前项目的薄层定位与未来接入 LLM 审计的扩展性，适合作为下一阶段实现基线。

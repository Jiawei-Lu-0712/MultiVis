# 可视化Agent系统规则框架

## 概述

本文档描述了可视化Agent系统中实际实现的规则框架。该框架通过四类核心规则约束和指导多Agent协作的决策过程，确保系统行为的一致性、可靠性和可维护性。

## 核心设计理念

- **决策标准化**: 通过形式化规则替代复杂的条件判断
- **行为一致性**: 相同输入在相同条件下产生一致的处理流程
- **错误可控**: 标准化的错误检测、分类和恢复机制
- **流程透明**: 每个决策都有明确的规则依据和执行路径

## 规则分类体系

### 1. 任务协调规则 (Task Coordination Rules - TC-Rule)
负责任务分类、Agent选择和工作流程控制

### 2. 工具执行规则 (Tool Execution Rules - TE-Rule)  
约束工具调用的参数验证、边界检查和执行环境

### 3. 错误处理规则 (Error Handling Rules - EH-Rule)
提供标准化的错误检测、分类和恢复机制

### 4. ReAct控制规则 (ReAct Control Rules - RC-Rule)
控制ReAct模式的迭代过程、响应格式和终止条件

---

## TC-Rule: 任务协调规则

### TC-Rule-1: 任务类型分类规则
**功能**: 根据输入参数自动判断任务类型

**形式化定义**:
```
TaskType: (Q, D, R?, C?) → {A, B, C, D}

TaskType(q, d, r, c) = {
    D,  if c ≠ ∅ ∧ exists(c)
    C,  if r ≠ ∅ ∧ ext(r) ∈ {.py, .ipynb}
    B,  if r ≠ ∅ ∧ ext(r) ∈ {.png, .jpg, .jpeg, .gif, .bmp}
    A,  otherwise
}

其中:
- Q: 用户查询空间
- D: 数据库路径空间
- R: 参考文件路径空间 (可选)
- C: 现有代码路径空间 (可选)
- exists(·): 文件存在性谓词
- ext(·): 文件扩展名提取函数
```

### TC-Rule-2: 工作流程推荐规则
**功能**: 根据任务类型推荐执行路径和工具调用顺序

**形式化定义**:
```
Workflow: TaskType → Sequence(Tool)

Workflow(t) = {
    ⟨eval, modify, eval⟩,                           if t = D
    ⟨sql_gen, code_gen, eval, modify⟩,              if t ∈ {A, B, C}
}

其中:
- sql_gen = generate_sql_from_query
- code_gen = generate_visualization_code  
- eval = evaluate_visualization
- modify = modify_visualization_code
- ⟨·⟩: 有序序列
```

### TC-Rule-3: 工具前置条件验证规则
**功能**: 验证工具调用的必要条件

**形式化定义**:
```
Prerequisites: Tool → P(Attribute)

Prerequisites(tool) = {
    {user_query, db_path},                          if tool = sql_gen
    {db_path, user_query, sql_query},              if tool = code_gen
    {visualization_code, recommendations},          if tool = modify
    {user_query, visualization_code},              if tool = eval
}

Validate(tool, state) ≡ Prerequisites(tool) ⊆ defined(state)

其中:
- P(·): 幂集
- defined(·): 状态中已定义属性的集合
```

### TC-Rule-4: 评估响应决策规则
**功能**: 根据评估结果决定后续动作

**形式化定义**:
```
NextAction: EvalResult → Action

NextAction(result) = {
    complete,           if result.passed = true
    modify_sql,         if result.sql_recommendations ≠ ∅
    modify_code,        if result.recommendations ≠ ∅ ∧ result.sql_recommendations = ∅
    retry_or_restart,    otherwise
}

其中:
- EvalResult = {passed: Boolean, recommendations: List, sql_recommendations: List}
- Action = {complete, modify_sql, modify_code, retry_or_restart}
```

### TC-Rule-5: 预迭代控制规则
**功能**: 控制ReAct模式启动前的预处理步骤

**形式化定义**:
```
PreIteration: TaskType × State → MessageSequence

PreIteration(t, s) = ⟨user_msg, assistant_msg, observation_msg⟩

其中:
first_action(t) = {
    evaluate_visualization,     if t = D
    generate_sql_from_query,    otherwise
}

assistant_msg = ⟨Thought(first_action(t)), Action(first_action(t))⟩
observation_msg = ⟨Observation(execute(first_action(t), s))⟩
```

### TC-Rule-6: 状态转换验证规则
**功能**: 验证Agent间状态转换的合法性

**形式化定义**:
```
StateTransition: Tool × Tool → Boolean

ValidTransitions = {
    (sql_gen, code_gen),
    (code_gen, eval),
    (eval, modify),
    (eval, complete),
    (modify, eval)
}

StateTransition(from, to) ≡ (from, to) ∈ ValidTransitions
```

---

## TE-Rule: 工具执行规则

### TE-Rule-1: 参数边界约束规则
**功能**: 限制查询参数防止过度查询和资源消耗

**形式化定义**:
```
ParameterConstraint: Parameter × Value → Value

ParameterConstraint(param, value) = {
    min(value, 5),      if param = sample_size
    min(value, 20),     if param = max_rows
    value,              otherwise
}

ApplyConstraints(params) = {(p, ParameterConstraint(p, v)) | (p, v) ∈ params}
```

### TE-Rule-2: 工具参数验证规则
**功能**: 验证工具调用参数的完整性和正确性

**形式化定义**:
```
ParameterValidation: Tool × Parameters → Boolean

ParameterValidation(tool, params) ≡ 
    RequiredParams(tool) ⊆ domain(params) ∧
    ∀p ∈ domain(params): TypeCheck(p, params(p), ExpectedType(tool, p))

其中:
- domain(·): 参数字典的键集合
- RequiredParams(·): 工具必需参数集合
- TypeCheck(·,·,·): 类型检查谓词
- ExpectedType(·,·): 期望参数类型函数
```

### TE-Rule-3: 代码执行环境规则
**功能**: 标准化代码执行环境和变量提取

**形式化定义**:
```
ExecutionEnvironment = {
    namespace: {alt, pd, np, io, os},
    chart_extraction: Pattern(r'(\w+)\s*=\s*alt\.Chart'),
    save_injection: InjectSave
}

InjectSave(code, output_path) = {
    code ∪ {chart_var.save(output_path)},   if ¬HasSave(code) ∧ chart_var ≠ ∅
    code,                                   otherwise
}

其中:
- chart_var = ExtractChartVariable(code)
- HasSave(·): 检查代码是否包含保存命令的谓词
```

---

## EH-Rule: 错误处理规则

### EH-Rule-1: 工具调用解析错误规则
**功能**: 处理Agent生成的格式错误

**形式化定义**:
```
ParseError: Text → ErrorType

ParseError(text) = {
    null_input,         if text = ∅
    missing_end_tag,    if "<Action>" ∈ text ∧ "</Action>" ∉ text
    json_error,         if ¬ValidJSON(ExtractAction(text))
    missing_fields,     if ¬HasRequiredFields(ParseJSON(ExtractAction(text)))
    format_error,       otherwise
}

其中:
- ExtractAction(·): 提取Action标签内容
- ValidJSON(·): JSON格式验证谓词
- HasRequiredFields(·): 检查必需字段存在性
```

### EH-Rule-2: 代码执行错误恢复规则
**功能**: 代码执行失败时的恢复策略

**形式化定义**:
```
ErrorRecovery: ErrorType × Context → RecoveryAction

ErrorRecovery(error, context) = {
    SuggestInstallation(ExtractModule(error)),      if error = ImportError
    ProvideLocationInfo(error.lineno),              if error = SyntaxError
    SuggestPermissionFix(context.output_path),     if error = PermissionError
    ProvideDebugInfo(error.traceback),             otherwise
}
```

### EH-Rule-3: 评估失败处理规则
**功能**: 处理可视化评估失败的情况并生成恢复策略

**形式化定义**:
```
EvaluationFailureHandler: EvalResult → RecoveryPlan

EvaluationFailureHandler(eval_result) = {
    ForcedFailureRecovery(eval_result),     if eval_result.automatic_failure_triggered = true
    IssueClassificationRecovery(eval_result), otherwise
}

ForcedFailureRecovery(eval_result) = {
    strategy: "forced_failure",
    reasons: eval_result.failure_reasons,
    next_action: "retry_or_restart"
}

IssueClassificationRecovery(eval_result) = 
    let issues = eval_result.failure_reasons ∪ eval_result.unmet_requirements in
    let classified = ClassifyIssues(issues) in
    {
        strategy: "issue_based_recovery",
        classification: classified,
        next_action: DetermineNextAction(classified)
    }

ClassifyIssues: P(Issue) → Classification

ClassifyIssues(issues) = {
    critical: {i ∈ issues | IsBlankIssue(i) ∨ IsEmptyIssue(i)},
    moderate: {i ∈ issues | IsReferenceIssue(i) ∨ IsMismatchIssue(i)},
    minor: issues \ (critical ∪ moderate)
}

DetermineNextAction: Classification → Action

DetermineNextAction(classification) = {
    "regenerate_code",          if |classification.critical| > 0
    "modify_visualization",     if |classification.moderate| > 0 ∧ |classification.critical| = 0
    "fine_tune_parameters",     if |classification.minor| > 0 ∧ |classification.moderate| = 0
    "complete",                 otherwise
}

其中:
- EvalResult = {matches_requirements: Boolean, automatic_failure_triggered: Boolean, 
               failure_reasons: List, unmet_requirements: List}
- Issue: 问题描述字符串
- IsBlankIssue(i) ≡ "blank" ∈ lowercase(i) ∨ "empty" ∈ lowercase(i)
- IsReferenceIssue(i) ≡ "reference" ∈ lowercase(i)
- IsMismatchIssue(i) ≡ "mismatch" ∈ lowercase(i)
```
```

---

## RC-Rule: ReAct控制规则

### RC-Rule-1: 迭代控制规则
**功能**: 控制ReAct模式的执行边界和终止条件

**形式化定义**:
```
IterationControl: Iteration × MaxIteration × Response → ControlAction

IterationControl(i, max_i, response) = {
    max_iterations_reached,     if i ≥ max_i
    final_answer_found,         if DetectFinalAnswer(response)
    too_many_errors,           if ConsecutiveErrors(response) ≥ 3
    continue,                  otherwise
}

其中:
- DetectFinalAnswer(·): 检测最终答案的谓词
- ConsecutiveErrors(·): 连续错误计数函数
```

### RC-Rule-2: 响应格式验证规则
**功能**: 验证Agent响应的结构正确性

**形式化定义**:
```
ResponseValidation: Response → Boolean × ErrorMessage

ResponseValidation(response) = 
    let blocks = ActiveBlocks(response) in
    if |blocks| = 1 then
        if "Action" ∈ blocks then
            (ValidJSON(ExtractActionContent(response)), "JSON格式检查")
        else
            (true, "格式正确")
    else
        (false, "必须包含且仅包含一个块类型")

其中:
- ActiveBlocks(·): 提取响应中活跃的块类型
- ExtractActionContent(·): 提取Action块的内容
```

### RC-Rule-3: 模型选择规则
**功能**: 根据内容类型选择合适的模型

**形式化定义**:
```
ModelSelection: Content × ImageURLs → ModelType

ModelSelection(content, img_urls) = {
    vision_model,   if img_urls ≠ ∅ ∨ HasImageContent(content)
    text_model,     otherwise
}

其中:
- HasImageContent(·): 检测内容是否包含图像的谓词
- ModelType = {vision_model, text_model}
```

### RC-Rule-4: 迭代反馈控制规则
**功能**: 控制迭代过程中的反馈和指导

**形式化定义**:
```
FeedbackControl: Iteration × MaxIteration × Response × ToolCallStatus → Feedback

FeedbackControl(i, max_i, response, has_tools) = {
    FinalAnswerPrompt,          if i > max_i/2
    FormatGuidance,             if "<Action>" ∈ response ∧ ¬has_tools
    ContinuePrompt,             otherwise
}

其中:
- FinalAnswerPrompt: 要求提供最终答案的提示
- FormatGuidance: JSON格式指导
- ContinuePrompt: 继续推理的提示
```

---

## 规则应用流程

### 主要执行路径
```
Execution: Input → Output

Execution(input) = 
    let task_type = TC-Rule-1(input) in
    let state = TC-Rule-6.initialize(input) in
    let pre_msgs = TC-Rule-5(task_type, state) in
    let workflow = TC-Rule-2(task_type) in
    ReactExecution(pre_msgs, workflow, state)

ReactExecution(msgs, workflow, state) =
    while ¬TerminationCondition(state) do
        response ← LLM(msgs)
        if RC-Rule-2(response) then
            tool_calls ← ParseToolCalls(response)
            results ← ExecuteTools(tool_calls, state)
            msgs ← msgs ∪ {response, FormatObservation(results)}
            state ← UpdateState(state, results)
        else
            msgs ← msgs ∪ {response, RC-Rule-4.feedback(response)}
```
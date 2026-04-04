# Agent Rule代码实现位置映射

本文档详细列出了`agent_rule.md`中定义的每个规则在代码中的具体实现位置。

---

## TC-Rule: 任务协调规则

### TC-Rule-1: 任务类型分类规则
**文档定义**: `agent_rule.md` 第32-53行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **方法**: `_determine_task_type()` 
- **行号**: 第645-679行

**实现代码**:
```python
def _determine_task_type(self, user_query: str, db_path: str, 
                         reference_path: str = None, 
                         existing_code_path: str = None) -> str:
    if existing_code_path:
        task_type = "D"  # 对应规则: if c ≠ ∅ ∧ exists(c)
    elif reference_path:
        if reference_path.lower().endswith(('.png', '.jpg', '.jpeg', '.gif', '.bmp')):
            task_type = "B"  # 对应规则: if r ≠ ∅ ∧ ext(r) ∈ {.png, ...}
        elif reference_path.lower().endswith(('.py', '.ipynb')):
            task_type = "C"  # 对应规则: if r ≠ ∅ ∧ ext(r) ∈ {.py, .ipynb}
        else:
            task_type = "A"
    else:
        task_type = "A"  # 对应规则: otherwise
    return task_type
```

---

### TC-Rule-2: 工作流程推荐规则
**文档定义**: `agent_rule.md` 第55-73行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **方法**: `_build_task_prompt()` 和 `process_task()` 中的逻辑
- **行号**: 
  - 工作流程描述: 第575-643行（`_build_task_prompt`）
  - 预迭代执行: 第520-532行（`process_task`）

**实现说明**: 
- 类型D: 第609-622行，描述`eval → modify → eval`流程
- 类型A/B/C: 第624-641行，描述`sql_gen → code_gen → eval → modify`流程

---

### TC-Rule-3: 工具前置条件验证规则
**文档定义**: `agent_rule.md` 第75-94行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **方法**: 各个工具函数的验证逻辑
- **行号**:
  - `_generate_sql_from_query_tool()`: 第251-253行（验证user_query和db_path）
  - `_generate_visualization_code_tool()`: 第275-281行（验证db_path, user_query, sql_query）
  - `_modify_visualization_code_tool()`: 第309-316行（验证visualization_code和evaluation_result）
  - `_evaluate_visualization_tool()`: 第345-349行（验证user_query和visualization_code）

**实现代码示例**:
```python
# _generate_visualization_code_tool() - 第275-281行
if not self.db_path:
    return {"status": False, "message": "Database path not specified"}
if not self.user_query:
    return {"status": False, "message": "User query is empty"}
if not self.sql_query:
    return {"status": False, "message": "SQL query not generated yet"}
```

---

### TC-Rule-4: 评估响应决策规则
**文档定义**: `agent_rule.md` 第96-113行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **方法**: `_evaluate_visualization_tool()`
- **行号**: 第367-398行

**实现代码**:
```python
# 第367-398行
if status:  # result.passed = true
    return {
        "complete": True  # 对应规则: complete
    }
else:
    if recommendations_count > 0:  # recommendations ≠ ∅
        next_action = "modify_visualization_code"  # 对应规则: modify_code
    else:
        next_action = "retry_or_restart"  # 对应规则: retry_or_restart
```

---

### TC-Rule-5: 预迭代控制规则
**文档定义**: `agent_rule.md` 第115-132行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **方法**: `process_task()`
- **行号**: 第517-553行

**实现代码**:
```python
# 第520-532行：确定第一个动作
if self.task_type == "D":
    first_action_tool_name = "evaluate_visualization"  # 对应规则: if t = D
else:
    first_action_tool_name = "generate_sql_from_query"  # 对应规则: otherwise

# 第535-552行：构建assistant_msg和observation_msg
assistant_content = f"<Thought>\n{first_action_thought}\n</Thought>\n<Action>..."
first_observation_result = first_action_func()
observation_content = f"<Observation>\n{json.dumps(...)}\n</Observation>"
```

---

### TC-Rule-6: 状态转换验证规则
**文档定义**: `agent_rule.md` 第134-150行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **说明**: 该规则主要通过系统提示词和工具前置条件验证间接实现，没有显式的状态转换验证函数
- **相关代码**: 
  - 系统提示词中的工作流程约束（第37-65行）
  - 工具函数中的前置条件验证（TC-Rule-3实现）

**注释**: 状态转换验证主要依赖LLM遵循系统提示词中的工作流程规则，而非硬编码的状态机验证。

---

## TE-Rule: 工具执行规则

### TE-Rule-1: 参数边界约束规则
**文档定义**: `agent_rule.md` 第156-170行

**代码实现位置**: 
- **文件**: `vis_system/database_query_agent.py`
- **方法**: 
  - `_get_table_tool()`: 第219-220行（限制sample_size ≤ 5）
  - `_execute_sql_tool()`: 第390-391行（限制max_rows ≤ 20）

**实现代码**:
```python
# _get_table_tool() - 第219-220行
if sample_size > 5:
    sample_size = 5  # 对应规则: min(value, 5) if param = sample_size

# _execute_sql_tool() - 第390-391行
if max_rows > 20:
    max_rows = 20  # 对应规则: min(value, 20) if param = max_rows
```

---

### TE-Rule-2: 工具参数验证规则
**文档定义**: `agent_rule.md` 第172-188行

**代码实现位置**: 
- **文件**: `vis_system/utils/ToolManager.py`
- **说明**: 工具注册时定义必需参数（第22行，required参数）
- **文件**: `vis_system/utils/Agent.py`
- **方法**: `_run_react_iterations()`中的工具调用验证
- **行号**: 第908-934行

**实现说明**: 
- 工具参数验证通过ToolManager在注册时定义必需参数
- 执行时通过检查参数存在性和JSON解析进行验证

---

### TE-Rule-3: 代码执行环境规则
**文档定义**: `agent_rule.md` 第190-209行

**代码实现位置**: 
- **文件**: `vis_system/code_generation_agent.py`
- **方法**: `_execute_altair_code()`
- **行号**: 
  - 图表变量提取: 第263-295行
  - 保存命令注入: 第298-303行

**实现代码**:
```python
# 图表变量提取 - 第267-295行
chart_assignments = re.findall(r'(\w+)\s*=\s*alt\.Chart', modified_code)  # 对应规则: chart_extraction
last_chart_var = chart_assignments[-1] if chart_assignments else None

# 保存命令注入 - 第298-303行
if last_chart_var:
    if ".save('" not in modified_code:  # 对应规则: ¬HasSave(code)
        save_code = f"\n{last_chart_var}.save('{output_path}')\n"  # 对应规则: InjectSave
        modified_code += save_code
```

**相关实现**:
- `vis_system/validation_evaluation_agent.py`: 第144-162行（类似逻辑）
- `vis_system/database_query_agent.py`: 第590-601行（类似逻辑）

---

## EH-Rule: 错误处理规则

### EH-Rule-1: 工具调用解析错误规则
**文档定义**: `agent_rule.md` 第215-234行

**代码实现位置**: 
- **文件**: `vis_system/utils/Agent.py`
- **方法**: `_parse_tool_calls_from_text()`
- **行号**: 第187-323行

**实现代码**:
```python
# 第204-215行：null_input检查
if text is None:
    error_msg = "输入文本为None"  # 对应规则: null_input

# 第287-300行：missing_end_tag检查
elif "<Action>" in text:
    error_msg = "发现起始标签<Action>但缺少结束标签</Action>"  # 对应规则: missing_end_tag

# 第234-251行：json_error检查
except json.JSONDecodeError as e:
    error_msg = f"JSON解析错误: {str(e)}"  # 对应规则: json_error

# 第338-377行：missing_fields检查
if "tool_name" not in tool_call or "parameters" not in tool_call:
    error_msg = f"Missing tool_name or parameters"  # 对应规则: missing_fields
```

---

### EH-Rule-2: 代码执行错误恢复规则
**文档定义**: `agent_rule.md` 第236-249行

**代码实现位置**: 
- **文件**: `vis_system/code_generation_agent.py`
- **方法**: `_execute_altair_code()` 和 `_execute_matplotlib_code()`
- **行号**: 
  - Altair执行: 第148-237行
  - Matplotlib执行: 类似位置

**实现说明**: 
- 错误捕获和分类通过try-except块实现
- 错误信息格式化并返回给LLM用于后续修正

---

### EH-Rule-3: 评估失败处理规则
**文档定义**: `agent_rule.md` 第251-303行

**代码实现位置**: 
- **文件**: `vis_system/validation_evaluation_agent.py`
- **方法**: `evaluate_visualization()`
- **行号**: 第282-840行（特别是第485-495行的force_failure处理）

**实现代码**:
```python
# 第485-492行：ForcedFailureRecovery
if force_failure and matches_requirements:
    matches_requirements = False
    evaluation_result["automatic_failure_triggered"] = True
    evaluation_result["failure_reasons"].append("Forced failure mode activated")
    # 对应规则: ForcedFailureRecovery
```

**相关实现**:
- `vis_system/coordinator_agent.py`: 第367-398行（根据评估结果决定下一步动作）

---

## RC-Rule: ReAct控制规则

### RC-Rule-1: 迭代控制规则
**文档定义**: `agent_rule.md` 第309-326行

**代码实现位置**: 
- **文件**: `vis_system/utils/Agent.py`
- **方法**: `_run_react_iterations()`
- **行号**: 第796-866行

**实现代码**:
```python
# 第796行：迭代控制循环
while final_answer is None and current_iteration < max_iterations:  # 对应规则: if i ≥ max_i
    current_iteration += 1
    # ...
    
    # 第857-866行：final_answer_found检测
    if "<Final_Answer>" in assistant_content and "</Final_Answer>" in assistant_content:
        final_answer = assistant_content[start_pos:end_pos].strip()
        break  # 对应规则: if DetectFinalAnswer(response)
```

---

### RC-Rule-2: 响应格式验证规则
**文档定义**: `agent_rule.md` 第328-348行

**代码实现位置**: 
- **文件**: `vis_system/utils/Agent.py`
- **方法**: 
  - `_parse_tool_calls_from_text()`: 第187-323行（解析和验证）
  - `_run_react_iterations()`: 第848-873行（格式检查和修复）

**实现代码**:
```python
# 第848-873行：响应格式验证和修复
if "<Action>" in assistant_content and "</Action>" in assistant_content:
    assistant_content = assistant_content.split("</Action>")[0].strip() + "\n</Action>"
    
# 第869-872行：缺失标签修复
if "<Action>" in assistant_content and "</Action>" not in assistant_content:
    assistant_content += "\n</Action>"

# 第873行：解析验证
tool_calls = self._parse_tool_calls_from_text(assistant_content)
```

---

### RC-Rule-3: 模型选择规则
**文档定义**: `agent_rule.md` 第350-365行

**代码实现位置**: 
- **文件**: `vis_system/utils/Agent.py`
- **方法**: `_prepare_messages()`
- **行号**: 第166-176行

**实现代码**:
```python
# 第166-176行：模型选择逻辑
if img_urls:  # 对应规则: if img_urls ≠ ∅
    user_content = [{"type": "text", "text": prompt}]
    for img_url in img_urls or []:
        user_content.append({"type": "image_url", "image_url": {"url": img_url}})
    # 使用vision_model（通过img_client）
else:
    # 使用text_model（通过text_client）
```

**相关实现**:
- `vis_system/utils/Agent.py`: `call_llm()`方法根据消息内容选择客户端（text_client或img_client）

---

### RC-Rule-4: 迭代反馈控制规则
**文档定义**: `agent_rule.md` 第367-384行

**代码实现位置**: 
- **文件**: `vis_system/utils/Agent.py`
- **方法**: `_build_react_system_prompt()` 和 `_run_react_iterations()`
- **行号**: 
  - 系统提示词构建: 第706-766行
  - 迭代过程中的反馈: 第796-1020行

**实现说明**: 
- 反馈控制主要通过系统提示词中的指导实现
- 在迭代过程中根据响应内容动态添加反馈信息

---

## 规则应用流程

**文档定义**: `agent_rule.md` 第388-411行

**代码实现位置**: 
- **文件**: `vis_system/coordinator_agent.py`
- **方法**: `process_task()`
- **行号**: 第472-573行

**实现流程**:
```python
# 第511-515行：TC-Rule-1（任务类型分类）
self.task_type = self._determine_task_type(...)

# 第515行：TC-Rule-2（工作流程推荐，通过提示词体现）
initial_prompt = self._build_task_prompt(max_iterations)

# 第517-553行：TC-Rule-5（预迭代控制）
user_messages = [...]
# 执行预迭代步骤

# 第559行：ReAct执行
result, used_tool = self.chat_ReAct(user_messages=user_messages, max_iterations=max_iterations)
```

---

## 总结

所有规则实现分散在以下主要文件中：

1. **`vis_system/coordinator_agent.py`** - TC-Rule的所有规则实现
2. **`vis_system/utils/Agent.py`** - RC-Rule和EH-Rule-1的实现
3. **`vis_system/database_query_agent.py`** - TE-Rule-1的实现
4. **`vis_system/code_generation_agent.py`** - TE-Rule-3和EH-Rule-2的实现
5. **`vis_system/validation_evaluation_agent.py`** - EH-Rule-3的实现
6. **`vis_system/utils/ToolManager.py`** - TE-Rule-2的辅助实现

这些规则的实现主要通过方法级别的逻辑实现，而非集中在一个规则引擎中。


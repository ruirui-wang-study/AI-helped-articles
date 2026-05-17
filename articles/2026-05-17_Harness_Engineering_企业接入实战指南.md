# Harness Engineering 企业接入实战指南：从 OpenAI 到海致，大厂都在用的 AI 驾驭方法论

> 文章来源：来财 AI 学习笔记  
> 适合读者：技术负责人、架构师、正在尝试将 AI Agent 引入生产环境的开发者  
> 阅读时间：约 8 分钟

---

## 什么是 Harness Engineering？

一句话讲清楚：**Agent = 模型 + 缰绳。模型是马，Harness 就是缰绳、马鞍和马蹄铁。**

2026 年 2 月，HashiCorp 联合创始人 Mitchell Hashimoto 写下了一段话："每当发现 Agent 犯了一个错误，就花时间工程化一个解决方案，让它永远不再犯同样的错误。"他把这个思路叫做 **Harness Engineering**。

六天后，OpenAI 发布了一篇炸裂的实践报告——3 名工程师，5 个月，0 行手写代码，产出了 100 万行生产级代码，合并了 1500 个 PR。效率约为人工的 10 倍。但他们说重点不是这个成绩，而是成绩背后的东西——一套精心设计的约束系统。

Martin Fowler 在 ThoughtWorks 技术博客深度分析，LangChain、Anthropic 先后跟进。一个月内，这个新范式席卷全球。

Harness Engineering 回答的根本问题是：**怎么设计一套系统，让 AI Agent 能稳定、可控、不抽风地干活？**

它不是 Prompt Engineering 那种"怎么跟模型说话"的技巧，而是更像搭赛道——有护栏、有标识、有维修站，让赛车手可以放心踩油门。

大模型竞争的决胜点已经变了。2025 年大家比谁的模型更强，2026 年比的是**谁能把同一个模型用得更好**。

---

## 大厂都在怎么干？五个真实案例

### 1. OpenAI — 100 万行代码的"驯马术"

2025 年 8 月，OpenAI 的 Codex 团队做了一个极端实验：三个人，不开 IDE，所有代码全部由 Agent 生成。从空仓库开始，到百万行级别的生产应用。

**层级依赖模型。** 团队给代码库建立了一套严格的依赖规则——Types 层不能依赖 Runtime，Config 层不能依赖 Service。然后把这些规则写成 Linter，直接在 CI 里卡死。

```python
# 一个 Harness Linter 的简化示例
# 检查 AI 生成的代码是否违反了层级依赖规则

def check_layer_violation(code_block: str) -> list[str]:
    """检查代码中是否存在下层反向依赖上层的情况"""
    violations = []
    layer_map = {
        "types": 0, "config": 1, "repo": 2,
        "service": 3, "runtime": 4, "ui": 5
    }
    
    for line in code_block.split("\n"):
        # 检测 import 语句
        if line.startswith("import") or line.startswith("from"):
            for dep in layer_map:
                if dep in line and dep != current_layer:
                    if layer_map[dep] < layer_map[current_layer]:
                        violations.append(
                            f"违规: {current_layer} 层不能依赖 {dep} 层"
                            f"\n  {line.strip()}"
                            f"\n  正确做法: 通过接口层间接调用或重构依赖关系"
                        )
    return violations
```

**"三明治"推理策略。** 他们发现全程用最高推理强度反而得分更低（53.9%），因为超时了。最终方案是：规划和验证阶段用高强度，中间写代码阶段降一档。分数飙到 63.6%。

```python
# "三明治"推理预算分配示例
class SandwichInferenceStrategy:
    """控制 Agent 在不同阶段的推理强度"""
    
    def get_reasoning_level(self, phase: str) -> str:
        strategy = {
            "plan": "xhigh",      # 规划阶段：最高强度
            "implement": "high",  # 编码阶段：中高强度
            "review": "xhigh",    # 审查阶段：最高强度
            "fix": "high",        # 修复阶段：中高强度
        }
        return strategy.get(phase, "high")
    
    def estimate_budget(self, task_size: int) -> dict:
        """估算每个阶段的 token 预算"""
        return {
            "plan_budget": task_size * 0.2,    # 20% 用于规划
            "impl_budget": task_size * 0.5,    # 50% 用于实现
            "review_budget": task_size * 0.3,  # 30% 用于审查
        }
```

**垃圾收集日。** 每天下午 4 到 5 点，工程师不写新功能，专门审视 Agent 当天生成的代码，把坏模式写成自动检查规则。

**开源参考：** 这类 Linter 可以参考 [ESLint](https://eslint.org/) 的插件机制或 [Semgrep](https://semgrep.dev/) 的模式匹配——都是现成的 Harness 基础设施。

### 2. Anthropic — 金鱼的记忆只有七秒，Agent 的更短

Anthropic 想让 Claude 从零开发一个完整的 Web 应用。跑着跑着发现一个问题：Agent 干活超过一定时间后，智商直线下降。

原因是上下文窗口满了。Anthropic 的解决方案很颠覆：**不是压缩记忆，是直接换条新金鱼。** 这叫 Context Reset。

```python
# Context Reset 的简化实现
class AgentHandover:
    """Agent 交接机制：把状态传给新 Agent"""
    
    def create_handover_doc(self, state: dict) -> str:
        """生成结构化交接文档"""
        return f"""
# Agent 交接报告
## 完成状态
- 已完成: {', '.join(state['completed'])}
- 进行中: {state['in_progress']}
- 阻塞项: {', '.join(state['blocked'])}
        
## 上下文快照
- 最新构建状态: {state['build_status']}
- 未通过的测试: {', '.join(state['failed_tests'])}
- 当前分支: {state['branch']}
        
## 下一步任务
1. {state['next_task']['description']}
   - 优先级: {state['next_task']['priority']}
   - 参考文件: {state['next_task']['references']}
   - 风险提示: {state['next_task']['risks']}
        
## 已知约束
{chr(10).join(f'- {c}' for c in state['constraints'])}
"""
    
    def should_reset(self, context: dict) -> bool:
        """判断是否该触发上下文重置"""
        total_tokens = context["input_tokens"] + context["output_tokens"]
        return total_tokens > context["max_window"] * 0.7  # 用到 70% 就触发
```

Anthropic 还用 Git 锁文件做并发隔离——多个 Worker 不能同时改同一段代码。

```python
# 基于 Git 锁的 Agent 并发隔离（Anthropic 方案简化版）
import os, hashlib

class GitLockHarness:
    """用 Git 锁文件实现 Agent 并发隔离"""
    
    def __init__(self, repo_path: str):
        self.lock_dir = os.path.join(repo_path, ".git", "agent-locks")
        os.makedirs(self.lock_dir, exist_ok=True)
    
    def acquire(self, file_path: str, agent_id: str) -> bool:
        """Agent 申请修改某个文件的锁"""
        lock_name = hashlib.md5(file_path.encode()).hexdigest()
        lock_file = os.path.join(self.lock_dir, lock_name)
        
        # 用 os.open 的原子操作防止竞态
        try:
            fd = os.open(lock_file, os.O_CREAT | os.O_EXCL | os.O_WRONLY)
            with os.fdopen(fd, 'w') as f:
                f.write(f"agent: {agent_id}\nacquired: {__import__('time').time()}")
            return True
        except FileExistsError:
            return False  # 被别人锁了
    
    def release(self, file_path: str):
        lock_name = hashlib.md5(file_path.encode()).hexdigest()
        lock_file = os.path.join(self.lock_dir, lock_name)
        if os.path.exists(lock_file):
            os.remove(lock_file)
```

### 3. LangChain — 不改模型，排名从 30 跳到 5

LangChain 在 Terminal Bench 2.0 上做了一个控制变量实验：模型固定为 gpt-5.2-codex，**只改 Harness**，得分从 52.8% 升到 66.5%，排名从第 30 跃升至第 5。

```python
# LangChain 式的中间件 Harness（简化版）
from typing import Callable, Any

class PreCompletionChecklistMiddleware:
    """Agent 完成任务前的强制检查清单"""
    
    def __init__(self, requirements: list[str]):
        self.requirements = requirements
    
    def intercept(self, agent_output: str, task: str) -> bool:
        """拦截 Agent 的输出，检查是否满足所有需求"""
        for req in self.requirements:
            if req not in agent_output:
                return False  # 不满足，阻止完成
        return True

class LoopDetectionMiddleware:
    """循环检测：发现 Agent 在反复改同一个文件"""
    
    def __init__(self, max_edits: int = 3):
        self.file_edit_count = {}
        self.max_edits = max_edits
    
    def on_file_edit(self, file_path: str) -> str | None:
        """每次编辑文件时调用，返回 None 或提示信息"""
        self.file_edit_count[file_path] = \
            self.file_edit_count.get(file_path, 0) + 1
        
        if self.file_edit_count[file_path] > self.max_edits:
            return (
                f"⚠️ 检测到重复编辑 ({file_path} 已修改 "
                f"{self.file_edit_count[file_path]} 次，"
                f"测试仍未通过)。建议回滚后重新审视需求。"
            )
        return None

class LocalContextMiddleware:
    """自动注入工作环境信息，帮 Agent 省探索时间"""
    
    def enhance_prompt(self, base_prompt: str, project_info: dict) -> str:
        return f"""
{base_prompt}

## 工作环境（自动注入）
- 项目路径: {project_info['root']}
- 主要目录: {', '.join(project_info['dirs'])}
- Python 版本: {project_info['python_version']}
- 可用工具: {', '.join(project_info['tools'])}
- 关键配置: {project_info.get('config_summary', '无')}
"""
```

**开源参考：** LangChain 的中间件系统完全开源，可以直接用 `langchain.middleware` 模块。LangGraph 则是它的 Agent 编排引擎，支持状态机和条件分支。

### 4. Cursor — 20 个 Agent 一起干活，效率还不如 3 个

Cursor 的实验发现：20 个 Agent 同时上线，有效吞吐量只相当于 2-3 个。抢不到核心代码的 Agent 开始改注释、调空格——假装自己在工作。

```python
# "规划者-执行者"门控模式示例
class CoordinatorAgent:
    """主 Agent（工头）：负责任务分配和验收"""
    
    def assign_task(self, worker_id: str, task: dict) -> str:
        """分配任务，返回许可令牌"""
        token = f"permit_{worker_id}_{task['id']}_{__import__('time').time()}"
        return token
    
    def approve_action(self, worker_id: str, action: dict, token: str) -> bool:
        """Worker 执行危险操作前必须申请许可"""
        if not token or worker_id not in token:
            return False
        return True

class WorkerAgent:
    """执行 Agent（小弟）：只能干活，要许可才能干危险事"""
    
    def __init__(self, coordinator: CoordinatorAgent, worker_id: str):
        self.coordinator = coordinator
        self.worker_id = worker_id
        self.current_token = None
    
    def request_permission(self, action: dict, task: dict):
        self.current_token = self.coordinator.assign_task(
            self.worker_id, task
        )
```

### 5. 海致科技 — 当 Harness 遇上 B 端

国内的情况更复杂。B 端企业的 IT 环境是几十年的"技术债博物馆"：ERP、CRM、OA、财务系统、自研中台……跨系统对接占总项目 60% 以上。

海致的解法分三层：知识层（企业知识图谱 + 动态检索降幻觉）、编排层（异构系统封装为标准化工具）、安全层（全程可追溯可审计）。

---

## 具体怎么搭缰绳？三步走，带代码

### 第一步：把文档从"百科全书"变成"地图"（1-2 天）

选一个 Agent 最近翻车的场景，从它的报错日志反推——它缺哪份文档？

```python
# 自动化文档索引生成（让 Agent 按需检索）
DOCS_INDEX = {
    "架构规范": {
        "path": "docs/architecture.md",
        "updated": "2026-05-15",
        "tags": ["依赖规则", "分层", "模块边界"],
        "when_to_read": "新增模块或修改核心依赖关系时",
    },
    "API 规范": {
        "path": "docs/api-standards.md", 
        "updated": "2026-05-10",
        "tags": ["REST", "命名", "错误码"],
        "when_to_read": "新建或修改 API 接口时",
    },
    "数据库规范": {
        "path": "docs/db-standards.md",
        "updated": "2026-05-01",
        "tags": ["表结构", "索引", "迁移"],
        "when_to_read": "修改数据库 schema 时",
    },
}
```

### 第二步：把"不准 XXX"变成硬规矩（3-5 天）

口头约定对 Agent 无效。选三到五条规则，写成硬性检查：

```python
# 自定义 Linter 规则（Harness 的核心）
# 可以在 CI 里用 ESLint/Semgrep 或自定义脚本跑

def harness_check(code: str, file_path: str) -> list[dict]:
    """Harness 检查器：返回所有违规项"""
    violations = []
    
    # 规则1: 不允许 magic number
    if re.search(r'(?<!= )\d{4,}(?![.\d])', code) and "config" not in file_path:
        violations.append({
            "rule": "no-magic-numbers",
            "severity": "error",
            "message": "发现魔法数字，请提取为配置常量",
            "fix_hint": "将数字抽取到 config.py 或常量文件",
        })
    
    # 规则2: 不允许直接 print
    if "print(" in code and "test" not in file_path:
        violations.append({
            "rule": "no-print", 
            "severity": "warning",
            "message": "生产代码不要用 print，请用 logger",
            "fix_hint": "将 print() 替换为 logging.info()",
        })
    
    # 规则3: 函数不能超过 50 行
    for func_match in re.finditer(r'def \w+\(.*\):.*?(?=\ndef |\Z)', code, re.DOTALL):
        lines = func_match.group().count('\n')
        if lines > 50:
            violations.append({
                "rule": "max-function-length",
                "severity": "error",
                "message": f"函数超过 50 行（当前 {lines} 行），请拆分",
                "fix_hint": "按单一职责原则拆分为多个小函数",
            })
    
    return violations
```

关键细节：报错信息要包含"为什么违规 + 怎么改"，这样 Agent 能自己修，不需要人介入。

### 第三步：让系统自己"长记性"（1-2 周）

建立父-子 Agent 架构，每翻一次车就加固一道：

```python
class HarnessSystem:
    """完整的 Harness 系统：自动记录错误并加固"""
    
    def __init__(self):
        self.error_log = []
        self.rules = []
    
    def record_failure(self, agent_id: str, task: str, error: str):
        """Agent 翻车时记录"""
        entry = {
            "agent": agent_id,
            "task": task,
            "error": error,
            "time": __import__('time').time(),
            "fixed": False,
        }
        self.error_log.append(entry)
        
        # 如果同一个错误出现两次，自动生成规则
        similar = [e for e in self.error_log 
                   if e["error"] == error and e["agent"] != agent_id]
        if len(similar) >= 1:
            self._auto_generate_rule(error)
    
    def _auto_generate_rule(self, error: str):
        """从错误中自动生成 Harness 规则"""
        rule = {
            "type": "linter" if "code" in error else "checklist",
            "pattern": error,
            "created": __import__('time').time(),
            "enabled": True,
        }
        self.rules.append(rule)
        print(f"🛡️ 自动生成规则 #{len(self.rules)}: {error[:50]}...")
```

---

## 开源框架推荐

光看理论还不够，下面这些开源项目可以直接拿来做你的 Harness 基础设施：

| 框架 | 用途 | GitHub |
|------|------|--------|
| **LangChain / LangGraph** | Agent 编排 + 状态机 + 中间件 | github.com/langchain-ai/langchain |
| **OpenAI Agents SDK** | Function Calling + Tool 定义 | github.com/openai/openai-agents-python |
| **Semgrep** | 代码模式匹配 Linter | github.com/semgrep/semgrep |
| **MCP (Model Context Protocol)** | Agent 接入外部工具的标准协议 | github.com/modelcontextprotocol |
| **Dify** | 低代码 AI 应用平台 | github.com/langgenius/dify |
| **vLLM** | 推理加速（量化 / PagedAttention） | github.com/vllm-project/vllm |
| **RAGAS** | RAG 系统评估框架 | github.com/explodinggradients/ragas |

这些项目都是开源的，可以直接 fork 下来看源码，很多 Harness 思想都植根其中。

---

## 最后说两句人话

> "为了获得更高的 AI 自主性，运行时必须受到更严格的约束。增加信任需要的不是更多自由，而是更多限制。"  
> — Birgitta Böckeler, ThoughtWorks

想想高速路的护栏——正因为有它们，你才敢开到 120。

2026 年的工程师正在经历一次角色转移：以前你问"怎么写出更好的代码"，现在你问"怎么设计一个让 Agent 稳定干活的环境"。

明天上班做什么？选一个 Agent 最近翻车的场景，写一条规则堵上。今天就干。

---

## 参考与延伸阅读

- OpenAI. (2026). Harness engineering: leveraging Codex in an agent-first world.
- Anthropic. (2025). Effective harnesses for long-running agents.
- Anthropic. (2025). Effective context engineering for AI agents.
- LangChain. (2026). The Anatomy of an Agent Harness.
- Mitchell Hashimoto. (2026). Harness Engineering (Personal Blog).
- Cursor. (2026). Scaling long-running autonomous coding.
- 海致科技. (2026). Harness 时代：海致科技如何定义 AI 的"驾驭工程".
- Martin Fowler / ThoughtWorks. (2026). Harness Engineering Analysis.

---
title: LangGraph 实操入门 — 从零到能跑的多 Agent
created: 2026-06-02
updated: 2026-11-10
type: methodology
tags: [agent, methodology]
status: stable
sources: [https://docs.langchain.com/oss/python/langgraph/interrupts, https://docs.langchain.com/oss/python/langgraph/checkpointers, https://docs.langchain.com/oss/python/langgraph/persistence, https://reference.langchain.com/python/langgraph/checkpoints/]
---

# LangGraph 实操入门 — 从零到能跑的多 Agent

> 配套概念页：[[10-知识库/AI模型与Agent/04-智能体架构与工作流设计/2026-06-00-Agent编排入门-由浅入深.md|Agent 编排入门（由浅入深）]]。
> 本文先用 mock 节点验证图骨架，不把控制流通过等同于模型质量通过。2026-11-10 按官方文档补充动态审批与跨进程恢复；复现固定使用 `langgraph==1.2.2`，不是“当前最新版”声明。真实 LLM、外部写接口和生产数据库不在本轮实测范围。

---

## 为什么第一个框架学 LangGraph

- 本页需要循环、显式状态、暂停与恢复，LangGraph 可以把这些控制点直接表示为节点和边
- 能与 LangChain 组件、LangSmith 观测集成，但教程骨架不依赖在线 LangSmith 服务
- 心智模型清晰：**Node 干活，Edge 决定下一步去哪**

代价：需要理解 reducer、checkpoint 与重放边界。只有固定顺序的两三个函数时，普通 Python 编排更简单；需要显式恢复点时再引入图，不能因为“多 Agent”就默认需要框架。图能表达暂停，不自动提供审批人身份校验、业务幂等或生产运维。

---

## 第 1 步：环境搭建

```bash
uv venv lg-venv --python python3
source lg-venv/bin/activate          # Windows 按所用 shell 激活
uv pip install 'langgraph==1.2.2' 'langgraph-checkpoint-sqlite==3.1.3'
# 用真实 LLM 时再装：uv pip install langchain-openai
```

> 只学图骨架不需要任何 API key——这正是 LangGraph 适合初学者的地方：**先把控制流跑通，再接模型**。

---

## 第 2 步：理解三个核心概念

1. **State（状态）** — 一个 `TypedDict`，贯穿整个图，所有节点读写它。
   - 每个字段可配 **reducer** 控制"更新方式"：`operator.add` = 追加/累加（不覆盖）；默认 = 覆盖。
2. **Node（节点）** — 一个普通函数：收 state → 干活 → 返回要更新的字段（dict）。
3. **Edge（边）** — 决定下一步去哪。
   - 普通 edge：A 做完固定去 B
   - **conditional edge**：按运行时 state 动态决定去哪（实现分支/循环）

> 一句话：**Nodes do the work, edges tell what to do next.**

---

## 第 3 步：第一个多 Agent 图（已实测跑通）

场景：研究员 → 文案 → 审核，审核不通过就打回重写（**循环 + 条件分支**）。

```python
from typing import TypedDict, Annotated, Literal
import operator
from langgraph.graph import StateGraph, START, END

# 1) 共享状态:messages 用 reducer 追加,steps_done 累加
class State(TypedDict):
    task: str
    messages: Annotated[list[str], operator.add]
    steps_done: Annotated[int, operator.add]
    route: str

# 2) 三个专职 agent 节点(mock:演示控制流,不调真 LLM)
def researcher(state: State) -> dict:
    return {"messages": [f"[研究员] 查到关于 '{state['task']}' 的资料"], "steps_done": 1}

def writer(state: State) -> dict:
    return {"messages": ["[文案] 基于资料写了初稿"], "steps_done": 1}

def reviewer(state: State) -> dict:
    # 模拟:前两步不够分就打回,够了就通过(演示循环)
    if state["steps_done"] < 4:
        return {"messages": ["[审核] 不合格,打回重写"], "route": "rewrite", "steps_done": 1}
    return {"messages": ["[审核] 通过"], "route": "done", "steps_done": 1}

# 3) supervisor:conditional edge 的判定函数
def route_after_review(state: State) -> Literal["writer", "__end__"]:
    return "writer" if state["route"] == "rewrite" else END

# 4) 组图
g = StateGraph(State)
g.add_node("researcher", researcher)
g.add_node("writer", writer)
g.add_node("reviewer", reviewer)
g.add_edge(START, "researcher")
g.add_edge("researcher", "writer")
g.add_edge("writer", "reviewer")
g.add_conditional_edges("reviewer", route_after_review, {"writer": "writer", END: END})
app = g.compile()

# 5) 执行
result = app.invoke({"task": "AI Agent 编排框架", "messages": [], "steps_done": 0, "route": ""})
for m in result["messages"]:
    print(" ", m)
print(f"总步数: {result['steps_done']} | 最终路由: {result['route']}")
```

**本机实际输出：**
```
  [研究员] 查到关于 'AI Agent 编排框架' 的资料
  [文案] 基于资料写了初稿
  [审核] 不合格,打回重写
  [文案] 基于资料写了初稿
  [审核] 通过
总步数: 5 | 最终路由: done
```

→ 注意第 3-4 行：审核打回后**真的绕回去重写了**。这就是图范式相对旧 AgentExecutor 的核心优势——**循环和分支是你显式定义的，可控可复现**。

---

## 第 4 步：内存 checkpoint + 静态断点（调试示例）

先让流程跑到某步暂停，再恢复。靠两个东西：**checkpointer**（存状态）+ **interrupt_before**（静态断点）。下面验证同一进程内的恢复，不是重启持久化或生产审批。官方建议 HITL 使用动态 `interrupt()`，见后面的工程补充。

```python
from typing import TypedDict, Annotated
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

class State(TypedDict):
    counter: Annotated[int, operator.add]
    log: Annotated[list[str], operator.add]

def step_a(state): return {"counter": 1, "log": ["step_a 执行"]}
def step_b(state): return {"counter": 1, "log": ["step_b 执行"]}

g = StateGraph(State)
g.add_node("a", step_a); g.add_node("b", step_b)
g.add_edge(START, "a"); g.add_edge("a", "b"); g.add_edge("b", END)

# 关键:挂 checkpointer + 在 b 前设断点(模拟"等人工确认")
checkpointer = InMemorySaver()
app = g.compile(checkpointer=checkpointer, interrupt_before=["b"])

cfg = {"configurable": {"thread_id": "demo-1"}}   # thread_id 标识一次会话

# 第一次:跑到 b 之前停下
r1 = app.invoke({"counter": 0, "log": []}, cfg)
print("中断时:", r1["counter"], r1["log"])          # b 还没跑

snap = app.get_state(cfg)
print("下一步将执行:", snap.next)                    # 证明状态被存下来了

# 恢复:传 None = 从断点继续
r2 = app.invoke(None, cfg)
print("恢复后:", r2["counter"], r2["log"])
```

**本机实际输出：**
```
中断时: 1 ['step_a 执行']
下一步将执行: ('b',)
恢复后: 2 ['step_a 执行', 'step_b 执行']
```

→ 流程在 `b` 前停住（counter 还是 1），`get_state` 能读到下一步是 `b`，传 `None` 后继续。这里尚未采集或校验审批结果；只要调用恢复就会进入 `b`，所以不能把这个示例直接当成审批授权。

---

## 第 5 步：接真实 LLM（需 API key，本机未实跑）

可以先把 mock 节点换成模型调用，观察数据流。以下片段只生成回答，**没有联网检索工具**，提示词里写“查找”不代表真的查过资料；事实任务仍需显式检索节点、来源保存和证据校验：

```python
# uv pip install langchain-openai;  export OPENAI_API_KEY=sk-...
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")

def researcher(state: State) -> dict:
    resp = llm.invoke(f"查找关于 {state['task']} 的关键资料,列 3 条")
    return {"messages": [f"[研究员] {resp.content}"], "steps_done": 1}
```

> 返回契约保持一致时可复用图骨架。接真实模型还需处理结构化输出、超时、限流、工具错误和重试预算；仅设置兼容接口的 `base_url` 与 `api_key` 不保证所有特性一致。示例模型名沿用原稿，不代表当前账户必然有权限。

---

## 第 6 步：进阶方向（学完上面再看）

- **预制多 agent 库**：`langgraph-supervisor`（中心调度）、`langgraph-swarm`（无中心 handoff）——不用手写路由
- **Streaming**：`app.stream(...)` 流式看每步中间结果
- **LangSmith**：按官方配置启用 tracing，检查节点输入、输出、工具调用与时序；trace 不等于可读取模型隐藏思考。接入前核日志脱敏与上传边界
- **持久化升级**：`InMemorySaver` 换成 SQLite/Postgres checkpointer，状态落库

---

## 常见坑

1. **state 字段不加 reducer 会被覆盖**：想累积消息/列表，必须 `Annotated[list, operator.add]`，否则后一个节点的返回会冲掉前面的。
2. **conditional edge 的返回值要和映射字典对上**：`route_after_review` 返回的字符串/END 必须是 `add_conditional_edges` 第三个参数 dict 的 key。
3. **忘记 compile()**：`StateGraph` 定义完必须 `.compile()` 才能 invoke。
4. **可恢复 HITL 必须有 checkpointer**：动态 `interrupt()` 配合 `Command(resume=...)`；静态断点主要用于调试。`InMemorySaver` 进程退出后丢失状态。
5. **thread_id 要固定**：恢复执行时 config 的 `thread_id` 必须和中断时一致，否则找不到之前的状态。

---

## 关联阅读
- [[10-知识库/AI模型与Agent/04-智能体架构与工作流设计/2026-06-00-Agent编排入门-由浅入深.md|Agent 编排入门（由浅入深）]] — 先读这篇懂概念
- [[40-调研报告/AI技术与产品/Agent与研发流程/2026-06-Agent编排框架横评-SK-MAF-LangChain.md|Agent 编排框架横评]] — 横向对比 8 个框架
- [[10-知识库/AI模型与Agent/04-智能体架构与工作流设计/4.2.1-LangGraph深度实操.md|LangGraph 深度实操]] — 库内已有的进阶笔记

## 工程补充：审批不是“暂停一下再继续”

### 实例一：工单变更预览，批准后只生成模拟回执

把“生成变更草案”“等待审批”“执行变更”分开。审批界面展示工单、操作和版本摘要；后端把审批结果绑定到当前草案摘要，拒绝草案变更后沿用旧摘要。下面保存为 `approval_demo.py`，它不发邮件、不改数据库，只返回模拟回执，适合先验证控制流。

```python
from typing import TypedDict
import hashlib
import json
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt

class ApprovalState(TypedDict):
    change_id: str
    action: str
    revision: int
    approved: bool
    receipt: str

def plan_digest(state):
    fields = {k: state[k] for k in ("change_id", "action", "revision")}
    return hashlib.sha256(json.dumps(
        fields, ensure_ascii=False, sort_keys=True,
        separators=(",", ":")
    ).encode("utf-8")).hexdigest()

def build_approval(checkpointer, visits):
    def approval(state):
        visits.append(state["change_id"])  # 测试探针，不是业务写入
        digest = plan_digest(state)
        decision = interrupt({
            "change_id": state["change_id"], "action": state["action"],
            "revision": state["revision"], "digest": digest
        })
        ok = (isinstance(decision, dict)
              and decision.get("approved") is True
              and decision.get("digest") == digest)
        return {"approved": ok}

    def apply_preview(state):
        return {"receipt": "SIMULATED:" + state["change_id"]}

    g = StateGraph(ApprovalState)
    g.add_node("approval", approval)
    g.add_node("apply_preview", apply_preview)
    g.add_edge(START, "approval")
    g.add_conditional_edges("approval", lambda s: (
        "apply_preview" if s["approved"] else END
    ))
    g.add_edge("apply_preview", END)
    return g.compile(checkpointer=checkpointer)

if __name__ == "__main__":
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.types import Command
    visits = []
    app = build_approval(InMemorySaver(), visits)
    cfg = {"configurable": {"thread_id": "change-demo-1"}}
    state = {"change_id": "CHG-DEMO", "action": "预览配置变更",
             "revision": 1, "approved": False, "receipt": ""}
    pending = app.invoke(state, cfg)
    digest = pending["__interrupt__"][0].value["digest"]
    result = app.invoke(Command(resume={
        "approved": True, "digest": digest
    }), cfg)
    print(result["receipt"], "approval_entries=", len(visits))
```

关键不是回执内容，而是 `approval_entries`：动态恢复时，包含 `interrupt()` 的节点从开头重跑，因此它前面的探针再次执行。不要在这一位置直接发邮件、扣款或追加工单记录。也不要用宽泛的 `try/except` 吞掉中断，或在不同执行轮次改变多个 `interrupt()` 的相对顺序。

摘要校验只防止错批内容，不验证审批人身份。生产恢复接口还要核登录身份、租户、工单所有权、审批过期与撤销状态；`thread_id` 是状态定位符，不是访问权限。未通过验证时不能发 `Command(resume=...)`。这里采用严格布尔判断，字符串 `"true"` 不能作为同意。

### 实例二：开发机退出后，重新打开 SQLite 恢复

内存示例只能证明当前进程能恢复。下面保存为 `resume_demo.py`，与 `approval_demo.py` 放在同一目录，用不同进程读写一个临时 SQLite checkpoint 文件。

```python
import sys
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.types import Command
from approval_demo import build_approval

mode, db_path = sys.argv[1:]
cfg = {"configurable": {"thread_id": "sqlite-demo-1"}}
with SqliteSaver.from_conn_string(db_path) as saver:
    app = build_approval(saver, [])
    if mode == "start":
        result = app.invoke({
            "change_id": "CHG-SQLITE", "action": "预览发布步骤",
            "revision": 1, "approved": False, "receipt": ""
        }, cfg)
        assert result["__interrupt__"]
        print("paused", app.get_state(cfg).next)
    elif mode == "resume":
        snap = app.get_state(cfg)
        pending = [i for t in snap.tasks for i in t.interrupts]
        assert len(pending) == 1, "需要一个待处理审批"
        # 仅本地测试：模拟同意，绝不可充当真实审批服务。
        result = app.invoke(Command(resume={
            "approved": True, "digest": pending[0].value["digest"]
        }), cfg)
        print(result["receipt"], app.get_state(cfg).next)
    else:
        raise ValueError("mode 必须是 start 或 resume")
```

在一次性的测试目录中依次运行；每次实验换新文件，不对生产 checkpoint 试验：

```bash
python approval_demo.py
python resume_demo.py start ./demo-checkpoints.sqlite
python resume_demo.py resume ./demo-checkpoints.sqlite
```

两次命令分别启动进程。验证对象是数据库中的 pending state、恢复后的回执与空的待执行节点列表；并非把一个 Python 对象留在内存里重复调用。此例是正常退出后恢复，不代表突然断电、并发审批或数据库高可用已经通过测试。

### 三种存储路线怎么选

- **InMemorySaver**：单元测试、教程与一次性调试。无外部数据库，进程结束即丢状态；不能接收需要等待隔夜的真实审批。
- **SqliteSaver**：本地复现、小型单机试验。文件易备份，但并发、锁、备份恢复和多副本部署需单独验证；本页只验证跨进程恢复。
- **PostgresSaver / AsyncPostgresSaver**：多人服务的候选。需另装对应包、初始化表并管理连接、凭据、备份和保留期；不因换了 saver 就自动获得业务 exactly-once。

对于带副作用的发布或工单变更，checkpoint 与业务提交可能不在同一事务。接口写成功但图保存前崩溃，恢复时仍可能重发。执行端需原子登记业务幂等键，并校验同键请求内容；超时后按键查询真实状态，不能直接把“没收到响应”当“没执行”。回放旧 checkpoint 也不能免审批重放生产写入。

Checkpointer 管单个 thread 的执行状态；跨 thread 的用户偏好或知识应按 Store/业务存储另行设计。不要把全部历史 checkpoint 当成可直接送入模型的长期记忆，也不要把密码或原始敏感资料无界落库。

### 来源、版本和验收边界

2026-11-10 读取官方 [Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)、[Checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers)、[Persistence](https://docs.langchain.com/oss/python/langgraph/persistence) 和 [Checkpoint API](https://reference.langchain.com/python/langgraph/checkpoints/)。文档会滚动更新，本页只使用已安装版本可复现的 `invoke`、`interrupt` 与 `Command` API，不把新文档中的额外接口推定为 1.2.2 可用。

从最终 Markdown 提取代码复跑：14 项测试通过，覆盖原循环、静态断点、暂停无回执、同意/拒绝、错误摘要、非布尔同意值、字段绑定、thread 隔离、新内存 saver 无旧状态与 SQLite 分进程恢复。5 个 Python 代码块均通过语法检查；真实 LLM 块只检查语法，不执行。

新增示例的实际输出：

```text
SIMULATED:CHG-DEMO approval_entries= 2
paused ('approval',)
SIMULATED:CHG-SQLITE ()
```

实测为 Python 3.11.15、LangGraph 1.2.2、checkpoint 4.3.1、SQLite saver 3.1.3、langchain-core 1.7.7。完整依赖锁定、提取代码与测试报告在 `~/.hermes/workspaces/kb-weekly-2026-11-10/`。所有“批准”和“变更”均为本地测试数据；未连接真实模型、审批平台或业务写接口。

## 更新日志

- 2026-11-10：纠正静态断点等于审批、内存 checkpoint 等于重启持久化的表述；保留原有循环与调试代码，补动态审批、摘要绑定、跨进程 SQLite 恢复、三种 saver 选型与副作用重放边界。真实 LLM 示例仍未实测。

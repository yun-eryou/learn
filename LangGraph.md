

# LongGraph基础
> <font style="color:rgb(60, 60, 67);">LangGraph 运行时底层基于自研的 Pregel 运行时，其核心思想借鉴了 Google Pregel 计算模型，用于组织和执行复杂的图计算流程。</font>
>



## `LangGraph`和`LangChain`的版本迭代及定位


+ LangChain 1.x 的定位发生了明显变化：旧式 Chain、Retriever、Memory 等组件被迁移至 langchain-classic；LCEL 保留为 langchain-core 的底层组合机制，但不再是主要开发入口。
+ **<font style="color:#DF2A3F;">LangChain 1.x</font>** 现在聚焦于 Agent 开发，核心入口是 create_agent，围绕 Agent 提供模型、工具、消息、结构化输出、Middleware` 和记忆管理等高层抽象。
+ **<font style="color:#DF2A3F;">LangGraph </font>**是更底层的编排框架与 Agent Runtime，负责复杂工作流和有状态 Agent 的执行，提供持久化、流式输出、Durable Execution、Human-in-the-loop 等运行时能力。create_agent 底层基于 LangGraph` 实现。





| 对比维度 | LangChain | LangGraph |
| --- | --- | --- |
| 定位 | **<font style="color:#DF2A3F;">Agent 高层开发框架</font>** | **<font style="color:#DF2A3F;">底层编排框架 & Agent Runtime</font>** |
| 核心入口 | `create_agent` | `**<font style="color:#DF2A3F;">StateGraph</font>**`<br/>**<font style="color:#DF2A3F;"> / </font>**`**<font style="color:#DF2A3F;">@entrypoint</font>**` |
| 适用场景 | 结构直接的 Agent 应用 | 复杂工作流、持久化状态、长时间运行、人工介入 |
| 流程控制 | Agent 循环自动管理 | 精细控制节点、边、条件分支 |
| 学习成本 | 较低 | 较高 |




> `LangChain` 提供易于使用的 Agent 高层抽象，`LangGraph` 提供可靠、可持久化且可精细控制的底层执行能力。
>



对于大多数 Agent 项目，从 LangChain 的 create_agent 开始即可；需要复杂工作流编排、确定性步骤与 Agent 步骤混合、长时间运行或底层状态控制时，再引入 LangGraph。



<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791774365109-fcc82627-3cb6-4aa7-bd6f-98a5515d19d1.png)







## <font style="color:rgb(60, 60, 67);">构成图的基本要素</font>


LangGraph 运行时主要由三个基本要素构成：State（状态）、Node（节点） 和 Edge（边）

+ **<font style="color:#DF2A3F;">State（状态）</font>**：LangGraph 运行过程中的**<font style="color:#DF2A3F;">共享数据结构</font>**，用于表示应用在某一时刻的状态快照。**<font style="color:#DF2A3F;background-color:#FBDE28;">它承载了图运行所需的上下文信息、中间结果和后续节点需要读取的数据，是节点之间传递信息的核心载体</font>**。它与我们在学习 LangChain Agent 时使用的 State 是同一概念。
+ **<font style="color:#DF2A3F;">Node（节点）</font>**：LangGraph 中的具体**<font style="color:#DF2A3F;">执行单元</font>**，通常实现为一个函数。节点会读取当前 State，执行相应的业务逻辑，并返回对 State 的局部更新。节点本身并不直接修改全局状态，状态的合并与提交由运行时统一完成。
+ **<font style="color:#DF2A3F;">Edges（边）</font>**：用于定义节点之间的**<font style="color:#DF2A3F;">流转关系</font>**，决定一个节点执行完成后下一步应该进入哪个节点。Edge 可以是固定流转，也可以根据当前 State 进行条件判断，从而实现分支、循环等复杂控制流程。



下图展示了一个非常简单的运行图，只有两个计算节点，其拓扑结构为

```python
START --> node_1 --> node_2 --> END
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791774519265-00d6e7d8-b2c5-447b-82ca-fc9fac8f56b5.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791774549689-2d5efe5d-b327-4f4c-8bd5-17a34b5f4cd8.png)







## <font style="color:rgb(60, 60, 67);">图运行过程</font>
LangGraph 的图运行过程基于 **<font style="color:#DF2A3F;">Superstep（超步） 来组织和推进。</font>**Superstep 可以理解为图运行过程中的一次”单步循环”。一次图运行过程从开始到结束，就是由一系列连续的 Superstep 串联而成。



下文以 node_1执行完毕后的Superstep 为例说明。

每个 Superstep 通常可以分为三个阶段：

1. **<font style="color:#DF2A3F;">计划/路由阶段（Plan / Routing）：根据当前的 State（状态） 和 Edge（边） 的逻辑，确定本轮超步中应该被执行的节点。</font>**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791774565147-1ffc8a14-3ad9-45cd-9cd9-2a3fef8b861b.png)

 

2. 执行阶段（Execution）：运行本轮被选中的节点。如果本轮有多个节点同时被触发，它们会并行执行。每个节点都会基于本轮开始时的状态快照进行计算，并输出各自对状态的局部更新。在本阶段中，一个节点产生的更新不会立即被其他节点读取到。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791774774284-d776ec80-dcea-4569-86c5-ec2b05515d98.png)

3. 状态更新/提交阶段（Update / Commit）：当本轮所有节点都执行完成后，LangGraph 会将它们的输出统一合并到 State 中，生成新的状态快照。这个新状态会作为下一轮 Superstep 的输入。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791774786872-a4871617-1c10-46e7-8c55-ca718c17140a.png)











## Graph API vs Functional API
> LangGraph 提供了两种不同的 API 来构建运行图：`Graph API`（图式 API） 和 `Functional API`（函数式 API）。这两种 API 共享相同的底层运行时，可以在同一应用程序中协同使用，但它们针对不同的使用场景和开发偏好而设计。
>



<font style="color:rgb(60, 60, 67);"></font>

### Graph API


Graph API 采用声明式方式构建工作流。开发者需要显式定义**<font style="color:#DF2A3F;"> State、Node 和 Edge，将业务流程组织成一个可视化的图结构。</font>**

当流程中存在较复杂的分支、多个节点之间共享状态、并行执行、结果汇聚，或者需要通过图结构帮助调试和团队协作时，更适合使用 Graph API。官方文档也明确建议，在需要复杂流程可视化、显式状态管理、多条件分支、并行路径以及团队协作时，优先选择 Graph API。



典型场景包括：

| 场景 | 说明 |
| --- | --- |
| 多节点复杂流程 | 流程中存在多个处理节点，需要清晰表达节点之间的关系 |
| 条件分支较多 | 根据 State 中的不同字段决定后续执行路径 |
| 并行执行与结果汇聚 | 多个节点并行运行，之后汇总结果 |
| 多组件共享状态 | 多个节点都需要读写同一个全局 State |
| 需要图结构展示 | 便于调试、讲解、文档化和团队协作 |








### Functional API
Functional API 采用命令式方式构建工作流，更接近普通 Python 函数调用。开发者可以使用 `@**<font style="color:#DF2A3F;">entrypoint</font>**`**<font style="color:#DF2A3F;"> 定义工作流入口，使用 </font>**`**<font style="color:#DF2A3F;">@task</font>**`**<font style="color:#DF2A3F;"> 定义可被检查点记录的任务</font>**，然后在函数内部使用普通的 `i**<font style="color:#DF2A3F;">f/else</font>**`**<font style="color:#DF2A3F;">、循环和函数调用来组织流程</font>**。官方文档指出，当已有过程式代码需要最小改造、流程主要是线性的、分支逻辑较简单、希望快速原型验证时，更适合使用 Functional API。

可以这样理解：

> Functional API 更适合在普通 Python 函数流程中，以较低成本接入 LangGraph 的持久化、中断恢复和任务记录能力。
>

典型场景包括：

| 场景 | 说明 |
| --- | --- |
| 现有代码改造 | 原本已有函数式或过程式代码，不希望重构成完整图结构 |
| 线性流程 | 主要是 A → B → C 的顺序执行 |
| 简单分支 | 只有少量 `if/else`<br/> 判断 |
| 快速原型验证 | 希望减少样板代码，快速验证业务逻辑 |
| 局部任务持久化 | 希望某些函数作为独立 task 被检查点记录 |




### 二者的核心区别
| 对比项 | Graph API | Functional API |
| --- | --- | --- |
| 编程风格 | 声明式图结构 | 命令式函数流程 |
| 核心抽象 | State、Node、Edge | entrypoint、task |
| 状态管理 | 显式定义全局 State | 更多依赖函数参数和返回值 |
| 流程表达 | 通过节点和边表达 | 通过普通 Python 控制流表达 |
| 可视化能力 | 强，天然适合画图和调试 | 弱，更像普通代码流程 |
| 适合场景 | 复杂工作流、多分支、多节点协作 | 简单流程、快速原型、已有代码改造 |
| 学习成本 | 相对更高 | 相对更低 |




### 选型建议
从零构建或流程结构复杂，用 `Graph API`；现有代码改造、快速原型验证或流程逻辑简单，用 `Functional API`。

学习 `**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 建议优先掌握 </font>**`**<font style="color:#DF2A3F;">Graph API</font>**`。因为后者更能体现 `LangGraph` 的核心思想：通过 `State、Node、Edge` 显式描述一个可执行的计算图。

本文介绍 `Graph API`，对 `Functional API` 感兴趣的同学自行查阅

https://docs.langchain.com/oss/python/langgraph/functional-api





## <font style="color:rgb(60, 60, 67);">定义状态图</font>
状态图的定义可以分为以下几个步骤：

+ 1. 定义全局状态 定义整个运行图共享的全局状态（实际上，除了全局状态，LangGraph还支持其它类型的状态，下文详述）
+ 2. 创建状态图 创建 StateGraph实例，将状态和状态图绑定
+ 3. 定义图结构 定义节点和边并添加到状态图中

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

### <font style="color:rgb(60, 60, 67);">定义全局状态</font>
全局状态是 LangGraph 运行图每个节点都可以访问的公共对象。

```python
class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    cur_id: str
```









### <font style="color:rgb(60, 60, 67);">创建状态图</font>
```python
builder = StateGraph(state_schema=OverAllState)
```

  
 





### <font style="color:rgb(60, 60, 67);">定义图结构</font>
#### <font style="color:rgb(60, 60, 67);">定义图节点</font>
```python
def node_1(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }
```









#### <font style="color:rgb(60, 60, 67);">添加节点</font>
```python
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
```







#### <font style="color:rgb(60, 60, 67);">添加边</font>
```python
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)
```







## <font style="color:rgb(60, 60, 67);">调用状态图</font>
```python
print(graph.invoke({"cur_id": "start"}))
```











## <font style="color:rgb(60, 60, 67);">完整代码</font>
```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    cur_id: str

def node_1(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()

print(graph.invoke({"cur_id": "start"}))
```



运行结果

```python
{'logs': ['node_1 运行完毕', 'node_2 运行完毕'], 'cur_id': 'start, node_1, node_2'}
```











## <font style="color:rgb(60, 60, 67);">状态定义</font>
状态的定义实际上是在声明状态的Schema，后者是状态字段的完整描述。

官方推荐了三种定义Schema的方式：**<font style="color:#DF2A3F;">TypedDict、dataclass、Pydantic</font>**







### <font style="color:rgb(60, 60, 67);">TypedDict</font>
```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    cur_id: str

def node_1(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()

print(graph.invoke({"cur_id": "start"}))
```



结果

```python
{ "logs": ["node_1 运行完毕", "node_2 运行完毕"], "cur_id": "start, node_1, node_2" }
```



### <font style="color:rgb(60, 60, 67);">dataclass</font>
属性调用方式由`['字段名']`变为`.字段名`。

```python
from langgraph.graph import StateGraph, START, END
from typing import Annotated
from dataclasses import dataclass
from operator import add

@dataclass
class OverAllState:
    logs: Annotated[list[str], add]
    cur_id: str

def node_1(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()

print(graph.invoke({"cur_id": "start"}))
```





结果

```python
{ "logs": ["node_1 运行完毕", "node_2 运行完毕"], "cur_id": "start, node_1, node_2" }

```









### <font style="color:rgb(60, 60, 67);">Pydantic</font>
Pydantic模型的字段访问方式和dataclass相同。

```python
from langgraph.graph import StateGraph, START, END
from typing import Annotated
from pydantic import BaseModel
from operator import add

class OverAllState(BaseModel):
    logs: Annotated[list[str], add]
    cur_id: str

def node_1(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()

print(graph.invoke({"cur_id": "start"}))
```





结果

```python
{ "logs": ["node_1 运行完毕", "node_2 运行完毕"], "cur_id": "start, node_1, node_2" }
```









### <font style="color:rgb(60, 60, 67);">校验行为</font>
学习LangChain的结构化输出时我们提到：Pydantic对格式要求最严格，如果模型返回的内容不符合结构化Schema的要求，则抛出`ValidationError`。而其余方式都不会对模型的返回结果进行校验，即便模型返回的内容不符合结构化要求，也会原样返回给用户。

而作为LangGraph计算图的状态时，这三种方式都要求字段名称完全一致。只是处理方式不同。具体规则如下







#### <font style="color:rgb(60, 60, 67);">输入字段不匹配</font>
##### 1. TypedDict
TypedDict将**<font style="color:#DF2A3F;">输入字段视为字典的</font>**`**<font style="color:#DF2A3F;">Key</font>**`，不匹配时抛出`KeyError`异常。

##### 2. dataclass
**<font style="color:#DF2A3F;">dataclass将输入字段视为类的</font>**`**<font style="color:#DF2A3F;">属性</font>**`，不匹配时抛出`TypeError（类型错误）`异常。

##### 3. Pydantic
**<font style="color:#DF2A3F;">Pydantic对输入字段进行校验</font>**，不匹配时抛出`ValidationError`异常。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

#### <font style="color:rgb(60, 60, 67);">节点返回字段不匹配</font>
<font style="color:rgb(60, 60, 67);">图节点返回的是对于状态的更新，如果返回字段和状态字段不匹配，上述三种Schema定义方式的行为是统一的：状态更新会被忽略。</font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

### <font style="color:rgb(60, 60, 67);">推荐用法</font>
在实际使用中，**<font style="color:#DF2A3F;">推荐优先使用 </font>**`**<font style="color:#DF2A3F;">TypedDict</font>**`**<font style="color:#DF2A3F;"> 定义 LangGraph 状态图的 State Schema</font>**。

大多数官方案例也采用 `TypedDict` 方式定义状态 `Schema`。这种方式写法简洁、结构清晰，能够直接描述状态中包含哪些字段，以及每个字段对应的数据类型，非常适合用于定义图运行过程中的共享状态。

相比普通 `dict`，`TypedDict` 可以提供更明确的字段约束和类型提示；相比 `dataclass`，`TypedDict` 更贴近 LangGraph 中状态的更新方式，因为节点通常返回的是表示“部分状态更新”的字典，而不是完整对象；相比 `Pydantic BaseModel`，它又更加轻量，不会引入额外的数据校验开销。因此，在没有复杂校验需求的情况下，`TypedDict` 是定义 LangGraph State Schema 的首选方式。







## State Reducer
### 什么是State Reducer
State Reducer 是 LangGraph 中用于合并状态更新的核心机制。在 **<font style="color:#DF2A3F;">LangGraph 的 </font>**`**<font style="color:#DF2A3F;">StateGraph</font>**`**<font style="color:#DF2A3F;"> 中，每个节点可以读取和写入共享状态</font>**，而 Reducer 定义了如何将**<font style="color:#DF2A3F;">多个节点对同一状态键的更新合并</font>**

**<font style="color:#DF2A3F;">Reducer 的核心特征：</font>**

+ 函数签名：`**<font style="color:#DF2A3F;background-color:#FBDE28;">(Value, Value) -> Value</font>**`**<font style="color:#DF2A3F;">，接收当前值和更新值，返回合并后的新值</font>**
+ 注解定义：通过 `Annotated[Type, reducer_function]` 为状态键指定 Reducer
+ 默认行为：未指定 Reducer 的状态键使用覆盖策略（Last-Write-Wins）



### 如何定义Reducer
#### 定义Reducer函数
Reducer 本质上是一个二元合并函数，用于定义当同一个字段产生多个更新值时，LangGraph 应该如何将这些值合并为一个最终结果。

函数签名：`(Value, Value) -> Value`

示例代码如下：

```python
def my_reducer(left: list[str], right: list[str]) -> list[str]:
    return left + right

left = ['a', 'b']
right = ['c']
print(my_reducer(left, right))
```

其中，`my_reducer` 用于处理 `list[str]` 类型的数据。它接收两个列表参数：

+ `**<font style="color:#DF2A3F;">left</font>**`**<font style="color:#DF2A3F;">：当前已累计的状态值；</font>**
+ `**<font style="color:#DF2A3F;">right</font>**`**<font style="color:#DF2A3F;">：本次待合并的新值。</font>**

函数内部通过 `**<font style="color:#DF2A3F;">left + right</font>**`**<font style="color:#DF2A3F;"> 将两个列表合并，并返回合并后的结果。</font>**

因此，该 Reducer 的作用是：当某个状态字段存在多次列表更新时，将这些列表内容追加合并，而不是直接覆盖原值。



运行结果如下

```python
['a', 'b', 'c']
```







#### 将Reducer和状态字段关联
在 LangGraph 中，**<font style="color:#DF2A3F;">Reducer 通常通过 Python 的 </font>**`**<font style="color:#DF2A3F;">typing.Annotated</font>**` 与状态字段进行关联。

`**<font style="color:#DF2A3F;">Annotated[]</font>**`**<font style="color:#DF2A3F;"> 是 Python 提供的一种类型注解扩展机制，用于在原始类型之外附加额外的元数据信息。需要注意的是，</font>**`**<font style="color:#DF2A3F;">Annotated[]</font>**`**<font style="color:#DF2A3F;"> 本身并不规定这些元数据的具体含义</font>**，它只负责在类型注解中保留这些信息。

严格来说，`**<font style="color:#DF2A3F;">Annotated</font>**`**<font style="color:#DF2A3F;"> 的第一个参数是被注解的原始类型，后续参数是附加的元数据。</font>**至于这些元数据表示什么、如何解析，**<font style="color:#DF2A3F;">则由使用它的框架或工具自行决定。</font>**

在 LangGraph 中，框架利用这一机制，将状态字段的类型和 Reducer 规则同时声明在字段定义中。其基本形式如下：

```python
Annotated[Type, reducer_function]
```

其中：

+ `**<font style="color:#DF2A3F;">Type</font>**`**<font style="color:#DF2A3F;">：表示状态字段的数据类型；</font>**
+ `**<font style="color:#DF2A3F;">reducer_function</font>**`**<font style="color:#DF2A3F;">：表示该字段对应的 Reducer 函数。</font>**





```python
from typing import TypedDict, Annotated

class OverAllState(TypedDict):
    logs: Annotated[list[str], my_reducer]
    cur_id: str
```

在上述代码中：

+ `logs` 字段的类型是 `list[str]`；
+ `**<font style="color:#DF2A3F;">my_reducer</font>**`**<font style="color:#DF2A3F;"> 是与 </font>**`**<font style="color:#DF2A3F;">logs</font>**`**<font style="color:#DF2A3F;"> 字段关联的 </font>****<font style="color:#DF2A3F;background-color:#FBDE28;">Reducer 函数；</font>**
+ 当多个节点同时更新 `logs` 字段时，**<font style="color:#DF2A3F;">LangGraph 会使用 </font>**`**<font style="color:#DF2A3F;">my_reducer</font>**`**<font style="color:#DF2A3F;"> 将多个列表合并</font>**；
+ `cur_id` 字段没有指定 Reducer，因此采用**<font style="color:#DF2A3F;">默认更新规则。</font>**











### <font style="color:rgb(60, 60, 67);">常用内置Reducer函数</font>
#### operator.add
`operator.add` 是 Python 内置的加法操作函数，底层由 C 实现

**<font style="color:#DF2A3F;">它接收两个参数，等价于 </font>**`**<font style="color:#DF2A3F;">a（第一个参数）+b（第二个参数）</font>**`

代码如下

```python
from operator import add

print(f"{add(1,2) = }")
print(f"{add([1,2], [3,4]) = }")
print(f"{add(['a','b'], ['c']) = }")
```



```python
add(1,2) = 3
add([1,2], [3,4]) = [1, 2, 3, 4]
add(['a','b'], ['c']) = ['a', 'b', 'c']
```







#### <font style="color:rgb(60, 60, 67);"> langgraph.graph.message.add_messages</font>
`add_messages` 是 LangGraph 中专用于合并消息列表的 Reducer 函数，常用于维护对话历史类的状态字段。其函数签名如下：

```python
def add_messages(
    left: Messages,
    right: Messages,
    *,
    format: Literal["langchain-openai"] | None = None,
) -> Messages:
    ...
    return merged
```

参数说明：

+ `left`：**<font style="color:#DF2A3F;">状态中已有的消息列表</font>**；
+ `right`：**<font style="color:#DF2A3F;">当前节点返回的消息更新值；</font>**
+ `format`：可选参数，用于指定返回消息的格式，通常无需手动设置。

`left` 与 `right` 的类型均为 `Messages`。`**<font style="color:#DF2A3F;">Messages</font>**`**<font style="color:#DF2A3F;"> 可以理解为 LangChain 消息对象的列表，其中每个元素都是 </font>**`**<font style="color:#DF2A3F;">BaseMessage</font>**`**<font style="color:#DF2A3F;"> 或其子类的实例</font>**，常见子类包括：

+ `HumanMessage`：用户的输入消息；
+ `AIMessage`：AI 的回复消息；
+ `SystemMessage`：系统提示消息；
+ `ToolMessage`：工具调用的结果消息。



`add_messages` **<font style="color:#DF2A3F;">处理的是对话消息序列，而非普通的字符串列表。</font>**

`BaseMessage` 包含一个可选的 `id` 属性，用于唯一标识一条消息。`add_messages` 在合并 `left` 与 `right` 时，不是简单地执行列表拼接，而**<font style="color:#DF2A3F;">是依据消息的 </font>**`**<font style="color:#DF2A3F;">id</font>**`**<font style="color:#DF2A3F;"> 进行合并：</font>**

+ 若 `right` 中的某条消息的 `id` 在 `left` 中不存在，**<font style="color:#DF2A3F;">则将该消息追加到结果列表末尾；</font>**
+ **<font style="color:#DF2A3F;">若 </font>**`**<font style="color:#DF2A3F;">right</font>**`**<font style="color:#DF2A3F;"> 中的某条消息的 </font>**`**<font style="color:#DF2A3F;">id</font>**`**<font style="color:#DF2A3F;"> 与 </font>**`**<font style="color:#DF2A3F;">left</font>**`**<font style="color:#DF2A3F;"> 中已有消息的 </font>**`**<font style="color:#DF2A3F;">id</font>**`**<font style="color:#DF2A3F;"> 相同，则使用 </font>**`**<font style="color:#DF2A3F;">right</font>**`**<font style="color:#DF2A3F;"> 中的新消息替换 </font>**`**<font style="color:#DF2A3F;">left</font>**`**<font style="color:#DF2A3F;"> 中的旧消息。</font>**

**<font style="color:#DF2A3F;"></font>**

**<font style="color:#DF2A3F;"></font>**

因此，`add_messages` 的作用可以概括为：**<font style="color:#DF2A3F;">在保留历史消息的基础上追加新消息，并允许通过相同的消息 </font>**`**<font style="color:#DF2A3F;">id</font>**`**<font style="color:#DF2A3F;"> 覆盖已有消息。</font>**

需要特别说明，`add_messages` 并非简单地对 `left` 与 `right` 求“并集”。更准确地说，它是一个基于消息 `id` 的消息列表合并函数：既支持追加新消息，也支持更新已有消息。







可以理解为

```python
merged = left + right
```



但若 `right` 中存在与 `left` 相同 `id` 的消息，**<font style="color:#DF2A3F;">则最终结果中不会出现重复消息，而是用 </font>**`**<font style="color:#DF2A3F;">right</font>**`**<font style="color:#DF2A3F;"> 中的消息覆盖 </font>**`**<font style="color:#DF2A3F;">left</font>**`**<font style="color:#DF2A3F;"> 中对应的旧消息。</font>**

示例代码如下

```python
from langgraph.graph.message import add_messages
from langchain.messages import HumanMessage, AIMessage, SystemMessage

left = [
    SystemMessage(content="你是个善解人意的助手", id='1'),
    HumanMessage(content="你好", id='2'),
    AIMessage(content="你好~", id='3'),
]

right = [
    HumanMessage(content="我是老王，你是小王", id='2'),
    AIMessage(content="好的，我记住啦", id='3'),
    HumanMessage(content="你是谁？", id='4'),
    AIMessage(content="我是小王", id='5'),
]

merged = add_messages(left, right)

for msg in merged:
    print(msg)
```



<font style="color:rgb(60, 60, 67);">输出如下</font>

```python
content='你是个善解人意的助手' additional_kwargs={} response_metadata={} id='1'
content='我是老王，你是小王' additional_kwargs={} response_metadata={} id='2'
content='好的，我记住啦' additional_kwargs={} response_metadata={} id='3' tool_calls=[] invalid_tool_calls=[]
content='你是谁？' additional_kwargs={} response_metadata={} id='4'
content='我是小王' additional_kwargs={} response_metadata={} id='5' tool_calls=[] invalid_tool_calls=[]
```









### 默认行为
如果某个 State 字段没有显式定义 `Reducer`，**<font style="color:#DF2A3F;">LangGraph 会使用默认的状态更新行为：后一次更新值会覆盖该字段原有的状态值。</font>**

换句话说，当节点返回的更新结果中包含某个字段时，如果该字段没有配置 Reducer，LangGraph 不会对新旧值进行追加、合并或累加，**<font style="color:#DF2A3F;">而是直接使用本次返回的新值替换原来的旧值。</font>**

示例代码如下：

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class OverAllState(TypedDict):
    logs: list[str]
    id: str

def node_a(state: OverAllState):
    return {
        "logs": ["node_a"],
        "id": "node_a"
    }

def node_b(state: OverAllState):
    return {
        "logs": ["node_b"],
        "id": "node_b"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
print('=' * 30, '-> result <-', '=' * 30)
print(result)
```

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);">输出如下</font>

```python
============================== -> result <- ==============================
{'logs': ['node_b'], 'id': 'node_b'}
```

可以看到，`logs`字段和`id`字段都没有定义`Reducer`，因此，节点返回的新值会覆盖初始状态中的旧值，图运行结果中的状态值和最后一次更新保持一致。









### 对于并发节点的配置
**<font style="color:#DF2A3F;background-color:#FBDE28;">对于我们的并发节点来说，我们必须要配置reducer才可以，否者我们节点并发执行的时候，会直接报错，因为不知道谁覆盖谁，直接报错</font>**























## 节点中访问State
### 图节点中读取State


在 LangGraph 中，节点本质上是一个可调用对象，**<font style="color:#DF2A3F;">通常定义为普通 Python 函数。节点函数被执行时，LangGraph 会自动将当前图运行到该节点时的 </font>**`**<font style="color:#DF2A3F;">State</font>**`**<font style="color:#DF2A3F;"> </font>**传入节点函数。

节点函数的第一个参数通常是当前运行图的状态对象，**<font style="color:#DF2A3F;">也就是 </font>**`**<font style="color:#DF2A3F;">State</font>**`**<font style="color:#DF2A3F;">。</font>**

```python
def node(state: StateSchema):
```

其中，`**<font style="color:#DF2A3F;">state</font>**`**<font style="color:#DF2A3F;"> 表示当前节点执行时可以访问到的全局状态快照。节点可以通过读取 </font>**`**<font style="color:#DF2A3F;">state</font>**`**<font style="color:#DF2A3F;"> 中的字段获取上游节点写入的数据，并基于这些数据完成当前节点的业务逻辑</font>**。





示例代码如下：

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    id: str

def node_a(state: OverAllState):
    for k, v in state.items():
        print(f"k: {k}, v: {v}")

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
```



```python
k: logs, v: ['START']
k: id, v: start
```







### 图节点更新State
在 LangGraph 中，节点函数通常不需要返回更新后的完整状态，只需要返回本节点对状态的局部更新。

也就是说，节点的返回值可以只包含需要修改的状态字段。

+ 对于节点没有返回的字段，**<font style="color:#DF2A3F;">LangGraph 会保留其原有状态值；</font>**
+ 对于节点返回的字段，LangGraph 会根据该字段是否配置了 `Reducer` 来决定如何合并更新值。
    - **<font style="color:#DF2A3F;">如果字段配置了 </font>**`**<font style="color:#DF2A3F;">Reducer</font>**`**<font style="color:#DF2A3F;">，则使用对应的 </font>**`**<font style="color:#DF2A3F;">Reducer</font>**`**<font style="color:#DF2A3F;"> 函数将旧值和新值合并</font>**；
    - **<font style="color:#DF2A3F;">如果字段没有配置 </font>**`**<font style="color:#DF2A3F;">Reducer</font>**`**<font style="color:#DF2A3F;">，则按照默认规则使用节点返回的新值覆盖原值。</font>**

LangGraph运行时会按照状态字段的`Reducer`函数将其与当前的最新状态合并。





```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    id: str

def node_a(state: OverAllState):
    for k, v in state.items():
        print(f"k: {k}, v: {v}")
    return {
        "logs": ["node_a 更新状态"]
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
print('=' * 30, '-> result <-', '=' * 30)
print(result)
```





<font style="color:rgb(60, 60, 67);">输出如下</font>

```python
k: logs, v: ['START']
k: id, v: start
============================== -> result <- ==============================
{'logs': ['START', 'node_a 更新状态'], 'id': 'start'}
```



在上述示例中：

+ `logs` 字段通过 `Annotated[list[str], add]` 绑定了 Reducer 函数 `operator.add`，因此 LangGraph 会将原有的 `**<font style="color:#DF2A3F;">logs</font>**`**<font style="color:#DF2A3F;"> 值和 </font>**`**<font style="color:#DF2A3F;">node_a</font>**`**<font style="color:#DF2A3F;"> 返回的新 </font>**`**<font style="color:#DF2A3F;">logs</font>**`**<font style="color:#DF2A3F;"> 值进行列表拼接</font>**：

```python
["START"] + ["node_a 更新状态"]
```





最终得到：

```python
["START", "node_a 更新状态"]
```

+ `id` 字段没有出现在 `node_a` 的返回值中，因此该字段不会被更新。图运行结束后，输出状态中的 `id` 仍然保持输入时的值：

```python
"id": "start"
```



因此，LangGraph 节点更新 State 的核心规则可以概括为：

**<font style="color:#DF2A3F;">节点只返回需要更新的字段；未返回的字段保持不变；返回的字段根据是否配置 Reducer 决定是合并还是覆盖。</font>**

**<font style="color:#DF2A3F;"></font>**

**<font style="color:#DF2A3F;"></font>**

**<font style="color:#DF2A3F;"></font>**

### Overwrite绕过Reducer
+ 在前面的示例中，如果某个状态字段定义了 `Reducer`，那么节点返回该字段的更新值时，LangGraph 默认会通过对应的 `Reducer` 将新值与已有状态值进行合并。
+ 在某些场景下，我们可能并不希望继续执行 `Reducer` 的聚合逻辑，而是希望本次更新直接覆盖旧值。**<font style="color:#DF2A3F;">这时可以使用 </font>**`**<font style="color:#DF2A3F;">Overwrite</font>**`**<font style="color:#DF2A3F;">。</font>**
+ `Overwrite` 的作用是：**<font style="color:#DF2A3F;">告诉 LangGraph 本次状态更新不走该字段原本定义的 </font>**`**<font style="color:#DF2A3F;">Reducer</font>**`**<font style="color:#DF2A3F;">，而是直接用新值覆盖状态中的旧值。</font>**
+ 需要注意的是，`Overwrite` 只影响当前这一次更新，并不会修改状态字段本身的 `Reducer` 定义。后续节点如果继续正常返回该字段的更新值，**<font style="color:#DF2A3F;">仍然会按照原来的 </font>**`**<font style="color:#DF2A3F;">Reducer</font>**`**<font style="color:#DF2A3F;"> 逻辑进行合并</font>**。

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Overwrite
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    id: str

def node_a(state: OverAllState):
    return {
        "logs": ["node_a"],
        "id": "node_a"
    }

def node_b(state: OverAllState):
    return {
        "logs": Overwrite(["node_b"]),
        "id": "node_b"
    }

def node_c(state: OverAllState):
    return {
        "logs": ["node_c"],
        "id": "node_c"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_c")
builder.add_edge("node_c", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
print('=' * 30, '-> result <-', '=' * 30)
print(result)
```



<font style="color:rgb(60, 60, 67);">输出如下</font>

```python
============================== -> result <- ==============================
{'logs': ['node_b', 'node_c'], 'id': 'node_c'}
```



上述代码中：

+ `logs` 字段绑定了 `operator.add()` 函数
    - 如果没有 `**<font style="color:#DF2A3F;">Overwrite</font>**`**<font style="color:#DF2A3F;">，则最终输出的 </font>**`**<font style="color:#DF2A3F;">logs</font>**`**<font style="color:#DF2A3F;"> 的值应为</font>** `['START', 'node_a', 'node_b', 'node_c']`
    - `node_b` 返回更新时，**<font style="color:#DF2A3F;">用 </font>**`**<font style="color:#DF2A3F;">Overwrite</font>**`**<font style="color:#DF2A3F;"> 包裹了 </font>**`**<font style="color:#DF2A3F;">logs</font>**`**<font style="color:#DF2A3F;"> 字段的值</font>**，那么当前状态的 `logs` 会被 `["node_b"]` 覆盖，因此最终输出的 `logs` 字段值变成了 `['node_b', 'node_c']`
+ `id` 字段按照默认行为，保留最后一次更新的值。







## Multi Schema用法


### 状态类型
`LangGraph` 支持在一个图中使用多个状态 Schema，用于区分图的外部输入、外部输出、内部共享状态以及节点间的临时状态。

常见状态类型可以分为以下几类：

+ **<font style="color:#DF2A3F;background-color:#FBDE28;">全局状态 / 内部状态：</font>**图内部主要使用的状态，创建 `StateGraph` 时传递给 `state_schema` 参数。**<font style="color:#DF2A3F;">它通常包含图运行过程中需要读写的大部分字段。</font>**
+ **<font style="color:#DF2A3F;background-color:#FBDE28;">输入状态：图对外接收输入时使用的状态</font>****<font style="color:#DF2A3F;">，</font>**创建 `StateGraph` 时传递给 `input_schema` 参数。**<font style="color:#DF2A3F;">它用于约束调用图时允许传入哪些字段。</font>**
+ **<font style="color:#DF2A3F;background-color:#FBDE28;">输出状态：图最终对外返回结果时使用的状态</font>**，创建 `StateGraph` 时传递给 `output_schema` 参数。**<font style="color:#DF2A3F;">它用于约束图运行结束后只返回哪些字段。</font>**
+ **<font style="color:#DF2A3F;background-color:#FBDE28;">私有状态：图内部节点之间传递的临时状态</font>****<font style="color:#DF2A3F;">，通常不作为图的输入，也不作为图的最终输出</font>**。它可以通过节点函数的入参类型注解声明，并在节点返回值中写入。

需要注意，输入状态和输出状态主要面向图的边界，即“图如何接收外部输入”和“图如何返回外部结果”；**<font style="color:#DF2A3F;">而全局状态和私有状态主要面向图内部节点之间的数据传递。</font>**















### 状态之间的关系
#### 设计规范


本节主要说明 `LangGraph` 状态设计中的规范。以下规则属于工程上的最佳实践，违反这些规范未必一定导致程序报错，但容易降低代码的可读性和可维护性。

1. 输入状态和输出状态通常应是全局状态的子集 输入状态描述图对外需要接收的数据，输出状态描述图最终需要返回的数据。通常情况下，它们都应该是全局状态的一部分。 如：

```python
class InputState(TypedDict):
    username: str

class OutputState(TypedDict):
    graph_output: str

class OverAllState(TypedDict):
    username: str
    nickname: str
    graph_output: str
```



1. 其中，`InputState` 和 `OutputState` 的所有字段均存在于 `OverAllState` 中。
2. **<font style="color:#DF2A3F;">私有状态和全局状态应尽量避免字段重名</font>** 私有状态的定位是图内部某些节点之间传递的临时字段。如果私有状态字段和全局状态字段重名，虽然某些情况下程序仍然可以运行，但容易让人误以为该字段是全局共享字段，从而造成理解混乱。 因此，推荐让私有状态字段和全局状态字段保持清晰边界。
3. **<font style="color:#DF2A3F;">节点函数应明确声明入参状态类型和返回状态类型</font>** 节点函数的第一个参数通常是当前节点可读取的状态。**<font style="color:#DF2A3F;">通过类型注解声明该参数，可以明确表达该节点需要读取哪些字段</font>**。 同时，给节点函数声明返回状态类型，也可以帮助阅读者理解该节点会更新哪些字段。 例如：

```python
def node_1(state: InputState) -> OverAllState:
    return {
        "nickname": "Dear " + state["username"]
    }
```



4. **<font style="color:#DF2A3F;">节点函数中不应该访问入参状态类型中不存在的字段 </font>**节点实际接收到的状态会按照其入参类型进行裁剪。因此，如果节点入参声明为 `InputState`，就不应该在节点内部访问 `InputState` 中不存在的字段。 例如：

```python
def node_1(state: InputState) -> OverAllState:
    return {
        "nickname": state["username"]
    }
```



如果在该函数中访问：

```python
state["nickname"]
```



1. 而 `nickname` 不属于 `InputState`，**<font style="color:#DF2A3F;">运行时就可能抛出 </font>**`**<font style="color:#DF2A3F;">KeyError</font>**`**<font style="color:#DF2A3F;">。</font>**
2. 节点函数返回的字典应尽量和返回类型注解保持一致 从 Python 类型注解的角度看，函数返回类型只是静态提示，**<font style="color:#DF2A3F;">运行时不会自动强制校验</font>**。 从 LangGraph 的运行机制看，节点返回的是对状态的部分更新，不是完整状态。只要返回字段已经被图记录为可用状态字段，LangGraph 就可以将其作为状态更新处理。 不过，从工程规范上讲，节点返回字典中的字段最好和函数返回类型注解保持一致，这样更利于阅读、调试和维护。







#### 源码层面的约束
##### 状态的记录


`LangGraph` 的状态并不是简单保存在一个普通字典中，而是会被拆分成多个可读写的状态字段。每个状态字段在**<font style="color:#DF2A3F;">底层通常对应一个 </font>**`**<font style="color:#DF2A3F;">Channel</font>**`**<font style="color:#DF2A3F;">。</font>**

这些状态字段会在不同阶段被记录到状态图中。



1. `StateGraph` 记录状态字段的核心方法是 `_add_schema()``_add_schema()` 会解析传入的状态 **<font style="color:#DF2A3F;">Schema，并将其中声明的字段记录到图中</font>**，使这些字段成为图运行时可以读写的状态字段。
2. 创建 `StateGraph` 时，会记录 `state_schema`、`input_schema` 和 `output_schema` 中的字段 当创建状态图时：

```python
builder = StateGraph(
    OverAllState,
    input_schema=InputState,
    output_schema=OutputState
)
```

LangGraph 会解析这些**<font style="color:#DF2A3F;"> Schema，</font>**并将其中涉及的字段加入**<font style="color:#DF2A3F;">图的状态管理体系。</font>**

3. 调用 `add_node()` 添加节点时，也可能记录节点入参声明的状态 Schema 当添加节点时，`LangGraph` 会根据节点函数第一个参数的类型注解推断该节点的输入状态类型。 **<font style="color:#DF2A3F;background-color:#FBDE28;">如果这个输入状态类型之前没有被图记录过</font>****<font style="color:#DF2A3F;">，LangGraph 也会通过 </font>**`**<font style="color:#DF2A3F;">_add_schema()</font>**`**<font style="color:#DF2A3F;"> 将其加入图中</font>**。 这也是私有状态能够生效的原因。 例如：

```python
class PrivateState(TypedDict):
    greeting: str

def node_3(state: PrivateState) -> OutputState:
    return {
        "graph_output": state["greeting"]
    }
```

当 `node_3` 被添加到图中时，`PrivateState` 中的 `greeting` 字段会被记录到图中，**<font style="color:#DF2A3F;background-color:#FBDE28;">从而成为图内部可以传递的状态字段。</font>**



4. 总结
    - 全局状态、输入状态、输出状态通常在创建 `StateGraph` 时被记录。
    - 私有状态通常在调用 `add_node()` 添加节点时，**<font style="color:#DF2A3F;background-color:#FBDE28;">根据节点入参类型注解被记录</font>**。
    - 被记录后的状态字段，**<font style="color:#DF2A3F;background-color:#FBDE28;">底层会成为图运行时可以读写的状态字段</font>**。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

##### 状态的访问
1. 调用图时，输入会按照 `**<font style="color:#DF2A3F;">input_schema</font>**` 进行约束 当调用图时

```python
graph.invoke({"username": "小黄"})
```

+ 如果创建图时声明了 `**<font style="color:#DF2A3F;">input_schema</font>**`**<font style="color:#DF2A3F;">，那么外部输入会按照 </font>**`**<font style="color:#DF2A3F;">input_schema</font>**`**<font style="color:#DF2A3F;"> 进行约束。 </font>**
+ 如果没有声明 `input_schema`，则通常按照 `**<font style="color:#DF2A3F;">state_schema</font>**`**<font style="color:#DF2A3F;"> 作为图的输入 Schema。</font>** 
+ 因此，`input_schema` 的作用不是“只让第一个节点可见”，而是约束图的外部输入结构。 此处的约束是指：按照 `schema` 裁剪输入，只保留 `schema` 中出现的状态字段



2. 节点接收到的状态会**<font style="color:#DF2A3F;">按照节点入参类型进行裁剪</font>** **<font style="color:#DF2A3F;">每个节点能读取哪些字段</font>**，主要取决于该节点第一个参数的类型注解。 例如：

```python
def node_1(state: InputState) -> OverAllState:
    ...
```

此时，`node_1` 接收到的 `state` 会按照 `InputState` 进行裁剪。即使图的全局状态中还有其他字段，`node_1` 也不应该访问不属于 `InputState` 的字段。 如果访问了入参状态中不存在的字段，例如：

```python
state["nickname"]
```

就可能抛出：

```python
KeyError
```

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

3. 节点返回的是状态更新**<font style="color:#DF2A3F;">，而不是完整状态 节点函数不需要返回完整状态</font>**，只需要返回本节点想要更新的字段。 例如：

```python
def node_1(state: InputState) -> OverAllState:
    return {
        "nickname": "Dear " + state["username"]
    }
```

这里虽然返回类型注解是 `OverAllState`，但函数实际只返回了 `nickname` 一个字段。这是允许的，**<font style="color:#DF2A3F;">因为 LangGraph 会把节点返回值视为对状态的部分更新。</font>**





4. 节点返回值的应用主要由字段名称和图中已记录的状态字段决定
    1. 节点返回的字典会根据字段名称写入对应状态字段，并按照该字段的 `Reducer` 规则进行合并。 
    2. 需要注意的是，**<font style="color:#DF2A3F;">函数返回类型注解主要用于表达代码意图，不是严格的运行时写入边界</font>**。 也就是说，如果某个字段已经被图记录为可用状态字段，那么节点即使没有在返回类型注解中声明该字段，也可能仍然可以返回并更新它。
    3.  不过，为了代码清晰，仍然推荐让节点的返回值和返回类型注解保持一致。



5. **<font style="color:#DF2A3F;">最终输出会按照 </font>**`**<font style="color:#DF2A3F;">output_schema</font>**`**<font style="color:#DF2A3F;"> 进行裁剪</font>** 图运行完成后，最终返回给外部调用方的结果会按照 `output_schema` 进行裁剪。 因此，`output_schema` 的作用不是“只让最后一个节点可见”，**<font style="color:#DF2A3F;">而是约束图最终对外暴露哪些字段。</font>** 例如，图内部状态中可能同时存在：

```python
username
nickname
greeting
graph_output
```

<font style="color:rgb(60, 60, 67);"></font>

但如果 `output_schema` 只包含：

```python
graph_output
```

那么最终 `graph.invoke()` 的返回结果就只会包含 `graph_output`。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

### 案例
下面通过一个简单案例说明四类状态的定义和使用。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class InputState(TypedDict):
    username: str

class OutputState(TypedDict):
    graph_output: str

class OverAllState(TypedDict):
    nickname: str
    username: str
    graph_output: str

class PrivateState(TypedDict):
    greeting: str

def node_1(state: InputState) -> OverAllState:
    # 向全局状态写入数据
    return {
        "nickname": "Dear " + state["username"]
    }

def node_2(state: OverAllState) -> PrivateState:
    # 从全局状态读取数据，写入私有状态
    return {
        "greeting": state["nickname"] + ", 早上好~"
    }

def node_3(state: PrivateState) -> OutputState:
    # 从私有状态读取数据，写入输出状态
    return {
        "graph_output": state["greeting"] + " 很高兴认识你！"
    }

builder = StateGraph(OverAllState,input_schema=InputState,output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
print(graph.invoke({"username":"小黄"}))
```

输出如下

```python
{ "graph_output": "Dear 小黄, 早上好~ 很高兴认识你！" }
```

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>



### 案例执行过程分析
上述案例中，各类状态的作用如下。

#### 1. 输入状态
```python
class InputState(TypedDict):
    username: str
```

`InputState` 用于约束图的外部输入。

因此调用图时，只需要传入：

```python
{"username": "小黄"}
```

#### 2. 全局状态
```python
class OverAllState(TypedDict):
    username: str
    nickname: str
    graph_output: str
```

`OverAllState` 是图内部主要使用的状态 Schema。

其中：

+ `username` 来自图的输入；
+ `nickname` 由 `node_1` 写入；
+ `graph_output` 由 `node_3` 写入，并最终作为图的输出返回。

#### 3. 私有状态
```python
class PrivateState(TypedDict):
    greeting: str
```

`PrivateState` 用于节点之间传递临时数据。

在本例中：

```python
def node_2(state: OverAllState) -> PrivateState:
    return {
        "greeting": state["nickname"] + ", 早上好~"
    }
```

`node_2` 写入了 `greeting` 字段。

随后：

```python
def node_3(state: PrivateState) -> OutputState:
    return {
        "graph_output": state["greeting"] + " 很高兴认识你！"
    }
```

`node_3` 通过 `PrivateState` 读取 `greeting` 字段，并生成最终输出。

#### 4. 输出状态
```python
class OutputState(TypedDict):
    graph_output: str
```

`OutputState` 用于约束图最终返回给外部调用方的数据。

虽然图内部运行过程中还存在 `username`、`nickname`、`greeting` 等字段，但最终结果只返回：

```python
\{ "graph_output": "Dear 小黄, 早上好~ 很高兴认识你！" }
```



这是因为图创建时声明了：

```python
output_schema=OutputState
```

所以最终输出会按照 `OutputState` 进行裁剪。

<font style="color:rgb(60, 60, 67);"></font>

## 预定义状态 
### MessagesState
`LangGraph` 构建的计算图通常会和 `LLM` 结合使用，而 `LLM` 在运行过程中通常需要维护一组消息列表。为了提升开发效率，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 官方提供了一个预定义状态类型：</font>**`**<font style="color:#DF2A3F;">langgraph.graph.message.MessagesState</font>**`**<font style="color:#DF2A3F;">。</font>**

开发者可以直接继承该状态类型，并在其基础上扩展自定义状态字段。



源码如下：

```python
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

由此可知，`MessagesState` 只有一个字段：`**<font style="color:#DF2A3F;">messages</font>**`。

该字段的类型是列表，元素类型为 `AnyMessage`；同时，它通过 `Annotated` 绑定了内置 `Reducer` 函数 `add_messages`。

+ `add_messages` 的完全限定名（英文全称 `fully qualified name`）是：
+ `langgraph.graph.message.add_messages`，正是上文 3.2.2.3.2 节介绍的内置 `Reducer` 函数

```python
from langchain_deepseek import ChatDeepSeek
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model='deepseek-v4-flash',
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(MessagesState):
    username: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "messages": [HumanMessage("你好，我是 " + state["username"])]
    }

def llm_node(state: OverAllState) -> OverAllState:
    res = model.invoke(state["messages"])

    return {
        "messages": [res],
        "output": res.content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "llm_node")
builder.add_edge("llm_node", END)

graph = builder.compile()
response = graph.invoke({"username": "小黄"})
print(response)
```

<font style="color:rgb(60, 60, 67);">输出如下</font>

```python
{
    "messages": [
        HumanMessage(
            content="你好，我是 小黄",
            additional_kwargs={},
            response_metadata={},
            id="155e1ef4-5bbc-4250-b978-62a7a5918cef",
        ),
        AIMessage(
            content="你好呀，小黄！😊 我是DeepSeek，很高兴认识你！有什么我可以帮你的吗？无论是聊天、解答问题、帮你写作、编程，还是其他任何需要，尽管告诉我吧！你名字里的“黄”是哪个黄呀？😄",
            additional_kwargs={
                "refusal": "None",
            },
            response_metadata={
                "token_usage": {
                    "completion_tokens": 57,
                    "prompt_tokens": 10,
                    "total_tokens": 67,
                    "completion_tokens_details": "None",
                    "prompt_tokens_details": {
                        "audio_tokens": "None",
                        "cached_tokens": 0,
                    },
                    "prompt_cache_hit_tokens": 0,
                    "prompt_cache_miss_tokens": 10,
                },
                "model_provider": "deepseek",
                "model_name": "deepseek-v4-flash",
                "system_fingerprint": "fp_8b330d02d0_prod0820_fp8_kvcache_20260402",
                "id": "17d57633-2ffb-4af3-be87-b3d58f9acd2b",
                "finish_reason": "stop",
                "logprobs": "None",
            },
            id="lc_run--019e6890-4544-7212-b4f2-aa20fc079911-0",
            tool_calls=[],
            invalid_tool_calls=[],
            usage_metadata={
                "input_tokens": 10,
                "output_tokens": 57,
                "total_tokens": 67,
                "input_token_details": {
                    "cache_read": 0,
                },
                "output_token_details": {},
            },
        ),
    ],
    "username": "小黄",
    "output": "你好呀，小黄！😊 我是DeepSeek，很高兴认识你！有什么我可以帮你的吗？无论是聊天、解答问题、帮你写作、编程，还是其他任何需要，尽管告诉我吧！你名字里的“黄”是哪个黄呀？😄",
}
```

上述案例中，`OverAllState` 继承了 `MessagesState`，因此可用的状态字段为

```python
messages
username
output
```

  
 只关注 `messages` 状态，执行流程如下：

1. `node_a` 返回一条 `HumanMessage`；
2. `LangGraph` 使用 `add_messages` 将该消息合并到 `messages` 状态字段中；
3. `llm_node` 从 `state["messages"]` 中读取完整消息列表，并调用模型；
4. `llm_node` 将模型生成的 `AIMessage` 作为状态更新返回；
5. `LangGraph` 再次通过 `add_messages` 将 `AIMessage` 合并到 `messages` 中；
6. 最终状态中包含完整消息列表。

`MessagesState` 帮开发者预先定义好了 `messages` 字段及其合并规则。在构建聊天机器人、Agent、工具调用流程、多轮对话流程时，它可以提升开发效率。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

### AgentState
> 这是LangChain agent 的内部状态，不推荐在LangGraph中使用
>

`AgentState` 是 `LangChain Agent` 内部使用的状态类型。由于 `LangChain Agent` 底层也是基于 `LangGraph` 运行图构建的，所以从技术上讲，**<font style="color:#DF2A3F;">开发者也可以将 </font>**`**<font style="color:#DF2A3F;">AgentState</font>**`**<font style="color:#DF2A3F;"> 或其子类作为自定义 </font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 的状态类型。</font>**

`AgentState` 的全类名，也可以称为类的完全限定名（英文全称： `fully qualified class name`）是：`langchain.agents.middleware.types.AgentState`

```python
class AgentState(TypedDict, Generic[ResponseT]):
    """State schema for the agent."""

    messages: Required[Annotated[list[AnyMessage], add_messages]]
    jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
    structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
```

<font style="color:rgb(60, 60, 67);"></font>

该状态中主要包含三个字段。

1. messages

```python
messages: Required[Annotated[list[AnyMessage], add_messages]]
```

`messages` 用于存储 Agent 运行过程中的消息列表。

该字段和 `MessagesState` 中的 `messages` 字段类似，也使用 `add_messages` 作为 `Reducer`。





2. jump_to

```python
jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
```

`jump_to` 是 `LangChain Agent` 内部使用的控制字段，主要服务于 Agent 中间件体系。

它通常用于表示运行流程的跳转意图，例如某些中间件希望影响 Agent 后续应该进入哪个节点。

需要注意的是，`jump_to` 并不是普通 LangGraph 状态图中的通用跳转机制。

在自定义 `StateGraph` 中，**<font style="color:#DF2A3F;">即使状态中定义了 </font>**`**<font style="color:#DF2A3F;">jump_to</font>**`**<font style="color:#DF2A3F;"> 字段，</font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 也不会因为该字段的值自动跳转到某个节点</font>**。普通 `LangGraph` 运行图如果需要控制后续流向，通常应使用：

```python
Command(goto="node_name")
```







3. structured_response

```python
structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
```

`structured_response` 用于存储 Agent 最终生成的结构化输出。

当使用 `LangChain Agent` 的结构化输出能力时，例如指定 `response_format`，Agent 最终生成的结构化结果通常会被写入该字段。

其中，`**<font style="color:#DF2A3F;">OmitFromInput</font>**`**<font style="color:#DF2A3F;"> 表示该字段不应作为外部输入字段暴露给调用方</font>**，而是由 Agent 运行过程中内部生成。

总体来看，`**<font style="color:#DF2A3F;">AgentState</font>**`**<font style="color:#DF2A3F;"> 是专门为 LangChain Agent 运行时设计的状态类型</font>**。

因此，在普通自定义 `LangGraph` 项目中，一般不建议直接基于 `AgentState` 扩展图状态。











# 控制流
## 顺序结构
### add_edge
`add_edge` 用于在两个节点之间添加一条有向边。边是图结构中最基本的元素之一，节点之间的执行顺序、分支跳转以及循环控制，最终都依赖节点和边共同表达。

因此，在 LangGraph 中，**<font style="color:#DF2A3F;">基础的控制流结构都可以通过 </font>**`**<font style="color:#DF2A3F;">add_edge</font>**`**<font style="color:#DF2A3F;"> 进行构建。</font>**

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class OverAllState(TypedDict):
    username: str
    greeting: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "greeting": "Dear " + state["username"]
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "output": state["greeting"] + "，你好！"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
res = graph.invoke({"username": "小黄"})
print(res)

from IPython.display import display

display(graph)
```



```python
{
  "username": "小黄",
  "greeting": "Dear 小黄",
  "output": "Dear 小黄，你好！"
}
```

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791881461467-c2c60a50-e741-42e3-8b1d-2e1d0964b1dd.png)



其中：

+ `START -> node_a` 表示图从 `node_a` 开始执行；
+ `node_a -> node_b` 表示 `node_a` 执行完成后，继续执行 `node_b`；
+ `node_b -> END` 表示 `node_b` 执行完成后，图进入结束状态。









### add_sequence
如果需要构建一组按顺序执行的节点，也可以使用 `add_sequence`。

`add_sequence` 支持传入一个可执行对象列表。LangGraph 会按照列表顺序依次添加节点，并在相邻节点之间自动添加边。默认情况下，函数名会被用作节点名称。

<font style="color:rgb(60, 60, 67);"></font>

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class OverAllState(TypedDict):
    username: str
    greeting: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "greeting": "Dear " + state["username"]
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "output": state["greeting"] + "，你好！"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_edge(START, "node_a")
builder.add_sequence([node_a, node_b])
builder.add_edge("node_b", END)

graph = builder.compile()
res = graph.invoke({"username": "小黄"})
print(res)

from IPython.display import display

display(graph)
```



```python
{
  "username": "小黄",
  "greeting": "Dear 小黄",
  "output": "Dear 小黄，你好！"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791881566885-44bcb95b-7016-453a-8664-56d03fca8ea5.png)



```python
builder.add_sequence([node_a, node_b])
```



大致等价于：

```python
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge("node_a", "node_b")
```



因此，完整执行流程仍然是：

```python
START -> node_a -> node_b -> END
```

相比手动调用多次 `add_node` 和 `add_edge`，`**<font style="color:#DF2A3F;">add_sequence</font>**`**<font style="color:#DF2A3F;"> 更适合用于构建简单的线性执行流程。</font>**







### 省略指向 `END` 的边
在 LangGraph 中，**<font style="color:#DF2A3F;">图的终止并不完全依赖某个真实执行的特殊节点。</font>**

从运行机制上看，LangGraph 会在每个 `SuperStep` 开始时，根据当前发生更新的 `Channel` 以及节点之间的触发关系，计算本轮需要执行的任务列表。如果没有新的节点被激活，**<font style="color:#DF2A3F;">也就没有新的任务需要执行，图运行自然结束。</font>**

需要注意的是，`E**<font style="color:#DF2A3F;">ND</font>**`**<font style="color:#DF2A3F;"> 并不是运行阶段真正执行的节点</font>**。也就是说，指向 `END` 的边不会像指向普通节点的边那样，触发一个真实的节点任务。**<font style="color:#DF2A3F;">它更多用于表达图结构中的“终止语义”</font>**：当前路径执行到这里即可结束。

因此，在一些简单的线性流程中，即使省略指向 `END` 的边，最后一个节点执行完成后，如果没有后续节点被触发，图也可以正常结束。



示例如下：

```python
from langgraph.graph import StateGraph, START
from typing import TypedDict

class OverAllState(TypedDict):
    username: str
    greeting: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "greeting": "Dear " + state["username"]
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "output": state["greeting"] + "，你好！"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")

graph = builder.compile()
res = graph.invoke({"username": "小黄"})
print(res)

from IPython.display import display

display(graph)
```



输出如下

```python
{
  "username": "小黄",
  "greeting": "Dear 小黄",
  "output": "Dear 小黄，你好！"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791881846609-c5c545da-8c2f-4172-94f2-250d8ecfb0e5.png)



在这个例子中，虽然没有显式添加：

```python
builder.add_edge("node_b", END)
```

但是 `node_b` 执行完成后，没有新的节点被触发，图运行会自然结束。

不过，**<font style="color:#DF2A3F;">可以省略指向 </font>**`**<font style="color:#DF2A3F;">END</font>**`**<font style="color:#DF2A3F;"> 的边</font>**，并不表示 `END` 没有意义。





**<font style="color:#DF2A3F;">显式添加 </font>**`**<font style="color:#DF2A3F;">END</font>**`**<font style="color:#DF2A3F;"> 边可以让图结构更加完整</font>**，也能更清晰地表达“流程到此结束”的语义。尤其是在分支、条件跳转、循环退出等场景中，显式指向 `END` 通常更利于阅读和维护。

**<font style="color:#DF2A3F;">和 </font>**`**<font style="color:#DF2A3F;">END</font>**`**<font style="color:#DF2A3F;"> 不同，</font>**`**<font style="color:#DF2A3F;">START</font>**`**<font style="color:#DF2A3F;"> 通常不能省略</font>**。因为 `START` 不只是一个语义上的起点标记，它还用于告诉 LangGraph：图运行时应当从哪些节点开始执行。也就是说，下面这条边是启动图执行的关键



```python
builder.add_edge(START, "node_a")
```

**<font style="color:#DF2A3F;background-color:#FBDE28;">如果没有从 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">START</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 出发的边</font>****<font style="color:#DF2A3F;">，或者没有通过其他方式指定入口节点，LangGraph 就无法确定图的初始执行节点。</font>**

因此，在实际开发中，建议保留指向 `END` 的边。虽然在某些简单流程中省略这些边也能正常运行，但显式添加它们，可以让图结构更加完整、语义更加清晰，便于后续维护。









## 分支结构


### 静态分支（Static Branch）
+ 定义：节点的下游候选节点 在图编译阶段就完全确定，只是运行时根据条件选择哪条边执行。
+ 特点：
    - 下游节点集合固定，数量、目标在编译时确定
    - **<font style="color:#DF2A3F;">运行时可选择一个或多个下游目标</font>**
    - 可以用来做条件分支，但不生成新的节点

⚡ 核心判断：

编译期知道**<font style="color:#DF2A3F;">下游集合 → 静态分支</font>**













#### 并行节点
并行节点是最简单的静态分支形式。

当多个节点都从同一个上游节点触发时，**<font style="color:#DF2A3F;">它们会在同一个超步被激活</font>**。典型写法如下：

```python
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
```









##### <font style="color:rgb(60, 60, 67);">示例：两个节点并行执行</font>
```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict
from langchain_deepseek import ChatDeepSeek
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke(
        [
            HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")
        ]
    ).content
    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke(
        [
            HumanMessage(f"写一个关于 {state['topic']} 的笑话")
        ]
    ).content
    return {
        "joke": joke
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")

builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

graph = builder.compile()
res = graph.invoke({"topic": "猫咪"})
print(res)

from IPython.display import display

display(graph)
```

上述案例中，`**<font style="color:#DF2A3F;">node_a</font>**`**<font style="color:#DF2A3F;"> 和 </font>**`**<font style="color:#DF2A3F;">node_b</font>**`**<font style="color:#DF2A3F;"> 都由 </font>**`**<font style="color:#DF2A3F;">START</font>**`**<font style="color:#DF2A3F;"> 触发。</font>****<font style="color:#DF2A3F;background-color:#FBDE28;">图运行时，二者会在同一个超步中被调度</font>****<font style="color:#DF2A3F;">，它们各自读取当前状态并独立执行。</font>**

需要注意的是：

+ 这里的“并行”主要指调度语义上的并行；两个节点之间没有先后依赖；
+ 它们的输出会在当前超步执行完成后统一合并到状态中；
+ 如果两个节点写入同一个状态字段，则该字段通常需要配置合适的 `Reducer`，否则可能出现状态更新冲突，抛出 `InvalidUpdateError` 异常



```python
{
    "topic": "猫咪",
    "poem": "《猫咪》\n夜巡檐角步轻悄，昼卧花阴晒暖毛。\n偶逐流莺穿柳过，忽惊蝴蝶上眉梢。\n\n赏析：这首作品以猫咪日常为切入点，通过“夜巡檐角”、“昼卧花阴”展现其昼伏夜出的习性，“逐流莺”、“惊蝴蝶”则捕捉其灵动瞬间。全诗未著“猫”字，却以“轻悄步”、“晒暖毛”等细节勾勒出猫咪的形神特征，尾句“上眉梢”更将好奇神态融入蝴蝶意象，物我交融，余韵悠长。",
    "joke": "有一只猫走进了一家披萨店，店员问：“您好，请问您要什么？”\n猫淡定地说：“喵～（我要一个12寸的海鲜披萨，加双倍芝士。）”\n店员愣了愣，说：“猫先生，您确定？”\n猫点点头，店员只好照做，然后猫掏出钱包付了钱，趴在小桌子上优雅地吃了起来。\n\n这时，旁边桌的客人看呆了，忍不住问店员：“它怎么能像人一样点餐、付钱、吃披萨？”\n店员耸耸肩，小声说：“我也不知道，但自从它上次用我的电脑打了一局《猫里奥》通关后，我就没敢拒绝它的任何要求。”"
}
```



<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791882232301-f1d49d7b-6e5d-44be-94ad-9756d90e3362.png)





#### 条件分支
`StateGraph` 提供了<font style="background-color:#FBDE28;"> </font>`**<font style="color:#DF2A3F;background-color:#FBDE28;">add_conditional_edges</font>**`**<font style="color:#DF2A3F;"> 方法，用于从某个上游节点出发，根据运行时状态选择下游节点。</font>**

```python
def add_conditional_edges(
    self,
    source: str,
    path: Callable[..., Hashable | Sequence[Hashable]]
    | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
    | Runnable[Any, Hashable | Sequence[Hashable]],
    path_map: dict[Hashable, str] | list[str] | None = None,
) -> Self:
```

不考虑 `self`，核心参数有三个：

+ `source`：条件分支的起始节点；
+ `path`：路由规则，是一个可执行对象，通常是函数
+ `path_map`：路由规则的返回值到真实节点名之间的映射关系。

其中，`path` 的返回值表示跳转的目标节点，可以是：

+ 字符串或特殊对象 `END` 表示的单个目标；
+ 字符串或特殊对象 `END` 表示的多个目标组成的序列；

`path_map`

+ 可以省略，即取默认值 `None`，此时 `path` 返回值中出现的字符串必须是合法的节点名称。
+ 可以是字典，**<font style="color:#DF2A3F;">维护 </font>**`**<font style="color:#DF2A3F;">path</font>**`**<font style="color:#DF2A3F;"> 返回值和真实节点的映射。</font>**
+ 也可以是列表，如下

```python
path_map=["node_a", "node_b", "node_c"]
```



相当于

```python
path_map={
    "node_a": "node_a",
    "node_b": "node_b",
    "node_c": "node_c",
}
```

此时也要求 `path` 返回值中出现的字符串必须是合法的节点名称。









##### <font style="color:rgb(60, 60, 67);">不使用 </font>`<font style="color:rgb(60, 60, 67);background-color:rgba(142, 150, 170, 0.14);">path_map</font>`
如果不传 `path_map`，那么路由函数的返回值通常应当直接是图中的节点名称。如下

```python
def router(state: OverAllState) -> Literal["node_a", "node_b"]: 
    if "诗" in state["content_type"]: 
        return "node_a" 
    return "node_b"
```

此时，`router` 返回的 `"node_a"` 和 `"node_b"` 必须能够直接对应图中已经注册的节点名。

  
 

完整案例如下

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    content_type: str
    poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def router(state: OverAllState) -> Literal["node_a", "node_b"]:
    if "诗" in state["content_type"]:
        return "node_a"
    return "node_b"

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_conditional_edges(START, router)
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})

print('=' * 30, '-> poem_res <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke_res <-', '=' * 30)
print(joke_res)

from IPython.display import display

display(graph)
```

  
 输出如下

```python
============================== -> poem_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "诗",
    "poem": "《咏布偶狗》\n绒身静卧似憨眠，无骨何妨守户前。\n不吠晨昏随主步，垂髫怜抱共跚连。\n\n注：我的诗中“无骨”暗喻布偶狗没有生命骨架，却依然守护着家门；“跚连”形容孩童与布偶狗相偕学步的蹒跚之态。通过“绒身”“垂髫”等意象，既保留了布偶狗的柔软特质，又赋予其超越玩偶的温情守护之意，在静态与动态的转换间实现了拟人化的诗意升华。"
}

============================== -> joke_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "笑话",
    "joke": "有一天，一只布偶狗在街上闲逛，遇到了一只真狗。\n\n真狗好奇地问：“你是什么品种？怎么看起来毛这么长、这么滑，眼神还这么呆？”\n\n布偶狗骄傲地挺起胸膛：“我可是最顶级的仿真布偶狗！纯手工缝制，水晶眼睛，进口棉花填充，咬不坏、摔不烂，还不用溜。”\n\n真狗愣了一下，问：“那你平时都干嘛？”\n\n布偶狗叹了口气：“看家呗……主人说我最大的优点就是——丢不了，谁捡了都想还回来，因为太假了，连肉都不香。”"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791882613290-896bb876-14da-4184-a607-21377e33fea3.png)

观察图结构可以发现，`__start__`指向`node_a`和`node_b`的线是虚线，这表示节点的跳转是有条件的，不是固定的



##### <font style="color:rgb(60, 60, 67);">使用 </font>`<font style="color:rgb(60, 60, 67);background-color:rgba(142, 150, 170, 0.14);">path_map</font>`
如果不希望路由函数直接返回节点名，而是返回业务语义更强的标识，可以使用 `path_map` 进行映射。如下：

```python
def router(state: OverAllState) -> Literal["a", "b"]: 
    if "诗" in state["content_type"]: 
        return "a" 
    return "b"

builder.add_conditional_edges( 
    START,
    router,
    path_map={ 
        "a": "node_a", 
        "b": "node_b", 
    } 
)
```

此时返回的 `"a"` 和 `"b"` 可以不是图中已注册的节点名称，但要通过 `path_map` 映射到正确的节点

这种写法的好处是：

+ 路由函数可以返回业务含义更清晰的标签；
+ 图节点名称可以保持工程化命名；
+ 渲染图结构时，边上可以显示路由标签，使图更容易理解。



示例如下

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    content_type: str
    poem: str
    ci_poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def router(state: OverAllState) -> Literal["a", "b"]:
    if "诗" in state["content_type"]:
        return "a"
    return "b"

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_conditional_edges(
    START,
    router,
    path_map={
        "a": "node_a",
        "b": "node_b",
    }
)
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})

print('=' * 30, '-> poem_res <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke_res <-', '=' * 30)
print(joke_res)

from IPython.display import display

display(graph)
```

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);">输出如下</font>**

```python
============================== -> poem_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "诗",
    "poem": "《布偶狗》\n布偶妆成似犬形，绒毛细软眼如星。\n不吠不跳乖巧坐，疑是仙童降画屏。\n\n注：我的仿写创作思路是以玩具布偶狗为意象，通过“妆成似犬形”点明其拟态特征。后三句运用比喻（眼如星）、拟人（乖巧坐）、想象（仙童降画屏）手法，赋予静态玩偶动态神韵。末句以“仙童”暗喻其灵动气质，呼应首句的“布偶”属性，使全诗在虚实之间形成童趣与仙气的张力。"
}

============================== -> joke_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "笑话",
    "joke": "好的，这里有一个关于布偶狗的笑话：\n\n有一只布偶狗，它从来不叫，也从来不跑。它的主人很担心，就带它去看兽医。\n\n兽医检查了半天，推了推眼镜说：“你这狗啊，没别的毛病，就是太乖了，乖到连狗的身份都忘了。你知道它为什么这么安静吗？”\n\n主人摇摇头。\n\n兽医叹了口气，说：“因为它是个‘布偶’，没有嘴，也没有腿。它唯一的技能，就是等你给它配个‘程序’，然后坐在那里，假装自己是一只不会乱咬东西的电子狗。说真的，这笑话比它还冷。”"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791882878092-0820a711-8010-41e8-9435-84d74d05dd05.png)

观察图结构可以发现，`__start__`指向`node_a`和`node_b`的虚线上出现了文本`a`和`b`，它们是路由函数返回的名称，通过`path_map`映射到具体的下游节点。



##### 同时路由至多个节点
`add_conditional_edges` 也支持一次路由到多个下游节点。



###### 不用`path_map`
```python
from collections.abc import Sequence
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    content_type: str
    poem: str
    ci_poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def node_c(state: OverAllState) -> OverAllState:
    ci_poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的词")]).content

    return {
        "ci_poem": ci_poem
    }

def router(state: OverAllState) -> Sequence[Literal["node_a", "node_b", "node_c"]]:
    if "诗" in state["content_type"]:
        return ["node_a", "node_c"]
    return ["node_b", "node_c"]

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_conditional_edges(
    START,
    router
)
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)
builder.add_edge("node_c", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})

print('=' * 30, '-> poem_res <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke_res <-', '=' * 30)
print(joke_res)

from IPython.display import display

display(graph)
```

在这个例子中：

+ 当 `content_type` 包含 `"诗"` 时，同时触发 `node_a` 和 `node_c`；
+ 否则同时触发 `node_b` 和 `node_c`。







```python
============================== -> poem_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "诗",
    "poem": "《咏布偶狗》\n铁骨棉裘本异俦，垂头摇尾学温柔。\n非关皮相失真性，恐向人间惹旧愁。\n\n赏析：这首作品以布偶狗为题，通过“铁骨棉裘”的悖论意象，巧妙揭示材质与形态的错位。后两句笔锋一转，从拟态之“学温柔”深入至本真之“失真性”，结句“恐向人间惹旧愁”更以物喻人，赋予玩偶以灵性，暗含对世情虚伪的讽喻，余韵悠长。",
    "ci_poem": "《捣练子·布偶狗》\n针线巧，布绒柔。静卧妆台伴月幽。未解铃铛摇夜语，却随清梦上云舟。\n\n注：我的仿作将布偶狗拟人化，通过“针线巧，布绒柔”展现其手工质感，“伴月幽”营造静谧氛围。末句“随清梦上云舟”暗合现代人渴望逃离现实的隐喻，既保留古典词牌的意境美，又赋予布偶狗超越物象的情感寄托。"
}

============================== -> joke_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "笑话",
    "ci_poem": "《鹧鸪天·布偶狗》\n绒目含星尾作虹，垂蹄静卧笑春风。\n无人自戏绒球转，得主频摇铃铎浓。\n\n偎暖日，扑帘栊。娇憨总在爪痕中。\n痴心不解浮生事，一抱虚温万事空。\n\n注：本词以布偶狗为意象，通过“绒目含星”、“垂蹄静卧”等细节摹写其柔顺之态。下阕“偎暖日”、“扑帘栊”拟人化笔法，寄寓物我相忘之趣。结句“一抱虚温”暗喻繁华终归寂灭之理，将玩偶之趣升华为对生命本真的哲思。",
    "joke": "小明带他的布偶狗去宠物医院，医生检查后说：“你这狗没心跳没呼吸，已经死了。”\n小明急了：“不可能！它刚刚还对我摇尾巴！”\n医生叹了口气：“那是你兜里的钥匙串，走起路来叮当响，它尾巴上的铃铛在共振。”\n小明沉默片刻，掏出手机拍了张照发朋友圈：“我的狗死了，但它的尾巴还在听我走路。”\n评论区炸了：“建议主人也去查查脑子，可能也共振坏了。”"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791883850346-3d0850f2-8954-435c-a846-cd50c828ac39.png)



观察图结构可以发现，`node_a`、`node_b`和`node_c`独立于图结构之外。

和上一节案例相比，`router`函数返回的是序列而非单个节点，渲染器无法推断节点间的映射关系。



######   
 添加映射
通过 `path_map` 显示声明映射关系，明确下游节点集合，在提升代码可读性的同时，也有助于渲染器正确展示条件边。

当前场景下 `path` 返回值中的字符串就是合法的节点名称，`path_map` 可以是字典

```python
path_map={
    "node_a": "node_a",
    "node_b": "node_b",
    "node_c": "node_c",
}
```

也可以是列表

```python
path_map=["node_a", "node_b", "node_c"]
```



**<font style="color:rgb(60, 60, 67);">完整案例如下</font>**

```python
from collections.abc import Sequence
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    content_type: str
    poem: str
    ci_poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def node_c(state: OverAllState) -> OverAllState:
    ci_poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的词")]).content

    return {
        "ci_poem": ci_poem
    }

def router(state: OverAllState) -> Sequence[Literal["node_a", "node_b", "node_c"]]:
    if "诗" in state["content_type"]:
        return ["node_a", "node_c"]
    return ["node_b", "node_c"]

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_conditional_edges(
    START,
    router,
    path_map={
        "node_a": "node_a",
        "node_b": "node_b",
        "node_c": "node_c",
    }
    # path_map=["node_a", "node_b", "node_c"]
)
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)
builder.add_edge("node_c", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})

print('=' * 30, '-> poem_res <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke_res <-', '=' * 30)
print(joke_res)

from IPython.display import display

display(graph)
```



输出如下

```python
============================== -> poem_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "诗",
    "poem": "《布偶狗》\n布偶憨然卧锦茵，牵绳随主步香尘。\n偶因客至微昂首，原是无声座上宾。\n\n注：我的创作思路是抓住布偶狗静中有动的特质。首句以“憨然卧锦茵”勾勒其静态萌态，次句“牵绳随主步香尘”暗喻宠物与人的陪伴关系。转句“偶因客至微昂首”捕捉其反应，既保留布偶的拟真特性，又暗示机械性动作。结句“原是无声座上宾”点明其非生物属性，与开篇“布偶”形成闭环。全诗通过动静、真假的微妙转化，赋予静物以灵性。",
    "ci_poem": "《醉花阴·布偶狗》\n绒耳垂垂星作眸，憨卧小窗幽。棉絮裹憨态，线描眉眼，缝就三分柔。\n旧尘不染春衫袖，伴我度春秋。虽无温舌语，月斜人静，也解倚床头。\n\n注：我以拟人笔法赋予布偶狗灵性，通过“绒耳垂垂”“棉絮裹憨态”等细节勾勒其柔软形态，又于“月斜人静”处暗藏陪伴深情。下阕“旧尘不染”暗喻其纯净守护，末句“解倚床头”更将静物写活，道出玩偶与主人之间无声的慰藉。"
}

============================== -> joke_res <- ==============================

{
    "topic": "布偶狗",
    "content_type": "笑话",
    "ci_poem": "《南歌子·布偶狗》\n绒耳垂春水，圆睛印月痕。\n棉絮缝成憨态真。\n不向朱门摇尾，卧蓬门。\n\n旧线牵云袖，新绒补雪身。\n线头散作泪斑纹。\n但把残年缝进，掌中温。\n\n注：我的布偶狗，是外婆用旧棉裤的棉花和二姐穿小的灯芯绒外套一针一线缝成的。二十年过去，棉絮从它破了洞的耳朵里露出，像吐着舌头。我模仿陆游“我与狸奴不出门”的意境，通过“棉絮缝春水”喻指时光在针脚里流逝，而“线头泪斑”则是故意缝补的裂痕——我们总要在破碎的事物里，重新认出圆满的形状。",
    "joke": "好的，这是一个关于布偶狗的笑话：\n\n有一只布偶狗，它特别特别懒，懒到什么程度呢？它的主人叫它去捡飞盘，它都懒得动。主人决定训练它，就扔出一个飞盘，大声喊：“布偶，快去捡！”\n\n布偶狗懒洋洋地抬头看了一眼飞盘落在远处，然后又趴下了。主人很生气，又扔了一个，它还是不理。\n\n主人无奈，只好采取了终极手段——假扮成一只会动的布偶狗，自己冲过去把飞盘叼了回来。\n\n布偶狗一看，浑身毛发都惊得炸了起来，它对旁边的一只小花狗低声说：“天哪，你看，那玩意儿居然能自己动！它还是个永动机！吓死狗了，好赖哦！”"
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791883998613-7ddf70db-69d9-4cbe-9cc4-e68f1b46f817.png)

如图所示，图结构被正确渲染。





#### defer node execution： 延迟执行


某些情况下，我们希望在所有常规任务节点执行完毕后，再进行日志、审计等收尾工作

此时可以在添加节点时设置`defer=True`，如下：

```python
builder.add_node("audit_node", audit_node, defer=True)
```

`defer=True` 的含义是：

当前节点不会在其被触发后立即执行，**<font style="color:#DF2A3F;">而是被延迟到常规图运行流程结束后</font>**，**<font style="color:#DF2A3F;">再在额外的超步中触发执行。</font>**



这类节点适合用于：

+ 日志记录；
+ 审计检查；
+ 结果汇总；
+ 收尾清理；
+ 统一校验前面节点是否已完成。



##### 底层实现机制
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791884065824-69e47dab-3d18-4e04-aa85-7afc5252b8c1.png)



###### 编译阶段：使用特殊 Channel
1. `LangGraph` 在编译状态图时，会为边创建对应的 `Channel`。
2. 此时会根据节点的 `defer` 属性创建不同类型的 `Channel`，如下。

```python
self.channels[branch_channel] = (
    LastValueAfterFinish(Any)
    if node.defer
    else EphemeralValue(Any, guard=False)
)
```

`defer` 默认值为 `False`

对于普通节点，边对应的通道类型是 `EphemeralValue`，可以理解为普通临时通道；而对于 `defer=True` 的节点，边对应的通道类型是特殊的 `LastValueAfterFinish`。





###### 常规运行阶段-写入但不触发
1. 在图运行过程中，每个节点执行完成后，会向其下游边对应的 `Channel` 写入数据。
2. 常规的 `Channel` 在运行开始后处于可用状态，被写入后记录在 `updated_channels` 列表中，从而在下一个超步中触发下游节点的执行。
3. 但是，`LastValueAfterFinish` 类型的通道起初是不可用的，首次被写入时不会添加到 `updated_channels` 列表中，下游节点自然不会被触发。







###### 常规流程结束后：调用`finish()`唤醒延迟节点
1. `LangGraph` 底层用 `trigger_to_nodes` 维护了 边的 `Channel` -> 节点 的映射，是一个字典。
2. 在每个超步结束后，运行时会根据 `updated_channels` 判断是否还有新的节点需要被触发。
3. 如果 `updated_channels` 和 `trigger_to_nodes` 的 key 没有交集，说明当前没有新的普通节点需要继续执行，常规运行流程已结束。
4. 此时，`LangGraph` 运行时会调用所有 `Channel` 的 `finish()` 方法。
5. 对于普通 `Channel` ，`finish()` 通常不会产生新的触发效果；但对于 `LastValueAfterFinish` 类型通道，首次调用 `finish()` 时，会将内部的 `finished` 标记设置为 `True`，并返回 `True`。
6. 一旦 `finished=True`，该通道的 `is_available()` 就会变为 `True`。
7. 于是，原本被延迟的通道会被加入 `updated_channels`，从而在额外的超步中触发对应的 `defer` 节点。







###### 总结
因此，`defer=True` 的运行机制可以概括为：

触发边的 `Channel` 为特殊类型，首次写入不触发；常规流程结束后，**<font style="color:#DF2A3F;background-color:#FBDE28;">特殊通道 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">finish()</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">；通道变为可用</font>****<font style="color:#DF2A3F;">；从而触发延迟节点，后者在额外超步中执行。</font>**

<font style="color:rgb(103, 103, 108);"></font>

##### <font style="color:rgb(60, 60, 67);">案例</font>
```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek
from loguru import logger

from dotenv import load_dotenv

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def audit_node(state: OverAllState) -> OverAllState:
    logger.info(f"任务节点全部执行完毕，诗 {'已生成' if state['poem'] else '未生成'}，笑话 {'已生成' if state['joke'] else '未生成'}")

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("audit_node", audit_node, defer=True)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
builder.add_edge(START, "audit_node")
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)
builder.add_edge("audit_node", END)

graph = builder.compile()
res = graph.invoke({"topic": "布偶狗"})
print(res)

from IPython.display import display

display(graph)
```

在这个图中：

+ `node_a` 和 `node_b` 是普通节点，在常规流程中被触发；
+ `audit_node` 虽然也由 `START` 触发，但由于设置了 `**<font style="color:#DF2A3F;">defer=True</font>**`**<font style="color:#DF2A3F;">，不会立即执行；</font>**
+ `node_a` 和 `node_b` 都执行完成后，常规运行流程结束，`audit_node` 才会在额外的超步中执行；
+ 因此，`audit_node` 中可以读取到 `node_a` 和 `node_b` 已经写入并提交后的状态。

<font style="color:rgb(103, 103, 108);"></font>

```python
2026-05-28 17:07:09.618 | INFO | __main__:audit_node:40 - 任务节点全部执行完毕，诗 已生成，笑话 已生成

{
    'topic': '布偶狗',
    'poem': '《咏布偶狗》\n'
            '绸身棉腑自憨娇，静卧无眠守寂寥。\n'
            '不吠不嗔街巷事，偎人一晌解无聊。\n'
            '\n'
            '注：我的创作思路是通过拟人化手法，赋予布偶狗以温柔静默的品格。'
            '首句以“绸身棉腑”点明材质，次句“守寂寥”暗合宠物陪伴的特质。'
            '后两句通过“不吠不嗔”与“解无聊”的对比，凸显布偶狗无声胜有声的治愈力量，'
            '在机械玩物中注入人文温度。',
    'joke': '这是一个关于布偶狗的笑话：\n'
            '\n'
            '有一只布偶狗，它什么都好，毛茸茸的，蓝眼睛，性格温顺，'
            '唯一的缺点就是——它是个布偶，不会动。\n'
            '\n'
            '它的主人每天都把它放在沙发上，假装它在看电视、在思考狗生。\n'
            '\n'
            '有一天，主人带了一只真的活狗回家，想让布偶狗有个伴。'
            '活狗非常活泼，对着布偶狗又叫又摇尾巴，但布偶狗一动不动。\n'
            '\n'
            '活狗很奇怪，绕着布偶狗转了三圈，然后凑上去闻了闻。\n'
            '\n'
            '最后，活狗叹了口气，对主人说：\n'
            '\n'
            '“主人，这只狗……是不是在用‘静默模式’跟我交流？'
            '它的蓝牙是不是断了？”'
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791948387335-1bd595e8-8267-49fd-aa8c-62f650375802.png)





##### 小结
`defer=True` 适合“最后执行”的收尾节点。

它的本质是通过特殊的 `Channel` 控制触发时机：

1. 编译阶段为延迟节点创建 `LastValueAfterFinish` 类型通道；
2. 常规运行阶段该通道可以被写入，但不可用，不会触发下游节点；
3. 常规流程结束后调用通道的 `finish()` 方法；
4. `**<font style="color:#DF2A3F;">finish()</font>**`**<font style="color:#DF2A3F;"> 首次生效后将通道标记为可用；</font>**
5. **<font style="color:#DF2A3F;">延迟节点在额外超步中被触发执行。</font>**

因此， `defer=True` 将节点的触发信号延迟释放，使该节点在常规流程结束后执行。









### 动态分支（Dynamic Branch）
+ 定义：节点的后续执行路径在 运行时 才确定，可以根据当前状态、输入数据或中间结果，动态决定要触发哪些下游任务或跳转到哪个下游节点。
+ 特点：
    - 下游执行目标可以在运行时选择；
    - 下游任务数量可以在运行时决定；
    - 可以为同一个下游节点动态创建多个执行任务；
    - 适合实现 Map-Reduce 式动态扇出、多任务并行处理、运行时条件跳转等场景。

需要注意的是，所谓“动态”通常不是指运行时临时创建新的节点定义。节点本身一般仍需要在图编译前注册。动态性主要体现在：运行时决定触发哪些节点，以及为这些节点创建多少个执行任务。

+ 典型用法：
    - `Send`（动态扇出任务/数据）
    - `Command(goto=...)`（运行时跳转到下游节点）



核心判断：

运行时决定后续执行目标或任务数量 → 动态分支

<font style="color:rgb(103, 103, 108);"></font>

<font style="color:rgb(103, 103, 108);"></font>

<font style="color:rgb(103, 103, 108);"></font>

#### 并行节点
`Send` 结合 `**<font style="color:#DF2A3F;background-color:#FBDE28;">add_conditional_edges()</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 使用</font>**，可以用于动态扇出任务

所谓**<font style="color:#DF2A3F;">扇出（Fan-out），是指一个上游节点像扇子一样，向外分发出多个下游任务</font>**。例如，根据一个主题，同时生成诗、词、笑话三个任务；又如，根据一个列表，为列表中的每个元素动态启动一个处理任务。

具体来说，路由函数可以返回一个 `Send` 实例序列。**<font style="color:#DF2A3F;">每个 </font>**`**<font style="color:#DF2A3F;">Send</font>**`**<font style="color:#DF2A3F;"> 实例都描述了一次独立的任务分发：</font>**

+ 分发到哪个下游节点；
+ 给这个下游节点传入什么私有状态。

运行时，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 会根据返回的每个 </font>**`**<font style="color:#DF2A3F;">Send</font>**`**<font style="color:#DF2A3F;"> 实例创建对应的任务。这些任务通常会在同一个超步中并行执行。</font>**



```python
def __init__(self, /, node: str, arg: Any) -> None:
    """
    Initialize a new instance of the `Send` class.

    Args:
        node: The name of the target node to send the message to.
        arg: The state or message to send to the target node.
    """
    self.node = node
    self.arg = arg
```

该构造器接收两个参数，并记录为实例属性：

+ node：待启动的下游节点名称，必须是已在状态图中注册的合法名称
+ arg：传递给下游节点的信息，仅对`node`指向的节点可见，通常应是私有状态。每个 `Send` 实例接收到的 `arg` 是独立的，互不相干。







```python
from typing import TypedDict, Literal
from collections.abc import Sequence
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv
load_dotenv(override=True)

CONTENT_TYPES = ["poem", "ci_poem", "joke"]

model = ChatDeepSeek(
    model = "deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    ci_poem: str
    joke: str

class WorkerState(TypedDict):
    """
    私有状态，只对 Worker 节点可用
    """
    content_type: Literal["poem", "ci_poem", "joke"]
    prompt: str

class InputState(TypedDict):
    topic: str

class OutputState(TypedDict):
    poem: str
    ci_poem: str
    joke: str

def worker_node(state: WorkerState) -> OutputState:
    content_type = state["content_type"]
    prompt = state["prompt"]

    content = model.invoke([HumanMessage(prompt)]).content
    return {
        content_type: content
    }

def router(state: InputState) -> Sequence[Send]:
    prompt = "请生成关于 {} 的 {}"
    english2chinese = {
        "poem": "一首诗",
        "ci_poem": "一首词",
        "joke": "一个笑话"
    }

    topic = state["topic"]

    return [Send(
            "worker_node",
            {
                "content_type": content_type,
                "prompt": prompt.format(topic, english2chinese[content_type]),
            }
        ) for content_type in CONTENT_TYPES
    ]

builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("worker_node", worker_node)
builder.add_conditional_edges(
    START,
    router,
    path_map=["worker_node"]
)
builder.add_edge("worker_node", END)

graph = builder.compile()
res = graph.invoke({"topic": "布偶狗"})
print(res)

from IPython.display import display
display(graph)
```

在这个案例中，`router()` 并没有直接返回某一个固定的下游节点名称，而**<font style="color:#DF2A3F;">是返回了多个 </font>**`**<font style="color:#DF2A3F;">Send</font>**`**<font style="color:#DF2A3F;"> 实例：</font>**

```python
[ 
    Send("worker_node", {"content_type": "poem", ...}), 
    Send("worker_node", {"content_type": "ci_poem", ...}), 
    Send("worker_node", {"content_type": "joke", ...}), 
]
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791948716060-160efffb-c39a-487f-a5cd-b4d774152834.png)

此处图结构能成功渲染的关键，是在 `path_map` 中显式声明了可能被路由到的下游节点：

```python
path_map=["worker_node"]
```

  
 因为 `Send` 的目标和数量可以运行时动态决定，图渲染器无法仅靠运行时返回值提前知道图结构。通过 `path_map` 声明候选下游节点后，渲染图才能正确展示 `START` 到 `worker_node` 的条件边关系。

补充说明：如果多个并行任务写入同一个状态字段，通常需要为该字段定义 reducer，用于合并多个任务的输出。本例中三个任务分别写入 `poem`、`ci_poem`、`joke` 三个不同字段，因此不会发生同一字段的并发合并问题。

<font style="color:rgb(60, 60, 67);"></font>



#### 条件分支
##### Command介绍
`Command` 是 `LangGraph` 中用于控制图执行的多功能原语，它的构造器可以接受四个参数，并记录在同名类属性中：

+ `update`：更新图状态，效果等同于节点直接返回状态更新字典；
+ `goto`：指定节点执行完成后的跳转目标，可用于运行时条件分支。当需要同时更新状态并控制跳转时，比单独使用条件边更合适；
+ `graph`：存在子图时，用于指定跳转发生在哪一层图中，例如从子图跳转到父图；
+ `resume`：用于恢复被中断的图执行，常见于 `human-in-the-loop` 场景。



<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

##### 用`Command`实现条件分支
**<font style="color:#DF2A3F;">可以通过 </font>**`**<font style="color:#DF2A3F;">Command(goto=...)</font>**`**<font style="color:#DF2A3F;"> 在节点内部实现条件分支</font>**。

与 `add_conditional_edges()` 相比，`Command` 更适合“状态更新”和“控制流跳转”需要放在同一个节点返回值中的场景。

例如，一个节点既要更新状态，又要根据当前状态决定下一步跳转目标，就可以返回：

```python
return Command(
    update={"foo": "bar"},
    goto="next_node"
)
```

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

**<font style="color:rgb(60, 60, 67);">示例如下</font>**

```python
from typing import TypedDict, Literal
from langgraph.types import Command
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    content_type: Literal["poem", "joke"]
    poem: str
    joke: str

def router(state: OverAllState) -> Command[Literal["poem_node", "joke_node", END]]:
    content_type = state["content_type"]

    if content_type == "poem":
        return Command (
            goto="poem_node"
        )
    elif content_type == "joke":
        return Command (
            goto="joke_node"
        )
    return Command (
        goto=END
    )

def poem_node(state: OverAllState) -> OverAllState:
    topic = state["topic"]
    prompt = f"请生成一首关于 {topic} 的七言绝句"
    poem = model.invoke([HumanMessage(prompt)]).content

    return {
        "poem": poem
    }

def joke_node(state: OverAllState) -> OverAllState:
    topic = state["topic"]
    prompt = f"请生成一个关于 {topic} 的冷笑话"
    joke = model.invoke([HumanMessage(prompt)]).content

    return {
        "joke": joke
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("router", router)
builder.add_node("poem_node", poem_node)
builder.add_node("joke_node", joke_node)
builder.add_edge(START, "router")
builder.add_edge("poem_node", END)
builder.add_edge("joke_node", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶猫", "content_type": "poem"})
joke_res = graph.invoke({"topic": "布偶猫", "content_type": "joke"})

# 这里故意传入一个不符合 Literal["poem", "joke"] 的值，
# 用来观察运行时兜底分支。
# 注意：Literal 主要服务于静态类型检查，
# 默认不会在 Python 运行时阻止非法值传入。
illegal_res = graph.invoke({
    "topic": "布偶猫",
    "content_type": "illegal"
})

print('=' * 30, '-> poem <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke <-', '=' * 30)
print(joke_res)
print('=' * 30, '-> illegal <-', '=' * 30)
print(illegal_res)

from IPython.display import display
display(graph)
```

上述代码中，`router()` 节点根据 `content_type` 的值决定下一步跳转目标：

+ 当 `content_type == "poem"` 时，跳转到 `poem_node`；
+ 当 `content_type == "joke"` 时，跳转到 `joke_node`；
+ 当传入其他非法值时，跳转到 `END`，图直接结束。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

```python
============================== -> poem <- ==============================
{
    'topic': '布偶猫',
    'content_type': 'poem',
    'poem': '《布偶猫》\n雪砌绒身碧眼秋，一帘幽梦卧云舟。\n忽听银铃摇碎月，轻衔星子入西楼。\n\n（注：此诗通过“雪砌绒身”、“碧眼秋”等意象勾勒布偶猫雪白柔顺的毛发与澄澈眼眸；“卧云舟”、“衔星子”等超现实笔法，赋予其精灵般的气质；末句“入西楼”暗合其优雅步态与神秘秉性，整体形成月光浸染的梦幻意境。）'
}

============================== -> joke <- ==============================
{
    'topic': '布偶猫',
    'content_type': 'joke',
    'joke': '给你分享一个关于布偶猫的冷笑话：\n\n---\n\n有一天，一只布偶猫跑去宠物店里应聘。  \n店员问：“你有什么特长呢？”  \n布偶猫懒洋洋地回答：“我特别软。”  \n店员说：“还有呢？”  \n猫翻个身，露出肚皮：“你看，我连摸都不需要自己动——你只要一伸手，我自己就‘布偶’过去了。”\n\n---\n\n希望能让你会心一笑（或者冷到打个哆嗦）😺❄️'
}

============================== -> illegal <- ==============================
{
    'topic': '布偶猫',
    'content_type': 'illegal'
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791949095126-5b61c15d-8361-4bfd-926e-abd5548f0a44.png)

<font style="color:rgb(60, 60, 67);">这里使用：</font>

```python
Command[Literal["poem_node", "joke_node", END]]
```

作为 `router()` 的返回值类型注解。

注解不能限制 `goto` 在运行时只能写这些值，而是为了让 LangGraph 和类型检查工具知道：这个节点可能跳转到哪些目标。对于图结构渲染来说，这个注解非常重要。如果不声明，渲染出来的图可能无法正确表达 `router` 节点的潜在跳转关系。

需要注意的是，如果某个节点使用 `Command(goto=...)` 控制后续跳转，一般不要再给这个节点额外添加普通下游边。否则，普通边和 `Command` 指定的跳转都可能生效，导致多个下游节点被同时触发。

因此，**<font style="color:#DF2A3F;">本例中只添加：</font>**

```python
builder.add_edge(START, "router")
```

而不添加类似下面这样的边

```python
builder.add_edge("router", "poem_node")
builder.add_edge("router", "joke_node")
```

因为 `router` 的后续跳转已经由 `Command(goto=...)` 决定。<font style="color:rgb(60, 60, 67);">  
</font><font style="color:rgb(60, 60, 67);"> </font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

#### <font style="color:rgb(60, 60, 67);">小结</font>
动态分支有两类典型应用场景：

| 场景 | 典型 API | 说明 |
| --- | --- | --- |
| 动态扇出 | `Send`<br/> + `add_conditional_edges()` | **<font style="color:#DF2A3F;">运行时创建多个任务，常用于 </font>**`**<font style="color:#DF2A3F;">Map-Reduce</font>**` |
| 动态跳转 | `Command(goto=...)` | **<font style="color:#DF2A3F;">节点内部根据状态决定跳转目标</font>** |


二者的区别如下：

+ `**<font style="color:#DF2A3F;">Send</font>**`**<font style="color:#DF2A3F;"> 更强调“一个节点动态分发多个任务实例”；</font>**
+ `**<font style="color:#DF2A3F;">Command(goto=...)</font>**`**<font style="color:#DF2A3F;"> 更强调“当前节点执行完后动态跳转到哪个节点”。</font>**

`Send` 解决的是：**<font style="color:#DF2A3F;">运行时要启动多少个任务实例</font>**。`Command(goto=...)` **<font style="color:#DF2A3F;">解决的是：当前节点执行完后要去哪里。</font>**

不过，无论是 `Send` 还是 `Command(goto=...)`，**<font style="color:#DF2A3F;">目标节点通常都需要提前注册到图中。</font>**所谓“动态”，主要是**<font style="color:#DF2A3F;background-color:#FBDE28;">运行时动态决定任务数量、任务输入或跳转路径</font>**，而不是运行时临时创建新的节点。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

## 多分支汇聚：Fan-in
上文提到了扇出（Fan-out），它是指像打开扇子一样，由一个上游节点分发出多个下游分支。

与之相对，**<font style="color:#DF2A3F;">扇入（Fan-in）是指像合拢扇子一样，多个上游分支汇聚到同一个下游节点。</font>**

Fan-out 对应的是“分支结构”，Fan-in 对应的是“多分支汇聚结构”。





### 静态扇入
在计算机中，“与”表示两个或多个条件同时满足，而“或”表示两个或多个条件任意一个满足。

当多个上游分支同时汇入同一个下游节点时，根据触发条件的不同，可以分成两种情况：

+ “与”触发：上游所有分支全部到达才可触发下游节点
+ “或”触发：任意一个分支到达，都可以触发下游节点

需要注意，这里的 “与 / 或” 只是帮助理解的类比，并不是 `LangGraph` 的官方术语。











#### “与”触发：等待所有上游分支到达
这是多分支汇聚中最常见的情况。

+ `LangGraph` **<font style="color:#DF2A3F;">的计算图是在一系列超步中完成的，如果能看到每个节点实例运行时的超步编号</font>**，则节点执行顺序一目了然。
+ `LangGraph` 的节点可以通过名为 `**<font style="color:#DF2A3F;">config</font>**`**<font style="color:#DF2A3F;"> 的有名参数获取运行时配置</font>**
+ `config` 的元数据中记录了**<font style="color:#DF2A3F;">当前节点所在的 </font>**`**<font style="color:#DF2A3F;">SuperStep</font>**` 序号，可以通过下面的方式获取：

```python
config["metadata"]["langgraph_step"]
```







**<font style="color:rgb(60, 60, 67);">示例如下</font>**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_core.runnables import RunnableConfig

from loguru import logger

class EmptyState(TypedDict):
    pass

def node_a(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_a 被触发", cur_step)
    return {}

def node_b(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_b 被触发", cur_step)
    return {}

def node_c(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_c 被触发", cur_step)
    return {}

def node_d(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_d 被触发", cur_step)
    return {}

def node_e(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_e 被触发", cur_step)
    return {}

builder = StateGraph(state_schema=EmptyState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_node("node_d", node_d)
builder.add_node("node_e", node_e)

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_a", "node_c")
builder.add_edge("node_b", "node_d")
builder.add_edge(["node_c", "node_d"], "node_e")

graph = builder.compile()
graph.invoke({})

from IPython.display import display
display(graph)
```

<font style="color:rgb(60, 60, 67);"></font>

**<font style="color:rgb(60, 60, 67);">运行结果如下</font>**

```python
2026-06-05 14:55:01.705 | INFO     | __main__:node_a:12 - cur_step: 1, node_a 被触发
2026-06-05 14:55:01.708 | INFO     | __main__:node_b:17 - cur_step: 2, node_b 被触发
2026-06-05 14:55:01.710 | INFO     | __main__:node_c:22 - cur_step: 2, node_c 被触发
2026-06-05 14:55:01.713 | INFO     | __main__:node_d:27 - cur_step: 3, node_d 被触发
2026-06-05 14:55:01.714 | INFO     | __main__:node_e:32 - cur_step: 4, node_e 被触发
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791968410254-0f3f4b62-9a9e-4372-8ed5-ba52c3e08567.png)

由运行结果可知：

+ `node_a` 在第 1 个超步执行；
+ `node_b` 和 `node_c` 在第 2 个超步并行执行；
+ `node_d` 在第 3 个超步执行；
+ `node_e` 在第 4 个超步执行。

虽然 `node_c` 在第 2 个超步已经执行完成，**<font style="color:#DF2A3F;">但是 </font>**`**<font style="color:#DF2A3F;">node_e</font>**`**<font style="color:#DF2A3F;"> 并不会立刻触发</font>**。 因为这里使用的是：

```python
builder.add_edge(["node_c", "node_d"], "node_e")
```

这表示**<font style="color:#DF2A3F;"> </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">node_e</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 需要等待 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">node_c</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 和 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">node_d</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 两个上游节点全部完成后，才会被触发一次。</font>**

**<font style="color:#DF2A3F;background-color:#FBDE28;">因此，这种写法对应的是 “与”触发：等待所有上游分支到达方可触发。</font>**







#### “或”触发：任意上游分支到达即可触发
另一种情况是：**<font style="color:#DF2A3F;">多个上游分支分别连接到同一个下游节点，但它们之间没有显式的同步等待关系。</font>**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_core.runnables import RunnableConfig

from loguru import logger

class EmptyState(TypedDict):
    pass

def node_a(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_a 被触发", cur_step)
    return {}

def node_b(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_b 被触发", cur_step)
    return {}

def node_c(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_c 被触发", cur_step)
    return {}

def node_d(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_d 被触发", cur_step)
    return {}

def node_e(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("cur_step: {}, node_e 被触发", cur_step)
    return {}

builder = StateGraph(state_schema=EmptyState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_node("node_d", node_d)
builder.add_node("node_e", node_e)

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_a", "node_c")
builder.add_edge("node_b", "node_d")

# 独立触发：node_c 完成后可以触发 node_e
builder.add_edge("node_c", "node_e")
# 独立触发：node_d 完成后可以触发 node_e
builder.add_edge("node_d", "node_e")

graph = builder.compile()
graph.invoke({})

from IPython.display import display
display(graph)
```

  
 **<font style="color:rgb(60, 60, 67);">运行结果如下</font>**

```python
2026-06-05 15:09:11.715 | INFO     | __main__:node_a:12 - cur_step: 1, node_a 被触发
2026-06-05 15:09:11.720 | INFO     | __main__:node_b:17 - cur_step: 2, node_b 被触发
2026-06-05 15:09:11.723 | INFO     | __main__:node_c:22 - cur_step: 2, node_c 被触发
2026-06-05 15:09:11.729 | INFO     | __main__:node_d:27 - cur_step: 3, node_d 被触发
2026-06-05 15:09:11.729 | INFO     | __main__:node_e:32 - cur_step: 3, node_e 被触发
2026-06-05 15:09:11.732 | INFO     | __main__:node_e:32 - cur_step: 4, node_e 被触发
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791968523850-dd188ca9-c008-4e17-a2c3-54fe7d24b2ac.png)

从运行结果可以看出，`node_e` 被触发了两次：

1. `node_c` 在第 2 个超步执行完成后，触发 `node_e` 在第 3 个超步执行；
2. `node_d` 在第 3 个超步执行完成后，再次触发 `node_e` 在第 4 个超步执行。

原因是这里使用了两条独立的边：

```python
builder.add_edge("node_c", "node_e")
builder.add_edge("node_d", "node_e")
```

这表示 `node_c` 和 `node_d` 都可以分别触发 `node_e`，它们之间没有同步等待关系。

因此，**<font style="color:#DF2A3F;">这种写法对应的是 独立触发，也可以类比为 “或”触发。</font>**

需要特别注意：

```python
builder.add_edge(["node_c", "node_d"], "node_e")
```

和

```python
builder.add_edge("node_c", "node_e")
builder.add_edge("node_d", "node_e")
```

不是等价写法。

前者表示 等待多个上游全部完成后才触发一次； 后者表示 多个上游分支分别独立触发下游节点。





### 动态扇入-MapReduce结构
在上文我们通过 `Send` 动态生成了多个下游任务。

这种模式本质上是一种动态扇出：根据运行时数据，将一个任务拆分为多个子任务，并分别发送给下游节点处理。

如果这些子任务分别产生中间结果，并在后续通过 `Reducer` 进行合并，也就是重新 扇入 到一个下游节点中，最终得到统一的结果，那么就构成了典型的 `MapReduce` 结构。





`MapReduce` 是大数据计算中的经典模型，通常包含两个核心阶段：

+ `**<font style="color:#DF2A3F;">Map</font>**`**<font style="color:#DF2A3F;">：映射阶段 将输入数据映射为中间结果。</font>**

在 `LangGraph` 中，可以理解为通过 `Send` 将子任务分发给多个 `mapper` 节点实例，每个节点实例独立完成局部计算：将部分输入数据映射为中间结果。

+ `**<font style="color:#DF2A3F;">Reduce</font>**`**<font style="color:#DF2A3F;">：归约阶段 将多个子任务产生的中间结果进行汇总、合并或聚合，得到最终结果</font>**。

在 `LangGraph` 中，通常由一个特定的 `reducer` 节点完成归约：它接收上游 `mapper` 节点实例产生的中间结果，处理后得到计算图的最终输出。

本节实现一个经典的词频统计任务

```python
输入数据
  |
  | hello world
  | hello Atguigu
  | hello LLM
  v

Map：映射为中间键值对
  |
  | (hello, 1), (world, 1)
  | (hello, 1), (Atguigu, 1)
  | (hello, 1), (LLM, 1)
  v

Shuffle / Group：按 Key 分组
  |
  | hello   -> [1, 1, 1]
  | world   -> [1]
  | Atguigu -> [1]
  | LLM     -> [1]
  v

Reduce：归约 / 聚合
  |
  | hello   -> 3
  | world   -> 1
  | Atguigu -> 1
  | LLM     -> 1
  v

最终结果
```

需要注意，本节示例中没有单独实现一个 `Shuffle` 节点，而是把 `Shuffle / Group` 的逻辑也放在了 `reducer_node` 中完成。

在下面的示例中，我们用 `Send` 动态派发任务，创建了多个 `mapper_node` 实例。

每个 `mapper_node` 实例负责处理一条输入数据，并输出中间产物：键值对。

随后，下游的 `reducer_node` 收集所有中间结果，完成 `Shuffle / Group` 和最终的 `Reduce` 聚合。



```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from _collections_abc import Sequence
from operator import add

from loguru import logger

class OverAllState(TypedDict):
    input_values: list[str]
    entries: Annotated[list[tuple[str, int]], add]
    word_counts: dict[str, int]

def router_map(state: OverAllState) -> Sequence[Send]:
    input_values = state["input_values"]

    tasks = []
    for input_value in input_values:
        tasks.append(
            Send(
                "mapper_node",
                {"input_value": input_value}
            )
        )

    return tasks

class MaperInputState(TypedDict):
    input_value: str

def mapper_node(state: MaperInputState) -> OverAllState:
    input_value = state["input_value"]
    words = input_value.split(" ")
    entries = []

    for word in words:
        entries.append((word, 1))

    return {
        "entries": entries
    }

def reducer_node(state: OverAllState) -> OverAllState:
    entries = state["entries"]
    logger.info("reducer entries: {}", entries)

    shuffle_dict = {}

    for k, v in entries:
        if k not in shuffle_dict:
            shuffle_dict[k] = [v]
        else:
            shuffle_dict[k].append(v)

    logger.info("reducer shuffle entries: {}", shuffle_dict)

    reduce_dict = {}

    for k, v in shuffle_dict.items():
        reduce_dict[k] = sum(v)

    return {
        "word_counts": reduce_dict
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("mapper_node", mapper_node)
builder.add_node("reducer_node", reducer_node)
builder.add_conditional_edges(START, router_map, path_map=["mapper_node"])
builder.add_edge("mapper_node", "reducer_node")
builder.add_edge("reducer_node", END)

graph = builder.compile()
word_counts = graph.invoke(
    {"input_values": [
        "hello world",
        "hello Atguigu",
        "hello LLM"
    ]}
)

print(word_counts)

from IPython.display import display
display(graph)
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791969020981-0515f6bb-9304-4ed7-b4a4-8ccedb26cf81.png)

```python
2026-06-05 13:52:01.917 | INFO     | __main__:reducer_node:45 - reducer entries: [('hello', 1), ('world', 1), ('hello', 1), ('Atguigu', 1), ('hello', 1), ('LLM', 1)]
2026-06-05 13:52:01.918 | INFO     | __main__:reducer_node:55 - reducer shuffle entries: {'hello': [1, 1, 1], 'world': [1], 'Atguigu': [1], 'LLM': [1]}
{
    "input_values": [
        "hello world",
        "hello Atguigu",
        "hello LLM",
    ],
    "entries": [
        ("hello", 1),
        ("world", 1),
        ("hello", 1),
        ("Atguigu", 1),
        ("hello", 1),
        ("LLM", 1),
    ],
    "word_counts": {
        "hello": 3,
        "world": 1,
        "Atguigu": 1,
        "LLM": 1,
    },
}
```

从结果可以看出，三个输入字符串分别被映射为了中间键值对：

```python
("hello", 1), ("world", 1)
("hello", 1), ("Atguigu", 1)
("hello", 1), ("LLM", 1)
```

  
 由于 `entries` 字段使用了 `Reducer`（`LangGraph` 状态的 `Reducer`，而不是当前示例中的 `reducer_node`）：

```python
entries: Annotated[list[tuple[str, int]], add]
```

所以多个 `mapper_node` 任务实例返回的 `entries` 会被合并到同一个状态字段中。

随后，`reducer_node` 读取合并后的 `entries`，先按单词进行分组：

```python
{
    "hello": [1, 1, 1],
    "world": [1],
    "Atguigu": [1],
    "LLM": [1],
}
```



再对每个分组中的数值进行求和，得到最终词频统计结果：

```python
{
    "hello": 3,
    "world": 1,
    "Atguigu": 1,
    "LLM": 1,
}
```



如果按照 任务实例 而不是 节点定义 来绘制运行图，那么这个示例实际的任务级运行图如下所示：

```python
flowchart TD
    START([__start__])

    START --> M1["mapper_node # task 1"]
    START --> M2["mapper_node # task 2"]
    START --> M3["mapper_node # task 3"]

    M1 --> R["reducer_node task"]
    M2 --> R
    M3 --> R

    R --> END([__end__])
```

需要注意，`display(graph)` 展示的是节点级计算图，而不是运行时的任务实例图。 因此在可视化图中只会看到一个 `mapper_node` 节点；但在实际运行时，`Send` 会根据输入数据动态创建多个 `mapper_node` 任务实例。



## 循环结构
### 循环结构的实现
本节通过两种方式实现经典的 `**<font style="color:#DF2A3F;">ReAct</font>**`**<font style="color:#DF2A3F;"> 循环结构。</font>**`**<font style="color:#DF2A3F;">LangChain Agent</font>**`**<font style="color:#DF2A3F;"> 底层运行图架构正是 </font>**`**<font style="color:#DF2A3F;">ReAct</font>**` 。

`ReAct` 是 `Reason + Action` 的缩写，即“推理 + 行动”架构。其核心思想是：

1. **<font style="color:#DF2A3F;">Reason：大模型根据当前消息状态进行推理，判断是否需要调用工具；</font>**
2. **<font style="color:#DF2A3F;">Action</font>****<font style="color:#DF2A3F;">：如果需要调用工具，则生成工具调用请求；</font>**
3. **<font style="color:#DF2A3F;">Observation</font>****<font style="color:#DF2A3F;">：工具执行后，将执行结果以 </font>**`**<font style="color:#DF2A3F;">ToolMessage</font>**`**<font style="color:#DF2A3F;"> 的形式返回给大模型；</font>**
4. **<font style="color:#DF2A3F;">Loop</font>****<font style="color:#DF2A3F;">：大模型基于新的观察结果继续推理，决定是否继续调用工具；</font>**
5. **<font style="color:#DF2A3F;">Final Answer：当大模型不再发起工具调用时，生成最终回答，流程结束。</font>**

因此，`ReAct` 本质上是一个典型的 **<font style="color:#DF2A3F;">“LLM → Tool → LLM → Tool → ... → LLM”</font>** 循环结构。

本节分别使用两种方式实现该循环：

+ 静态实现：**<font style="color:#DF2A3F;">通过 </font>**`**<font style="color:#DF2A3F;">add_conditional_edges()</font>**`**<font style="color:#DF2A3F;"> 在图结构中显式定义条件路由；</font>**
+ 动态实现：**<font style="color:#DF2A3F;">通过 </font>**`**<font style="color:#DF2A3F;">Command(goto=...)</font>**`**<font style="color:#DF2A3F;"> 在节点返回值中触发运行时跳转。</font>**







#### 静态实现
所谓“静态实现”，并不是指执行路径完全固定，而是指：

节点的下游候选集合在图编译阶段已经确定，运行时只是从这些候选节点中选择下一步。

在本例中，`llm_node` 的下游候选节点是固定的：

+ **<font style="color:#DF2A3F;">如果大模型返回了 </font>**`**<font style="color:#DF2A3F;">tool_calls</font>**`**<font style="color:#DF2A3F;">，则进入 </font>**`**<font style="color:#DF2A3F;">tool_node</font>**`**<font style="color:#DF2A3F;">；</font>**
+ **<font style="color:#DF2A3F;">如果大模型没有返回 </font>**`**<font style="color:#DF2A3F;">tool_calls</font>**`**<font style="color:#DF2A3F;">，则进入 </font>**`**<font style="color:#DF2A3F;">output_node</font>**`**<font style="color:#DF2A3F;">。</font>**

也就是说，图结构在编译阶段已经知道 `llm_node` 可能流向 `tool_node` 或 `output_node`，运行时只负责判断具体走哪一条路径。

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langchain.messages import HumanMessage, ToolMessage, SystemMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from random import randint
from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

@tool(parse_docstring=True)
def get_weather(city: str="北京"):
    """
    查询指定城市当日天气

    Args:
        city: 城市名称
    """
    return f"{city} 今天天气晴朗，东南风三级，气温 25~30 ℃"

@tool(parse_docstring=True)
def get_news(domain: Literal["AI", "食品安全"]):
    """
    查询特定领域的当日热点

    Args:
        domain: 特定领域
    """
    if domain == "AI":
        return "Anthropic 发布了 Claude Opus-4.8，但通过 API 用中文向它发送“你是谁？”时，大多数情况下返回的却是“Qwen”或“Deepseek”。"
    else:
        return "双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍"

tools = [get_weather, get_news]

model_with_tools = model.bind_tools(tools=tools)

class OverAllState(MessagesState):
    user_input: str
    final_answer: str

def input_node(state: OverAllState) ->OverAllState:
    return {
        "messages": [HumanMessage(state["user_input"])],
    }

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    ai_msg = model_with_tools.invoke(messages)
    return {
        "messages": [ai_msg],
    }

def tool_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    ai_msg = messages[-1]
    tool_calls = ai_msg.tool_calls

    fail_prob = 6 # 失败概率，6表示 60% 概率失败，7 -> 70%，8 -> 80%，依次类推

    for tool_call in tool_calls:
        if tool_call["name"] == "get_weather":
            # 生成 [0, 9] 范围内的随机数
            if randint(0, 9) < fail_prob: # 60% 概率因 “网络波动” 而调用失败
                messages.append(
                    ToolMessage(
                        content="网络波动，调用失败，请重试",
                        tool_call_id=tool_call["id"]
                    )
                )
            else:
                messages.append(get_weather.invoke(tool_call))
        elif tool_call["name"] == "get_news":
            # 生成 [0, 9] 范围内的随机数
            if randint(0, 9) < fail_prob: # 60% 概率因 “网络波动” 而调用失败
                messages.append(
                    ToolMessage(
                        content="网络波动，调用失败，请重试",
                        tool_call_id=tool_call["id"]
                    )
                )
            else:
                messages.append(get_news.invoke(tool_call))
        else:
            messages.append(
                ToolMessage(
                    content="工具名称错误，调用失败，请重试",
                    tool_call_id=tool_call["id"]
                )
            )

    return {
        "messages": messages
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "final_answer": state["messages"][-1].content
    }

def router(state: OverAllState) -> Literal["tool_node", "output_node"]:
    messages = state["messages"]
    lst_msg = messages[-1]

    if lst_msg.tool_calls:
        return "tool_node"
    return "output_node"

builder = StateGraph(state_schema=OverAllState)
builder.add_node("input_node", input_node)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", tool_node)
builder.add_node("output_node", output_node)

builder.add_edge(START, "input_node")
builder.add_edge("input_node", "llm_node")
builder.add_conditional_edges(
    "llm_node",
    router
)
builder.add_edge("tool_node", "llm_node")
builder.add_edge("output_node", END)

graph = builder.compile()
ai_res = graph.invoke({
    "user_input": "帮我查询杭州当日天气和AI热点",
    "messages": [SystemMessage("如果工具调用失败，必须重新调用直至成功")]
})
food_safety_res = graph.invoke({
    "user_input": "帮我查询当日天气和食品安全相关的热点",
    "messages": [SystemMessage("如果工具调用失败，必须重新调用直至成功")]
})

print('=' * 30, '-> ai_res <-', '=' * 30)
print("user_input: ", ai_res["user_input"])
print("final_answer: ", ai_res["final_answer"])
for msg in ai_res["messages"]:
    msg.pretty_print()

print('=' * 30, '-> food_safety_res <-', '=' * 30)
print("user_input: ", food_safety_res["user_input"])
print("final_answer: ", food_safety_res["final_answer"])
for msg in food_safety_res["messages"]:
    msg.pretty_print()

from IPython.display import display
display(graph)
```



```python
============================== -> ai_res <- ==============================
user_input:  帮我查询杭州当日天气和AI热点
final_answer:  查询成功！以下是结果汇总：

---

### 🌤️ 杭州当日天气
- **天气状况**：晴朗
- **气温范围**：25°C ~ 30°C
- **风力**：东南风三级

### 🤖 AI热点
**Anthropic 发布了 Claude Opus-4.8**，但通过 API 用中文向它发送"你是谁？"时，大多数情况下返回的却是"Qwen"或"Deepseek"。（这条AI新闻可能反映了当前大模型领域的一些有趣现象或争议）

---

杭州今天天气不错，晴朗舒适，适合外出活动！而AI领域似乎又有了新的模型发布动态。您还想了解其他信息吗？
================================[1m System Message [0m================================

如果工具调用失败，必须重新调用直至成功
================================[1m Human Message [0m=================================

帮我查询杭州当日天气和AI热点
==================================[1m Ai Message [0m==================================

好的，我来同时查询杭州的天气和AI热点。
Tool Calls:
  get_weather (call_00_DPkgkfOYSBOuic3ipHj11702)
 Call ID: call_00_DPkgkfOYSBOuic3ipHj11702
  Args:
    city: 杭州
  get_news (call_01_clcon3yt50HhIKGKBaxl4926)
 Call ID: call_01_clcon3yt50HhIKGKBaxl4926
  Args:
    domain: AI
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
==================================[1m Ai Message [0m==================================

两次调用都因网络波动失败了，让我重新尝试查询。
Tool Calls:
  get_weather (call_00_ruPVWhBySBDG9B9AR7vN4587)
 Call ID: call_00_ruPVWhBySBDG9B9AR7vN4587
  Args:
    city: 杭州
  get_news (call_01_ENBOMtrHXaJn7u1o7PPS6090)
 Call ID: call_01_ENBOMtrHXaJn7u1o7PPS6090
  Args:
    domain: AI
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
=================================[1m Tool Message [0m=================================
Name: get_news

Anthropic 发布了 Claude Opus-4.8，但通过 API 用中文向它发送“你是谁？”时，大多数情况下返回的却是“Qwen”或“Deepseek”。
==================================[1m Ai Message [0m==================================

天气查询仍然失败，我再次尝试查询天气。
Tool Calls:
  get_weather (call_00_BSSyk3lOTQxZEzXLmbJr7873)
 Call ID: call_00_BSSyk3lOTQxZEzXLmbJr7873
  Args:
    city: 杭州
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
==================================[1m Ai Message [0m==================================
Tool Calls:
  get_weather (call_00_WQZzpmwpfIQfGjkO98752270)
 Call ID: call_00_WQZzpmwpfIQfGjkO98752270
  Args:
    city: 杭州
=================================[1m Tool Message [0m=================================
Name: get_weather

杭州 今天天气晴朗，东南风三级，气温 25~30 ℃
==================================[1m Ai Message [0m==================================

查询成功！以下是结果汇总：

---

### 🌤️ 杭州当日天气
- **天气状况**：晴朗
- **气温范围**：25°C ~ 30°C
- **风力**：东南风三级

### 🤖 AI热点
**Anthropic 发布了 Claude Opus-4.8**，但通过 API 用中文向它发送"你是谁？"时，大多数情况下返回的却是"Qwen"或"Deepseek"。（这条AI新闻可能反映了当前大模型领域的一些有趣现象或争议）

---

杭州今天天气不错，晴朗舒适，适合外出活动！而AI领域似乎又有了新的模型发布动态。您还想了解其他信息吗？
============================== -> food_safety_res <- ==============================
user_input:  帮我查询当日天气和食品安全相关的热点
final_answer:  查询成功！以下是您需要的信息：

---

### 🌤️ 当日天气（北京）
- **天气状况：** 晴朗
- **气温：** 25~30℃
- **风力：** 东南风三级

### 🔍 食品安全热点
**双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍**
> 这是一则关于知名食品企业双汇旗下子公司猪肉产品被检出抗生素残留严重超标的新闻，超标幅度高达37.5倍，属于较为严重的食品安全问题。

---

如果您想查询其他城市的天气，也可以告诉我哦！
================================[1m System Message [0m================================

如果工具调用失败，必须重新调用直至成功
================================[1m Human Message [0m=================================

帮我查询当日天气和食品安全相关的热点
==================================[1m Ai Message [0m==================================

好的，我来同时查询当日天气和食品安全相关的热点。
Tool Calls:
  get_weather (call_00_NkvnFHT4M4h9wwENZmyu9753)
 Call ID: call_00_NkvnFHT4M4h9wwENZmyu9753
  Args:
    city: 北京
  get_news (call_01_rbFBctitbMLZQivR1TYd2949)
 Call ID: call_01_rbFBctitbMLZQivR1TYd2949
  Args:
    domain: 食品安全
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
=================================[1m Tool Message [0m=================================
Name: get_news

双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍
==================================[1m Ai Message [0m==================================

天气查询因为网络波动失败了，我重新查询一下天气信息。
Tool Calls:
  get_weather (call_00_fWmSdUOseojF7PADvgOA7804)
 Call ID: call_00_fWmSdUOseojF7PADvgOA7804
  Args:
    city: 北京
=================================[1m Tool Message [0m=================================
Name: get_weather

北京 今天天气晴朗，东南风三级，气温 25~30 ℃
==================================[1m Ai Message [0m==================================

查询成功！以下是您需要的信息：

---

### 🌤️ 当日天气（北京）
- **天气状况：** 晴朗
- **气温：** 25~30℃
- **风力：** 东南风三级

### 🔍 食品安全热点
**双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍**
> 这是一则关于知名食品企业双汇旗下子公司猪肉产品被检出抗生素残留严重超标的新闻，超标幅度高达37.5倍，属于较为严重的食品安全问题。

---

如果您想查询其他城市的天气，也可以告诉我哦！
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791971299834-e48f2775-51c5-499f-946e-293d151e3be8.png)

本例运行图中的循环结构如下：

```python
START
  ↓
input_node
  ↓
llm_node
  ├── 有 tool_calls → tool_node → llm_node
  └── 无 tool_calls → output_node → END
```

其中，循环发生在：

```python
llm_node → tool_node → llm_node
```

只要大模型持续返回工具调用，流程就会不断回到 `llm_node`，形成 `ReAct` 循环。

需要注意的是，本例中“**<font style="color:#DF2A3F;">工具调用失败后继续重试”主要依赖这条系统提示词：</font>**

```python
SystemMessage("如果工具调用失败，必须重新调用直至成功")
```

也就是说，是否继续重试，在当前实现中主要由大模型决定，而不是由程序逻辑强制保证。

因此，如果模型在多次工具调用失败后选择停止重试并生成最终回答，运行图不会阻止它。







#### 动态实现
除了静态路由，我们还可以利用 `Command` 在运行时动态指定下一个要执行的节点。

在本节的实现中，`**<font style="color:#DF2A3F;">llm_node</font>**`**<font style="color:#DF2A3F;"> 借助 </font>**`**<font style="color:#DF2A3F;">Command</font>**`**<font style="color:#DF2A3F;"> 在返回值中同时完成两件事：</font>**

+ **<font style="color:#DF2A3F;">更新状态；</font>**
+ **<font style="color:#DF2A3F;">控制下一步的跳转。</font>**

这里的 动态 是指：

**<font style="color:#DF2A3F;">节点在运行时，根据自身的执行结果，直接返回“状态更新 + 下一跳控制指令”。</font>**

在本例中，`llm_node` 调用大模型后，会根据返回结果决定后续流程：

+ 如果 `ai_msg.tool_calls` 不为空，则 `goto="tool_node"`；
+ 如果 `ai_msg.tool_calls` 为空，则 `goto="output_node"`。

这样一来，**<font style="color:#DF2A3F;">路由逻辑就被内聚在 </font>**`**<font style="color:#DF2A3F;">llm_node</font>**`**<font style="color:#DF2A3F;"> 节点，</font>**不再需要额外定义 `router()` 函数，也不再需要调用 `add_conditional_edges()` 从 `llm_node` 添加条件边。





```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.types import Command
from langchain.messages import HumanMessage, ToolMessage, SystemMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from random import randint
from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

@tool(parse_docstring=True)
def get_weather(city: str="北京"):
    """
    查询指定城市当日天气

    Args:
        city: 城市名称
    """
    return f"{city} 今天天气晴朗，东南风三级，气温 25~30 ℃"

@tool(parse_docstring=True)
def get_news(domain: Literal["AI", "食品安全"]):
    """
    查询特定领域的当日热点

    Args:
        domain: 特定领域
    """
    if domain == "AI":
        return "Anthropic 发布了 Claude Opus-4.8，但通过 API 用中文向它发送“你是谁？”时，大多数情况下返回的却是“Qwen”或“Deepseek”。"
    else:
        return "双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍"

tools = [get_weather, get_news]

model_with_tools = model.bind_tools(tools=tools)

class OverAllState(MessagesState):
    user_input: str
    final_answer: str

def input_node(state: OverAllState) ->OverAllState:
    return {
        "messages": [HumanMessage(state["user_input"])],
    }

def llm_node(state: OverAllState) -> Command[Literal["tool_node", "output_node"]]:
    messages = state["messages"]
    ai_msg = model_with_tools.invoke(messages)
    if ai_msg.tool_calls:
        goto = "tool_node"
    else:
        goto = "output_node"

    return Command(
        goto=goto,
        update={
            "messages": [ai_msg],
        }
    )

def tool_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    ai_msg = messages[-1]
    tool_calls = ai_msg.tool_calls

    fail_prob = 6 # 失败概率，6表示 60% 概率失败，7 -> 70%，8 -> 80%，依次类推

    for tool_call in tool_calls:
        if tool_call["name"] == "get_weather":
            # 生成 [0, 9] 范围内的随机数
            if randint(0, 9) < fail_prob: # 60% 概率因 “网络波动” 而调用失败
                messages.append(
                    ToolMessage(
                        content="网络波动，调用失败，请重试",
                        tool_call_id=tool_call["id"]
                    )
                )
            else:
                messages.append(get_weather.invoke(tool_call))
        elif tool_call["name"] == "get_news":
            # 生成 [0, 9] 范围内的随机数
            if randint(0, 9) < fail_prob: # 60% 概率因 “网络波动” 而调用失败
                messages.append(
                    ToolMessage(
                        content="网络波动，调用失败，请重试",
                        tool_call_id=tool_call["id"]
                    )
                )
            else:
                messages.append(get_news.invoke(tool_call))
        else:
            messages.append(
                ToolMessage(
                    content="工具名称错误，调用失败，请重试",
                    tool_call_id=tool_call["id"]
                )
            )

    return {
        "messages": messages
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "final_answer": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("input_node", input_node)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", tool_node)
builder.add_node("output_node", output_node)

builder.add_edge(START, "input_node")
builder.add_edge("input_node", "llm_node")
builder.add_edge("tool_node", "llm_node")
builder.add_edge("output_node", END)

graph = builder.compile()
ai_res = graph.invoke({
    "user_input": "帮我查询杭州当日天气和AI热点",
    "messages": [SystemMessage("如果工具调用失败，必须重新调用直至成功")]
})
food_safety_res = graph.invoke({
    "user_input": "帮我查询当日天气和食品安全相关的热点",
    "messages": [SystemMessage("如果工具调用失败，必须重新调用直至成功")]
})

print('=' * 30, '-> ai_res <-', '=' * 30)
print("user_input: ", ai_res["user_input"])
print("final_answer: ", ai_res["final_answer"])
for msg in ai_res["messages"]:
    msg.pretty_print()

print('=' * 30, '-> food_safety_res <-', '=' * 30)
print("user_input: ", food_safety_res["user_input"])
print("final_answer: ", food_safety_res["final_answer"])
for msg in food_safety_res["messages"]:
    msg.pretty_print()

from IPython.display import display
display(graph)
```

运行结果如下所示

```python
============================== -> ai_res <- ==============================
user_input:  帮我查询杭州当日天气和AI热点
final_answer:  以下是查询结果：

---

### 🌤️ 杭州当日天气
- **天气状况**：晴朗
- **气温**：25~30 ℃
- **风力**：东南风三级

### 🤖 AI 热点
- **Anthropic 发布了 Claude Opus-4.8**，但有趣的是，通过 API 用中文向它发送"你是谁？"时，大多数情况下返回的却是"Qwen"或"Deepseek"。

---

看起来杭州今天天气不错，适合外出活动！而AI领域这则关于Claude的有趣现象也引发了不少讨论。还有其他想了解的吗？
================================[1m System Message [0m================================

如果工具调用失败，必须重新调用直至成功
================================[1m Human Message [0m=================================

帮我查询杭州当日天气和AI热点
==================================[1m Ai Message [0m==================================

好的，我来同时查询杭州的天气和AI热点。
Tool Calls:
  get_weather (call_00_QAXNeCIuy4MEe91ZiBQp0274)
 Call ID: call_00_QAXNeCIuy4MEe91ZiBQp0274
  Args:
    city: 杭州
  get_news (call_01_5hXZth3YLTZFZZKE48i62455)
 Call ID: call_01_5hXZth3YLTZFZZKE48i62455
  Args:
    domain: AI
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
=================================[1m Tool Message [0m=================================
Name: get_news

Anthropic 发布了 Claude Opus-4.8，但通过 API 用中文向它发送“你是谁？”时，大多数情况下返回的却是“Qwen”或“Deepseek”。
==================================[1m Ai Message [0m==================================

天气查询遇到了网络波动，我重新尝试一下。
Tool Calls:
  get_weather (call_00_ooZcTsB9ntQRU3tXDrVj7186)
 Call ID: call_00_ooZcTsB9ntQRU3tXDrVj7186
  Args:
    city: 杭州
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
==================================[1m Ai Message [0m==================================

再次重试：
Tool Calls:
  get_weather (call_00_EwVrJmFUZaYZfvkOxveK7822)
 Call ID: call_00_EwVrJmFUZaYZfvkOxveK7822
  Args:
    city: 杭州
=================================[1m Tool Message [0m=================================
Name: get_weather

杭州 今天天气晴朗，东南风三级，气温 25~30 ℃
==================================[1m Ai Message [0m==================================

以下是查询结果：

---

### 🌤️ 杭州当日天气
- **天气状况**：晴朗
- **气温**：25~30 ℃
- **风力**：东南风三级

### 🤖 AI 热点
- **Anthropic 发布了 Claude Opus-4.8**，但有趣的是，通过 API 用中文向它发送"你是谁？"时，大多数情况下返回的却是"Qwen"或"Deepseek"。

---

看起来杭州今天天气不错，适合外出活动！而AI领域这则关于Claude的有趣现象也引发了不少讨论。还有其他想了解的吗？
============================== -> food_safety_res <- ==============================
user_input:  帮我查询当日天气和食品安全相关的热点
final_answer:  非常抱歉，天气查询接口持续出现网络波动，暂时无法获取到天气信息。以下是目前已获取到的信息：

---

### 📰 食品安全热点
**双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍**
> 这是今日食品安全领域的一条重要新闻，涉及知名肉制品企业双汇发展，其子公司检测出猪肉产品中抗生素残留超标情况较为严重。

---

### 🌤️ 天气查询
天气接口当前多次重试均因网络波动失败，暂时无法获取当日天气信息，建议稍后再尝试查询。
================================[1m System Message [0m================================

如果工具调用失败，必须重新调用直至成功
================================[1m Human Message [0m=================================

帮我查询当日天气和食品安全相关的热点
==================================[1m Ai Message [0m==================================

我来为你查询当日天气和食品安全相关的热点。
Tool Calls:
  get_weather (call_00_JYYAWJk569VD0TZ4LCyr0840)
 Call ID: call_00_JYYAWJk569VD0TZ4LCyr0840
  Args:
    city: 北京
  get_news (call_01_YkzloPKV0DVwL1r5nBgi8164)
 Call ID: call_01_YkzloPKV0DVwL1r5nBgi8164
  Args:
    domain: 食品安全
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
=================================[1m Tool Message [0m=================================
Name: get_news

双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍
==================================[1m Ai Message [0m==================================

天气查询遇到网络波动，我来重试一下。
Tool Calls:
  get_weather (call_00_MjNdBAQz2gW6EzZavsZ91123)
 Call ID: call_00_MjNdBAQz2gW6EzZavsZ91123
  Args:
    city: 北京
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
==================================[1m Ai Message [0m==================================
Tool Calls:
  get_weather (call_00_pkcSelcAJiFDNpU7II0d9212)
 Call ID: call_00_pkcSelcAJiFDNpU7II0d9212
  Args:
    city: 北京
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
==================================[1m Ai Message [0m==================================
Tool Calls:
  get_weather (call_00_QKgt5xYTos0H1c6CIS9a6172)
 Call ID: call_00_QKgt5xYTos0H1c6CIS9a6172
  Args:
    city: 北京
=================================[1m Tool Message [0m=================================

网络波动，调用失败，请重试
==================================[1m Ai Message [0m==================================

非常抱歉，天气查询接口持续出现网络波动，暂时无法获取到天气信息。以下是目前已获取到的信息：

---

### 📰 食品安全热点
**双汇发展子公司猪肉产品被抽检出抗生素超标37.5倍**
> 这是今日食品安全领域的一条重要新闻，涉及知名肉制品企业双汇发展，其子公司检测出猪肉产品中抗生素残留超标情况较为严重。

---

### 🌤️ 天气查询
天气接口当前多次重试均因网络波动失败，暂时无法获取当日天气信息，建议稍后再尝试查询。
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791971453569-5178fd09-29c5-4989-8e78-4572442e3d8d.png)



本例和静态实现的核心区别体现在 `llm_node` 中：

```python
def llm_node(state: OverAllState) -> Command[Literal["tool_node", "output_node"]]:
    ai_msg = model_with_tools.invoke(state["messages"])

    if ai_msg.tool_calls:
        goto = "tool_node"
    else:
        goto = "output_node"

    return Command(
        update={
            "messages": [ai_msg],
        },
        goto=goto,
    )
```

其返回值类型：

```python
Command[Literal["tool_node", "output_node"]]
```

表明该节点返回的是一个 `Command` 对象，且 `goto` 字段的值只能是：

+ `"tool_node"`
+ 或 `"output_node"`

这是静态约束，不会参与运行时校验。主要作用是提升代码可读性、通过静态类型检查，同时帮助 `LangGraph` 更准确地推断图结构。  
   
 



#### 小结
|  | 静态实现 | 动态实现 |
| --- | --- | --- |
| 路由位置 | **<font style="color:#DF2A3F;">路由逻辑在独立的 </font>**`**<font style="color:#DF2A3F;">router()</font>**`<br/>**<font style="color:#DF2A3F;"> 函数中</font>** | **<font style="color:#DF2A3F;">路由逻辑写在节点返回值中</font>** |
| 图结构表达 | 显式声明条件边，更直观 | 控制逻辑更内聚，代码更紧凑 |
| 适合场景 | 路由规则独立、希望图结构更清晰 | 节点执行结果直接决定下一跳 |


两种方式都能实现 `ReAct` 循环，本质区别不在于是否能循环，而在于：

路由逻辑写在哪里。

+ 基于 `add_conditional_edges()` 的静态实现：把控制逻辑放在图结构定义阶段；
+ 基于 `Command(goto=...)` 的动态实现：把控制逻辑放在节点返回值中。

  
 









### 引入递归限制
在存在循环结构的图中，如果没有合理的停止条件，**<font style="color:#DF2A3F;">运行图可能会一直循环执行。为了避免无限循环，</font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 提供了递归限制机制</font>**，用于限制单次图运行过程中允许执行的**<font style="color:#DF2A3F;">最大 </font>**`**<font style="color:#DF2A3F;">SuperStep</font>**`**<font style="color:#DF2A3F;"> 数量。</font>**

当运行图在达到停止条件之前耗尽允许的最大步数时，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 会抛出 </font>**`**<font style="color:#DF2A3F;">GraphRecursionError</font>**`。开发者既可以在图内部提前检测剩余步数并优雅退出，也可以在图外部捕获异常并集中处理。

<font style="color:rgb(60, 60, 67);"></font>

#### 步骤计数器
##### 运行时配置对象`config`
`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 运行节点时，会向节点函数注入运行时配置对象 </font>**`**<font style="color:#DF2A3F;">config</font>**`**<font style="color:#DF2A3F;">。该对象通常使用 </font>**`**<font style="color:#DF2A3F;">RunnableConfig</font>**`**<font style="color:#DF2A3F;"> 类型标注</font>**，用于记录本次运行的配置信息和部分运行时元数据。

节点函数的第一个参数必须是运行时状态。如果需要访问运行时配置，可以在状态参数之后声明额外参数 `config`。**<font style="color:#DF2A3F;">此时，</font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 会在调用节点函数之前，将运行时配置对象以关键字参数的方式传入节点函数。</font>**

`config` 的元数据中记录了当前节点所在的 `SuperStep` 序号，可以通过下面的方式获取：

```python
config["metadata"]["langgraph_step"]
```

需要注意的是，这里的步骤编号对应图运行过程中的 `SuperStep`，从 `1` 开始计数。对于顺序执行的节点，不同节点通常位于不同的 `SuperStep`；对于并行执行的节点，多个节点可能处于同一个 `SuperStep`，因此它们读取到的 `langgraph_step` 可能相同。

<font style="color:rgb(60, 60, 67);"></font>

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_core.runnables import RunnableConfig

from loguru import logger

class EmptyState(TypedDict):
    pass

def node_a(state: EmptyState, config: RunnableConfig) -> EmptyState | None:
    current_step = config["metadata"]["langgraph_step"]
    logger.info(current_step)

def node_b(state: EmptyState, config: RunnableConfig) -> EmptyState | None:
    current_step = config["metadata"]["langgraph_step"]
    logger.info(current_step)

def node_c(state: EmptyState, config: RunnableConfig) -> EmptyState | None:
    current_step = config["metadata"]["langgraph_step"]
    logger.info(current_step)

builder = StateGraph(state_schema=EmptyState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_c")
builder.add_edge("node_c", END)

graph = builder.compile()
graph.invoke({})

from IPython.display import display
display(graph)
```

```python
2026-05-29 19:40:07.569 | INFO     | __main__:node_a:12 - 1
2026-05-29 19:40:07.571 | INFO     | __main__:node_b:16 - 2
2026-05-29 19:40:07.573 | INFO     | __main__:node_c:20 - 3
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1791971808868-e6418702-c053-413b-876c-24457ff9bf2b.png)

可以看到，`node_a`、`node_b`、`node_c` 是顺序执行的三个节点，因此分别处于第 `1`、`2`、`3` 个 `SuperStep`。



#### 配置递归限制
`**<font style="color:#DF2A3F;">recursion_limit</font>**`**<font style="color:#DF2A3F;"> 表示单次图运行过程中允许执行的最大 </font>**`**<font style="color:#DF2A3F;">SuperStep</font>**`**<font style="color:#DF2A3F;"> 数量</font>**。可以在调用运行图时通过 `config` 显式配置：

```python
graph.invoke(input, config={"recursion_limit": 10})
```

需要注意的是，`recursion_limit` 的默认值可能随版本变化、运行环境切换而变化。



##### 版本的变化
官方文档提到：从 `LangGraph 1.0.6` 开始，`recursion_limit` 默认为 `1000`，本项目运行在 `LangGraph 1.1.2` 版本，实测发现 `recursion_limit` 默认为 `10000`。



##### 环境的变化
阅读源码可知，`LangGraph` 运行时可能从多个来源读取默认值

如下：

| 来源 | 默认值 | 说明 |
| --- | --- | --- |
| `langchain_core.runnables.config` | `25` | `RunnableConfig`<br/> 中定义的默认值，`LangChain Agent`<br/> 运行时生效 |
| `langgraph._internal._config` | `10000` | `LangGraph`<br/> 本地图运行时内部配置中的默认值，自定义 `LangGraph`<br/> 图结构时生效 |
| `langgraph_api.utils.config` | `10011` | `LangGraph API / 服务侧`<br/>相关配置中的默认值 |


对于自定义图结构的 `LangGraph` 运行时，其默认值取自 `langgraph._internal._config`，所以调试看到的默认值为 `10000`。



##### 最佳实践
在实际开发中，建议显式传入 `recursion_limit`，避免不同版本、不同运行入口或不同部署方式下默认值不同导致实际运行结果和预期不符。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(103, 103, 108);"></font>

#### 优雅退出：主动方法（Proactive Approach）
`RemainingSteps` 是 `LangGraph` 提供的特殊托管值，表示剩余可用步数，由运行时维护。

`**<font style="color:#DF2A3F;background-color:#FBDE28;">LangGraph</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 运行时会根据当前步数和 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">recursion_limit</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 计算剩余步数，并填充到 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">RemainingSteps</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 类型的状态字段中</font>**。开发者可以在状态中**<font style="color:#DF2A3F;">声明一个 </font>**`**<font style="color:#DF2A3F;">RemainingSteps</font>**`**<font style="color:#DF2A3F;"> 类型的字段</font>**，来获取剩余可用步数。

借助 `RemainingSteps`，开发者可以在图内部提前判断剩余步数是否充足，并根据剩余步数动态路由。例如，**<font style="color:#DF2A3F;">当剩余步数较少时，不再继续循环，而是路由到 </font>**`**<font style="color:#DF2A3F;">END</font>**`**<font style="color:#DF2A3F;"> 或兜底节点，</font>**从而让运行图正常结束。

这种方式是在达到递归限制之前主动处理，因此称为 主动方法（Proactive Approach）。

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.managed import RemainingSteps
from langchain_core.runnables.config import RunnableConfig

from loguru import logger

class OverAllState(TypedDict):
    remaining_steps: RemainingSteps

def loop_node(state: OverAllState, config: RunnableConfig) -> OverAllState:
    cur_step = config["metadata"]["langgraph_step"]
    remaining_steps = state["remaining_steps"]
    logger.info("loop_node, cur_step: {}, remaining_step: {}", cur_step, remaining_steps)

def router(state: OverAllState) -> Literal["loop_node", END]:
    remaining_steps = state["remaining_steps"]
    if remaining_steps < 3:
        logger.info("当前可用超步：{}，已不足2步，终止运行图", remaining_steps)
        return END
    return "loop_node"

builder = StateGraph(state_schema=OverAllState)
builder.add_node("loop_node", loop_node)
builder.add_edge(START, "loop_node")
builder.add_conditional_edges("loop_node", router)

graph = builder.compile()
graph.invoke({}, config={"recursion_limit": 10})

from IPython.display import display
display(graph)
```

<font style="color:rgb(103, 103, 108);"></font>

```python
2026-06-01 10:23:40.412 | INFO | __main__:loop_node:14 - loop_node, cur_step: 1, remaining_step: 9
2026-06-01 10:23:40.413 | INFO | __main__:loop_node:14 - loop_node, cur_step: 2, remaining_step: 8
2026-06-01 10:23:40.414 | INFO | __main__:loop_node:14 - loop_node, cur_step: 3, remaining_step: 7
2026-06-01 10:23:40.414 | INFO | __main__:loop_node:14 - loop_node, cur_step: 4, remaining_step: 6
2026-06-01 10:23:40.415 | INFO | __main__:loop_node:14 - loop_node, cur_step: 5, remaining_step: 5
2026-06-01 10:23:40.416 | INFO | __main__:loop_node:14 - loop_node, cur_step: 6, remaining_step: 4
2026-06-01 10:23:40.417 | INFO | __main__:loop_node:14 - loop_node, cur_step: 7, remaining_step: 3
2026-06-01 10:23:40.417 | INFO | __main__:loop_node:14 - loop_node, cur_step: 8, remaining_step: 2
2026-06-01 10:23:40.418 | INFO | __main__:router:19    - 当前可用超步：2，已不足2步，终止运行图
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792055534808-8250c3cf-bb24-49c9-b8ba-98b476d5d9dd.png)

本例中，`**<font style="color:#DF2A3F;">recursion_limit</font>**`**<font style="color:#DF2A3F;"> 被显式设置为 </font>**`**<font style="color:#DF2A3F;">10</font>**`**<font style="color:#DF2A3F;">。</font>**

第 `1` 个 `SuperStep` 执行时，剩余步数为 `9`；第 `2` 个 `SuperStep` 执行时，剩余步数为 `8`；依次类推。。。当前超步序号和剩余步数之和等于 `recursion_limit`。

第 `8` 个 SuperStep 执行时，剩余步数为 `2`。此时路由函数判断剩余步数较少，不再继续循环，而是路由到 `END`，运行图正常结束。

这种方式在运行图内部主动处理，不会抛出异常，因此是优雅退出的主动方法。



#### 异常中断：被动方法（Reactive Approach）
如果图中存在循环结构，并且运行图在达到停止条件之前耗尽了允许的最大步数，`LangGraph` 会抛出 `GraphRecursionError`。

开发者可以在图外部捕获该异常，处理递归限制超限的情况。由于这种方式是在异常已经发生之后再处理，因此称为 被动方法（`Reactive Approach`）。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.errors import GraphRecursionError
from langchain_core.runnables import RunnableConfig

from loguru import logger

class EmptyState(TypedDict):
    pass

def loop_node(state: EmptyState, config: RunnableConfig) -> EmptyState:
    cur_step = config["metadata"]["langgraph_step"]
    logger.info("loop_node, cur_step: {}", cur_step)

builder = StateGraph(state_schema=EmptyState)
builder.add_node("loop_node", loop_node)
builder.add_edge(START, "loop_node")
builder.add_edge("loop_node", "loop_node")

graph = builder.compile()
try:
    graph.invoke({}, config={"recursion_limit": 10})
except GraphRecursionError as e:
    logger.info("超步数量达到最大限制，抛出异常: {}", e)

from IPython.display import display
display(graph)
```

```python
2026-06-01 10:13:06.116 | INFO | __main__:loop_node:13 - loop_node, cur_step: 1
2026-06-01 10:13:06.117 | INFO | __main__:loop_node:13 - loop_node, cur_step: 2
2026-06-01 10:13:06.118 | INFO | __main__:loop_node:13 - loop_node, cur_step: 3
2026-06-01 10:13:06.119 | INFO | __main__:loop_node:13 - loop_node, cur_step: 4
2026-06-01 10:13:06.119 | INFO | __main__:loop_node:13 - loop_node, cur_step: 5
2026-06-01 10:13:06.120 | INFO | __main__:loop_node:13 - loop_node, cur_step: 6
2026-06-01 10:13:06.120 | INFO | __main__:loop_node:13 - loop_node, cur_step: 7
2026-06-01 10:13:06.121 | INFO | __main__:loop_node:13 - loop_node, cur_step: 8
2026-06-01 10:13:06.121 | INFO | __main__:loop_node:13 - loop_node, cur_step: 9
2026-06-01 10:13:06.122 | INFO | __main__:loop_node:13 - loop_node, cur_step: 10
2026-06-01 10:13:06.122 | INFO | __main__:<module>:24  - 超步数量达到最大限制，抛出异常: Recursion limit of 10 reached without hitting a stop condition. You can increase the limit by setting the `recursion_limit` config key.
                                                    For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792055724299-a45bb0e8-457f-4562-9c75-79579e771a78.png)

本例中，`loop_node` 的下游仍然是 `loop_node` 本身，因此运行图会不断循环。由于没有任何停止条件，**<font style="color:#DF2A3F;">当执行步数达到 </font>**`**<font style="color:#DF2A3F;">recursion_limit=10</font>**`**<font style="color:#DF2A3F;"> 后，LangGraph 抛出 </font>**`**<font style="color:#DF2A3F;">GraphRecursionError</font>**`**<font style="color:#DF2A3F;">。</font>**

这种方式虽然可以避免异常导致整个进程崩溃，但图本身已经被异常中断，不能像主动方法那样正常结束。





#### 主动方法和被动方法的对比
主动方法和被动方法的最大区别在于：

主动方法是在图内部提前处理递归限制；被动方法是在图外部捕获`递归限制异常`。

| 方法 | 超限检测 | 处理方法 | 控制流 |
| --- | --- | --- | --- |
| 主动(借助 `RemainingSteps`<br/>) | 达到限制之前 | 在图结构内部通过条件路由处理 | 图继续运行，正常结束 |
| 被动(捕获 `GraphRecursionError`<br/>) | 超出限制之后 | 图结构外部，通过 `try/except`<br/> 处理 | 图运行被异常中断 |


主动方法的优势：

+ 在图内部实现优雅降级
+ 可以将中间状态保存到检查点中
+ 通过返回部分结果改善用户体验
+ 图可以正常完成，不会抛出异常

被动方法的优势：

+ 实现更简单
+ 不需要修改图的内部逻辑
+ 可以集中处理错误

实际开发中，如果循环结构本身是业务逻辑的一部分，例如 ReAct 循环、多轮反思、自我修正、工具调用重试 等，更推荐使用主动方法，在图内部提前处理递归限制。

如果只是为了兜底防止无限循环，也可以保留被动方法，在最外层捕获 `GraphRecursionError`，作为最后一道防线。





## Edges总结：LangGraph中的边
上文我们已经学习了 `LangGraph` 中各种边的用法。本节对 `LangGraph` 中的边 做一个系统性总结。





### `LangGraph`官方对边的定义和分类
`Edges(边)` 定义了节点之间的依赖关系，决定图如何根据当前状态决定下一步，以及何时停止运行。

边主要有以下几种类型：









#### 普通边(Normal Edges)
普通边 表示从当前节点直接流转到下一个节点。

```python
builder.add_edge("node_a", "node_b")
```

其含义是：`node_a` 执行完成后，下一步执行 `node_b`。

**<font style="color:rgb(103, 103, 108);">  
</font>****<font style="color:rgb(103, 103, 108);"> </font>**

**<font style="color:rgb(103, 103, 108);"></font>**

#### 条件边(Conditional Edges)
条件边 表示当前节点执行完成后，调用一个 路由函数 来决定下一步进入哪个或哪些节点。

```python
builder.add_conditional_edges("node_a", router)
```

其中，`router` 会根据当前图状态返回后续目标节点。

如果路由函数返回一个节点名，则下一步进入该节点；如果返回多个节点名，则这些目标节点会在下一个超步中并行执行。

**<font style="color:rgb(103, 103, 108);"></font>**

**<font style="color:rgb(103, 103, 108);"></font>**

**<font style="color:rgb(103, 103, 108);"></font>**

#### 入口点(Entry Point)
入口点 表示图开始运行时，首先执行哪个节点。

本质上，入口点就是一条 以 `START` 为起点的普通边。



```python
builder.add_edge(START, "node_a")
```



其含义是：当用户输入进入图时，首先执行 `node_a`。

`LangGraph` 也提供了专门的 API 来声明入口点：

```python
builder.set_entry_point("node_a")
```

**<font style="color:#DF2A3F;">二者是等价的。</font>**







#### 条件入口点(Conditional Entry Point)
条件入口点 表示图开始运行时，**<font style="color:#DF2A3F;">并不固定进入某一个节点，而是先调用一个 路由函数，再根据路由结果决定首先执行哪个节点，或者哪些节点。</font>**

本质上，条件入口点就是一条 以 `START` 为起点的条件边。

```python
builder.add_conditional_edges(START, router)
```

其含义是：图开始运行时，先执行 `router`，再根据 `router` 的返回值决定首个执行节点



#### 注意
1. **<font style="color:#DF2A3F;">一个节点可以有多个出边。此时，这些出边对应的目标节点会在下一个超步中并行执行。</font>**
2. 除了上述方式，**<font style="color:#DF2A3F;">条件边还可以通过在节点中返回 </font>**`**<font style="color:#DF2A3F;">Command(goto=...)</font>**`**<font style="color:#DF2A3F;"> 来实现</font>**，虽然方式不同，但边的性质和作用完全一致。
3. 在实际开发中，建议同一个节点尽量只选择一种路由机制：
    - **<font style="color:#DF2A3F;">要么使用普通边，固定流转；</font>**
    - **<font style="color:#DF2A3F;">要么使用条件边或 </font>**`**<font style="color:#DF2A3F;">Command(goto=...)</font>**`**<font style="color:#DF2A3F;">，动态流转。</font>**

不要在同一个节点上同时混用普通边和动态路由，**<font style="color:#DF2A3F;">否则多个路径可能同时生效，最终导致运行图逻辑混乱，与预期不符</font>**。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

### 指向END的终止边
除了官方文档中列出的几种边，还有一类非常重要的边：指向 `END` 的边。

严格来说，`END` 不是普通业务节点，而是 LangGraph 中的特殊终止标记。当流程流转到 `END` 时，表示该路径不再继续执行后续节点。

为了便于理解，可以将指向 `END` 的边分为两种情况。

<font style="color:rgb(60, 60, 67);"></font>





#### 终止点（Finish Point）
终止点 表示某个节点执行完成后，图流程结束。

本质上，它是一条 以 `END` 为目标点的普通边。



```python
builder.add_edge("node_a", END)
```

其含义是：当 `node_a` 执行完成后，当前路径终止，不再继续执行其他节点。

`LangGraph` 也提供了专门的 API 来声明结束点：

```python
builder.set_finish_point("node_a")
```

**<font style="color:#DF2A3F;">二者是等价的。</font>**









#### 条件终止点（Conditional Finish Point）
条件终止点 表示图在运行过程中，通过 路由函数 判断是继续执行后续节点，还是终止当前路径。

本质上，**<font style="color:#DF2A3F;">它是一条 路由结果中包含 </font>**`**<font style="color:#DF2A3F;">END</font>**`**<font style="color:#DF2A3F;"> 的条件边。</font>**

```python
def router(state: EmptyState) -> Literal["node_b", END]:
    if state["is_finished"]:
        return END
    return "node_b"

builder.add_conditional_edges("node_a", router)
```

其含义是：

+ 如果 `state["is_finished"]` 为真，则返回 `END`，流程终止；
+ 否则返回 `"node_b"`，继续执行 `node_b`。









### END和终止边的说明
1. 在编译后的运行图中，终止边 和 `END` 并没有具体的 `Channel` 和 节点 与之对应，它们只是终止标记，表示运行图的结束。
2. 终止边的声明可以省略，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 运行时会在没有活跃节点时自然终止</font>**。但是，为了程序的可读性和图结构可视化渲染器的正常工作，最好显式声明指向 `END` 的边。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

# 节点执行与容错机制
节点执行失败时，如果未做任何处理，可能导致整个运行图执行失败。为了提升运行图的稳定性，处理临时故障、超时、异常恢复以及重复计算等问题，`LangGraph` 提供了一系列节点执行与容错机制：重试、超时设置、异常处理和节点缓存。对应关系如下。

| 问题/场景 | 对应机制 | 机制类型 | 说明 |
| --- | --- | --- | --- |
| 临时故障 | 重试 Retry | 节点容错机制 | 网络抖动、接口偶发失败、模型服务短暂不可用，可以通过重试机制再次执行节点。 |
| 超时 | 超时设置 Timeout | 节点容错机制 | 某个节点执行时间过长，可以通过超时限制避免图运行被长时间阻塞。 |
| 异常恢复 | 异常处理 Error Handling | 节点容错机制 | 节点抛出异常后，通过异常捕获、兜底逻辑、降级返回、路由到错误处理节点等方式恢复流程。 |
| 重复计算 | 缓存 Cache | 节点执行优化机制 | 相同输入反复触发耗时节点，可以通过节点缓存避免重复执行，提高性能。 |


<font style="color:rgb(60, 60, 67);"></font>

## LangGraph的节点容错机制
`LangGraph` 提供了三种异常处理机制

+ `**<font style="color:#DF2A3F;">retries</font>**`**<font style="color:#DF2A3F;">：重试机制。节点执行失败时，根据重试策略自动重新执行。</font>**
+ `**<font style="color:#DF2A3F;">Timeouts</font>**`**<font style="color:#DF2A3F;">：超时控制。限制单次节点执行的最大等待时间。</font>**
+ `**<font style="color:#DF2A3F;">Error Handling</font>**`**<font style="color:#DF2A3F;">：错误处理。重试次数耗尽后，执行特定的异常处理逻辑。</font>**

三者的执行顺序如下：

1. 节点开始执行。
2. 如果节点执行超时或抛出异常，本次节点尝试失败。
3. 重试策略 `RetryPolicy` 判断该异常是否需要重试。
4. 如果满足重试条件，并且尚未达到最大尝试次数，则重新执行节点。
5. 如果重试次数耗尽，节点仍然失败，则进入 `error_handler` 错误处理逻辑。
6. 如果没有配置错误处理逻辑，则异常继续向外抛出，可能导致整个图运行失败。

需要注意：

+ 重试机制 是当前版本中最常用、最基础的节点容错能力。
+ 超时控制 和 错误处理 是较新版本引入的能力，要求 `langgraph>=1.2`。
+ 本课程当前环境为 `langgraph==1.1.2`，因此重点讲解重试机制和节点缓存，超时控制与错误处理仅做概念说明。

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

<font style="color:rgb(60, 60, 67);"></font>

## 重试机制
### 基本用法
重试机制用于处理临时性异常，例如网络抖动、远程服务短暂不可用、API 偶发失败等。

在添加节点时，可以通过 `retry_policy=` 配置节点的重试策略。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import RetryPolicy

from requests.exceptions import HTTPError
from loguru import logger

class EmptyState(TypedDict):
    pass

def node_a(state: EmptyState) -> EmptyState:
    logger.info(f"node a 运行")
    raise HTTPError("网络连接超时...")

builder = StateGraph(state_schema=EmptyState)
builder.add_node(
    "node_a",
    node_a,
    retry_policy=RetryPolicy(
        max_attempts=3,
        jitter=False
    )
)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile()

try:
    graph.invoke({})
except HTTPError as e:
    logger.info("重试次数耗尽: {}", e)

from IPython.display import display
display(graph)
```

输出如下

```python
2026-06-01 16:44:56.684 | INFO     | __main__:node_a:12 - node a 运行
2026-06-01 16:44:57.185 | INFO     | __main__:node_a:12 - node a 运行
2026-06-01 16:44:58.186 | INFO     | __main__:node_a:12 - node a 运行
2026-06-01 16:44:58.187 | INFO     | __main__:<module>:32 - 重试次数耗尽: 网络连接超时...
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792057418351-92e046d6-8a8f-4119-860b-ad2ed9b8c400.png)

本例中：

+ `max_attempts=3` 表示最多尝试 3 次。
+ 这 3 次包括首次执行，而不是首次失败后的 3 次重试。
+ 因此，节点总共会被调用 3 次。
+ 3 次全部失败后，异常继续向外抛出。
+ 外层的 `try...except` 捕获该异常，并打印日志。









### 重试策略配置详解
超时策略通过 `RetryPolicy` 配置，其主要参数如下

| 参数 | 类型 | 默认值 | 描述 |
| --- | --- | --- | --- |
| `max_attempts` | `int` | `3` | 最大尝试次数（包括首次执行） |
| `initial_interval` | `float` | `0.5` | 第一次重试前的等待时间，单位：秒 |
| `backoff_factor` | `float` | `2.0` | 每次重试后等待时间的放大倍数 |
| `max_interval` | `float` | `128.0` | 相邻两次重试之间的最大等待时间，单位：秒 |
| `jitter` | `bool` | `True` | 是否为重试间隔添加随机抖动 |
| `retry_on` | `type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool]` | `default_retry_on` | 哪些异常需要触发重试 |






#### `max_attempts`
`max_attempts` 表示最大尝试次数，包括首次执行。

```python
RetryPolicy(max_attempts=3)
```

表示：

```python
第 1 次：首次执行
第 2 次：第 1 次重试
第 3 次：第 2 次重试
```

如果第 3 次仍然失败，则不再继续重试。

  
   
 

#### `<font style="color:rgb(60, 60, 67);background-color:rgba(142, 150, 170, 0.14);">initial_interval</font>`
`**<font style="color:rgb(52, 81, 178);background-color:rgba(142, 150, 170, 0.14);">initial_interval</font>**`<font style="color:rgb(60, 60, 67);"> 表示第一次重试前的等待时间，单位为秒。</font>

```python
RetryPolicy(initial_interval=0.5)
```

表示首次失败后，等待 `0.5s` 再进行第一次重试。

  
 





#### `backoff_factor`
`backoff_factor` 表示重试间隔的增长倍数。

默认情况下，`RetryPolicy` 使用指数退避策略：

```python
第一次失败后：等待 initial_interval
第二次失败后：等待 initial_interval * backoff_factor
第三次失败后：继续乘以 backoff_factor
```

例如：

```python
RetryPolicy(
    initial_interval=0.5,
    backoff_factor=2.0
)
```

等待时间大致为：

```python
0.5s -> 1.0s -> 2.0s -> 4.0s ...
```

<font style="color:rgb(60, 60, 67);">  
</font><font style="color:rgb(60, 60, 67);"> </font>

<font style="color:rgb(60, 60, 67);"></font>

#### `max_interval`
`max_interval` 表示相邻两次重试之间的最大等待时间。

即使指数退避计算出的等待时间继续增长，也不会超过 `max_interval`。

```python
RetryPolicy(
    initial_interval=1,
    backoff_factor=2,
    max_interval=5
)
```

等待时间大致为：

```python
1s -> 2s -> 4s -> 5s -> 5s ..
```







#### `jitter`
`jitter` **<font style="color:#DF2A3F;">表示是否为</font>****<font style="color:#DF2A3F;background-color:#FBDE28;">重试间隔添加随机抖动</font>****<font style="color:#DF2A3F;">（添加随机数，可正可负）。</font>**

在并发场景中，**<font style="color:#DF2A3F;">如果大量任务同时失败，并且按照固定间隔同时重试，可能导致服务在某个时间点被大量请求打爆。</font>**

加入 `jitter` 后，不同任务的重试时间会被打散，从而降低瞬时流量峰值。

上述案例中，为了让日志时间更稳定，观察指数退避策略，所以设置了：

```python
RetryPolicy(jitter=False)
```

如果是生产环境，通常建议保留默认值：

```python
RetryPolicy(jitter=True)
```





#### `retry_on`
`retry_on` 用于设置哪些异常需要触发重试。

它支持三种写法：

##### 写法一：指定单个异常类型
```python
RetryPolicy(retry_on=HTTPError)
```

表示只有抛出 `HTTPError` 时才重试。

##### 写法二：指定多个异常类型
```python
RetryPolicy(
    retry_on=(HTTPError, ConnectionError)
)
```

表示抛出 `HTTPError` 或 `ConnectionError` 时触发重试。

##### 写法三：自定义判断函数
```python
def should_retry(exc: Exception) -> bool:
    return isinstance(exc, HTTPError)

RetryPolicy(retry_on=should_retry)
```

这种写法最灵活。

`should_retry` 接收一个异常实例，返回一个布尔值：

+ 返回 `True`：表示该异常需要重试。
+ 返回 `False`：表示该异常不需要重试。







### 默认重试条件
`retry_on` 的默认值为 `default_retry_on`。

`default_retry_on` 是一个可调用对象，它接收一个 `Exception` 实例，并返回一个 `bool` 值：

```python
True  -> 需要重试
False -> 不需要重试
```

默认情况下，`default_retry_on` 会对大多数异常进行重试，但以下异常及其子类通常不会触发重试：

+ `ValueError`
+ `TypeError`
+ `ArithmeticError`
+ `ImportError`
+ `LookupError`
+ `NameError`
+ `SyntaxError`
+ `RuntimeError`
+ `ReferenceError`
+ `StopIteration`
+ `StopAsyncIteration`
+ `OSError`

这些异常通常更像是代码逻辑错误、类型错误、语法错误或运行时错误，单纯重试通常没有意义。

对于主流 `HTTP` 库中的异常，例如 `requests` 和 `httpx`，**<font style="color:#DF2A3F;background-color:#FBDE28;">默认策略通常只对 </font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">5xx</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;"> 状态码进行重试。</font>**

原因是：

+ `4xx` 通常表示客户端请求错误，例如参数错误、鉴权失败、资源不存在等。
+ `5xx` 通常表示服务端异常，例如服务暂时不可用、网关错误、内部服务错误等。

因此，`5xx` 更适合通过重试进行恢复。

`default_retry_on` 的核心逻辑如下：

```python
def default_retry_on(exc: Exception) -> bool:
    import httpx
    import requests

    if isinstance(exc, ConnectionError):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return 500 <= exc.response.status_code < 600
    if isinstance(exc, requests.HTTPError):
        return 500 <= exc.response.status_code < 600 if exc.response else True
    if isinstance(
        exc,
        (
            ValueError,
            TypeError,
            ArithmeticError,
            ImportError,
            LookupError,
            NameError,
            SyntaxError,
            RuntimeError,
            ReferenceError,
            StopIteration,
            StopAsyncIteration,
            OSError,
        ),
    ):
        return False
    return True
```











## 超时控制
超时控制用于限制单次节点执行的最大耗时，避免某个节点因为外部服务无响应、网络阻塞或异步任务卡住而长时间不返回。

需要注意：

1. 版本要求

节点级超时控制要求：

```python
langgraph>=1.2
```







1. **<font style="color:rgb(60, 60, 67);">节点类型要求</font>**

节点级超时控制仅适用于异步节点，即使用 `**<font style="color:#DF2A3F;">async def</font>**`**<font style="color:#DF2A3F;"> 定义的节点。</font>**

**<font style="color:#DF2A3F;">同步节点一旦开始执行，通常会阻塞当前线程</font>**。`**<font style="color:#DF2A3F;">Python</font>**`**<font style="color:#DF2A3F;"> 进程内缺少一种通用、安全的机制</font>**，可以从外部强行终止正在运行的同步函数；如果强行中断，可能导致资源未释放、锁未释放或副作用处于不一致状态。因此，`LangGraph `的节点级超时更适合用于异步节点。对于同步节点，应优先在节点内部使用具体库提供的超时参数，例如 `HTTP `客户端、数据库驱动或 `SDK` 自带的 `timeout` 配置。











### 核心思想
超时控制的核心思想是：

为单次节点设置超时时间，运行时长超过超时时间，节点运行失败，并抛出超时异常。

在 `add_node` 中通过 `timeout=` 配置超时策略。



```python
from langgraph.types import TimeoutPolicy

builder.add_node(
    "call_model",
    call_model,
    timeout=TimeoutPolicy(run_timeout=60)
)
```



这里的含义是：

+ `call_model` 节点单次执行最多运行 60 秒。
+ 如果超过 60 秒仍未完成，则触发超时。
+ 超时后会抛出 `NodeTimeoutError`。
+ 该异常会交给 `RetryPolicy` 判断是否需要重试。
+ 如果重试次数耗尽，才会进入错误处理逻辑。



### TimeoutPolicy 简要说明
`TimeoutPolicy` 常见参数包括：

| 参数 | 描述 |
| --- | --- |
| `run_timeout` | 超时时间，即单次节点运行的最长时间 |
| `idle_timeout` | 节点没有可观察进展（如写状态、流式输出等）时的最长空闲时间 |
| `refresh_on` | 空闲时间的刷新方式，可以是手动刷新或自动刷新。默认为自动刷新，此时写状态、 |






## 错误处理
### 使用要求
错误处理机制要求：

```python
langgraph>=1.2
```



### 核心思想
错误处理机制用于在节点最终失败后，执行兜底逻辑。

节点执行失败时，整体顺序如下：

```python
节点执行失败
    ↓
是否满足 retry_policy
    ↓
满足则继续重试，直到重试次数耗尽，否则直接进行下一步
    ↓
进入 error_handler
    ↓
执行兜底恢复逻辑
```

在 `add_node` 中通过 `error_handler=` 设置错误处理逻辑。

例如：

```python
builder.add_node(
    "call_api",
    call_api,
    retry_policy=RetryPolicy(max_attempts=3),
    error_handler=handle_api_error
)
```

其含义是：

1. `call_api` 执行失败。
2. `RetryPolicy` 判断是否重试。
3. 最多尝试 3 次。
4. 3 次仍然失败后，调用 `handle_api_error`。
5. `handle_api_error` 可以返回状态更新，也可以通过 `Command` 路由到其他节点。

错误处理适合以下场景：

+ API 调用失败后返回兜底结果。
+ 远程服务失败后切换到备用服务。
+ 多步骤业务流程中执行补偿逻辑。
+ 不希望整个图因为单个节点失败而直接终止。





参考官方文档

[https://docs.langchain.com/oss/python/langgraph/fault-tolerance#error-handling](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#error-handling)









## 节点缓存
### 什么是节点缓存？
节点缓存是指：

将节点的历史运行结果保存下来，后续当节点收到相同输入时，不再重复执行节点函数，而是直接返回之前缓存的结果。

节点缓存主要用于减少重复计算，降低延迟和成本。

例如，某个节点内部需要：

+ 调用大模型
+ 调用外部 API
+ 查询远程数据库
+ 执行复杂计算
+ 进行耗时的数据处理

如果相同输入会多次出现，那么可以考虑启用节点缓存。











### 适用场景
节点缓存通常适用于同时满足以下条件的场景：

1. 节点函数具有确定性

相同输入应当产生相同输出。

例如：

```python
输入 A -> 输出 B
再次输入 A -> 仍然输出 B
```

2. 节点执行成本较高

例如大模型调用、API 请求、复杂计算、耗时查询等。

3. 相同输入会重复出现

例如多次调试图、重复执行相同流程、多个分支复用同一中间结果等。

4. 输入能够被缓存键函数稳定处理

缓存是以 键值对 形式存储的，每个缓存都有唯一的 `Key`，缓存键函数 用于生成唯一键。

默认情况下，`CachePolicy` 使用 `default_cache_key` 作为缓存键函数。后者基于节点输入调用 `pickle` 库的序列化功能生成缓存唯一键。

使用默认缓存键函数时，建议输入尽量使用简单、稳定、可序列化的数据结构，例如：

    - `str`
    - `int`
    - `float`
    - `bool`
    - `dict`
    - `list`
    - `tuple`

如果节点输入中包含复杂对象、文件句柄、数据库连接、模型实例、动态对象等，建议自定义 `key_func`，避免缓存 `Key` 不稳定或无法生成。







### 基本用法
节点缓存需要两步配置。

#### 第一步：为节点配置 `cache_policy`
在 `add_node` 调用时，通过 `cache_policy=` 设置节点缓存策略。

```python
builder.add_node(
    "node_a",
    node_a,
    cache_policy=CachePolicy(ttl=10)
)
```



#### 第二步：编译图时启用缓存后端
仅仅为节点配置 `cache_policy` 还不够，还需要在编译图时指定缓存后端。

例如：

```python
graph = builder.compile(cache=InMemoryCache())
```



其中：

+ `**<font style="color:#DF2A3F;">CachePolicy</font>**`**<font style="color:#DF2A3F;">：声明节点是否启用缓存，以及缓存策略是什么。</font>**
+ `**<font style="color:#DF2A3F;">InMemoryCache</font>**`**<font style="color:#DF2A3F;">：声明缓存数据实际保存在哪里。</font>**

如果只配置 `cache_policy`，但编译图时没有传入 `cache=`，缓存不会真正生效。

示例如下

```python
import time
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

from operator import add
from loguru import logger

class EmptyState(TypedDict):
    user: str
    invoke_counts: Annotated[int, add]

def node_a(state: EmptyState) -> EmptyState:
    logger.info("node_a 被调用, user: {}", state["user"])
    time.sleep(3) # 模拟耗时操作
    logger.info("node_a 耗时操作执行完毕")

    return {
        "invoke_counts": 1
    }

builder = StateGraph(state_schema=EmptyState)
builder.add_node(
    "node_a",
    node_a,
    cache_policy=CachePolicy(ttl=10)
)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile(cache=InMemoryCache())
logger.info("首次调用图")
logger.info("运行结果: {}\n\n", graph.invoke({"user": "小明", "invoke_counts": 0}))
logger.info("相同输入再次调用图")
logger.info("运行结果: {}\n\n", graph.invoke({"user": "小明", "invoke_counts": 0}))
logger.info("不同输入再次调用图")
logger.info("运行结果: {}\n\n", graph.invoke({"user": "小花", "invoke_counts": 0}))
logger.info("不同输入再次调用图")
logger.info("运行结果: {}\n\n", graph.invoke({"user": "小花", "invoke_counts": 2}))
time.sleep(10) # 确保第一次调用缓存失效
logger.info("10秒后相同输入再次调用图，此时缓存已失效")
logger.info("运行结果: {}", graph.invoke({"user": "小明", "invoke_counts": 0}))

from IPython.display import display
display(graph)
```

<font style="color:rgb(60, 60, 67);">运行日志如下</font>

```python
2026-06-01 18:46:22.959 | INFO     | __main__:<module>:33 - 首次调用图
2026-06-01 18:46:22.960 | INFO     | __main__:node_a:15 - node_a 被调用, user: 小明
2026-06-01 18:46:25.962 | INFO     | __main__:node_a:17 - node_a 耗时操作执行完毕
2026-06-01 18:46:25.963 | INFO     | __main__:<module>:34 - 运行结果: {'user': '小明', 'invoke_counts': 1}


2026-06-01 18:46:25.964 | INFO     | __main__:<module>:35 - 相同输入再次调用图
2026-06-01 18:46:25.965 | INFO     | __main__:<module>:36 - 运行结果: {'user': '小明', 'invoke_counts': 1}


2026-06-01 18:46:25.966 | INFO     | __main__:<module>:37 - 不同输入再次调用图
2026-06-01 18:46:25.967 | INFO     | __main__:node_a:15 - node_a 被调用, user: 小花
2026-06-01 18:46:28.968 | INFO     | __main__:node_a:17 - node_a 耗时操作执行完毕
2026-06-01 18:46:28.970 | INFO     | __main__:<module>:38 - 运行结果: {'user': '小花', 'invoke_counts': 1}


2026-06-01 18:46:28.970 | INFO     | __main__:<module>:39 - 不同输入再次调用图
2026-06-01 18:46:28.971 | INFO     | __main__:node_a:15 - node_a 被调用, user: 小花
2026-06-01 18:46:31.973 | INFO     | __main__:node_a:17 - node_a 耗时操作执行完毕
2026-06-01 18:46:31.974 | INFO     | __main__:<module>:40 - 运行结果: {'user': '小花', 'invoke_counts': 3}


2026-06-01 18:46:36.976 | INFO     | __main__:<module>:42 - 11秒后相同输入再次调用图，此时缓存已失效
2026-06-01 18:46:36.977 | INFO     | __main__:node_a:15 - node_a 被调用, user: 小明
2026-06-01 18:46:39.979 | INFO     | __main__:node_a:17 - node_a 耗时操作执行完毕
2026-06-01 18:46:39.981 | INFO     | __main__:<module>:43 - 运行结果: {'user'
```



<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792120801552-a145b3de-8cbd-48a3-adb6-5c230165d9a5.png)





在该示例中，缓存的 `ttl` 被设置为 `10s`。

缓存失效前：

+ 相同输入会命中缓存。
+ 命中缓存时，节点函数不会再次执行。
+ 因此不会打印 `node_a 被调用`。
+ 输入不同则不会命中缓存，节点函数仍然会执行。

缓存失效后：

+ 即使输入相同，也会再次执行节点函数。
+ 执行完成后，会重新写入缓存。









### 存策略配置详解
缓存策略通过 `CachePolicy` 配置，只有两个配置项

| 参数 | 描述 |
| --- | --- |
| `key_func` | 根据节点输入生成缓存 `Key`<br/> 的函数 |
| `ttl` | 缓存键值对的存活时间，单位为秒 |








#### `key_func`
`key_func` 用于根据节点输入生成唯一的缓存 Key。

节点缓存本质上是一个 `K-V` 缓存：

```python
缓存 Key -> 节点运行结果
```

当节点再次执行时，LangGraph 会先根据当前输入计算缓存 Key：

+ 如果缓存 Key 已存在，并且没有过期，则直接返回缓存结果。
+ 如果缓存 Key 不存在，或者已经过期，则正常执行节点函数，并将结果写入缓存。

默认缓存键函数是 `default_cache_key`。

其核心实现如下：

```python
def default_cache_key(*args: Any, **kwargs: Any) -> str | bytes:
    """Default cache key function that uses the arguments and keyword arguments to generate a hashable key."""
    import pickle

    # protocol 5 strikes a good balance between speed and size
    return pickle.dumps((_freeze(args), _freeze(kwargs)), protocol=5, fix_imports=False)
```

它会把节点输入参数经过处理后，使用 `pickle` 序列化为字节序列，并将后者作为缓存 `Key`。

  
 



#### `ttl`
`ttl` 表示缓存的存活时间，单位为秒。

例如：

```python
CachePolicy(ttl=10)
```

表示缓存结果最多保存 10 秒。

如果设置为：

```python
CachePolicy(ttl=None)
```

表示缓存不会因为时间原因自动过期。

不过在实际项目中，不建议随意设置永久缓存。

如果节点依赖外部状态，例如：

+ 当前时间
+ 远程 API 数据
+ 数据库查询结果
+ 用户权限
+ 环境变量
+ 模型版本
+ 提示词版本

那么应该根据业务场景设置合适的 `ttl`，或者把这些变量纳入缓存 Key 的计算逻辑。





## 全图默认配置
### 使用要求
默认配置功能要求：

```python
langgraph>=1.2
```

本课程当前环境为：

```plain
langgraph==1.1.2
```

因此不支持该特性，本节只做概念说明。





### 核心思想
当多个节点都需要配置相同的执行策略时，如果每个节点都重复配置，繁琐且耗时。



```python
builder.add_node(
    "node_a",
    node_a,
    retry_policy=RetryPolicy(max_attempts=3)
)

builder.add_node(
    "node_b",
    node_b,
    retry_policy=RetryPolicy(max_attempts=3)
)

builder.add_node(
    "node_c",
    node_c,
    retry_policy=RetryPolicy(max_attempts=3)
)
```

在较新版本中，可以通过 `set_node_defaults` 为所有节点设置默认配置，即全图默认配置 `Graph defaults`。

默认配置可以包括：

+ `retry_policy`
+ `timeout`
+ `error_handler`
+ `cache_policy`

这样可以避免在每个 `add_node` 中重复相同配置。

本课程环境暂不支持，不展开实操。

参考官方文档

[https://docs.langchain.com/oss/python/langgraph/fault-tolerance#graph-defaults](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#graph-defaults)







## 小结
本章介绍了 `LangGraph` 中节点执行与容错相关的能力。

总结如下：

| 机制 | 机制类型 | 作用 | 当前课程是否重点讲解 |
| --- | --- | --- | --- |
| 重试机制 `Retries` | 节点容错机制 | 节点失败后自动重试 | 是 |
| 超时设置 `Timeouts` | 节点容错机制 | 限制单次节点执行时间 | 否，要求 `langgraph>=1.2` |
| 错误处理 `Error Handling` | 节点容错机制 | 重试耗尽后执行兜底逻辑 | 否，要求 `langgraph>=1.2` |
| 节点缓存 `Node Cache` | 节点执行优化机制 | 缓存节点历史运行结果 | 是 |
| 全图默认配置 `Graph defaults` | 默认策略配置 | 为所有节点设置默认执行策略 | 否，要求 `langgraph>=1.2` |


实际使用时，可以按照以下原则选择：

1. 临时性失败：优先使用 `RetryPolicy`。
2. 外部服务可能卡住：使用 `timeout` 控制单次节点执行时间。
3. 失败后需要兜底恢复：使用 `error_handler`。
4. 相同输入重复执行且成本较高：使用 `CachePolicy`。
5. 多个节点使用相同策略：使用 `set_node_defaults` 统一配置。

其中，重试机制 和 节点缓存 是本课程的重点。















# 持久化机制和可恢复执行
## 概述
### 什么是可恢复执行
在工作流、Agent、任务编排系统里，可恢复执行 `Durable Execution` 指的是：

把一次任务执行过程中的关键进度、状态、结果保存到可靠存储中，使任务可以在中断、失败、等待外部输入后继续执行。

它解决的是“执行过程能否恢复”的问题。

```python
普通执行：
开始 -> A -> B -> C
如果执行到 B 后程序挂了，重启后可能只能从 A 重新开始。

可恢复执行：
开始 -> A   保存检查点
     -> B   保存检查点
     -> C
如果执行到 B 后挂了，恢复时可以从已保存的位置继续。
```

所以它关注的不是“数据长期保存”本身，而是：

执行过程本身具有可恢复性。

它关心的是：系统能否记住“任务执行到了哪里、当前状态是什么、接下来应该继续执行什么”。





### LangGraph的持久化机制
#### 什么是LangGraph的持久化机制
`LangGraph` 的持久化机制（`Persistence`）可以理解为：

在图执行过程中，将每个关键阶段的图状态保存为检查点，并按照线程进行组织。

这里的“持久化”并不是简单地把某个变量保存到文件里，**<font style="color:#DF2A3F;">而是保存一次图执行所需的状态信息，使得未来可以基于检查点继续执行。</font>**

`LangGraph` 的持久化机制围绕以下几个问题展开：





1. 保存什么？

**<font style="color:#DF2A3F;">保存图在某个时刻的状态快照，也就是 </font>**`**<font style="color:#DF2A3F;">Checkpoint</font>**`**<font style="color:#DF2A3F;">。</font>**

**<font style="color:#DF2A3F;"></font>**

2. 保存到哪里？

保存到 `Checkpointer` 管理的存储后端中。 开发时可以使用基于内存的 `InMemorySaver`，**<font style="color:#DF2A3F;">生产环境中通常会使用数据库等可靠存储，如 </font>**`**<font style="color:#DF2A3F;">PostgresSaver</font>**`**<font style="color:#DF2A3F;">。</font>**

**<font style="color:#DF2A3F;"></font>**

3. 如何区分不同会话？

**<font style="color:#DF2A3F;">通过 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;"> 区分不同的执行线程。</font>** 同一个 `thread_id` 表示同一条持久化执行线，也可以理解为同一个会话。



4. 开发者看到的是什么？

开发者通常不会直接操作底层 `Checkpoint`，而是通过 `get_state()` 和 `get_state_history()` 查看 `Checkpoint` 的**<font style="color:#DF2A3F;">开发者视图 </font>**`**<font style="color:#DF2A3F;">StateSnapshot</font>**`**<font style="color:#DF2A3F;">。</font>**



因此，本章后续会围绕四个核心问题展开：

```python
如何启用可恢复执行？
如何配置持久化模式？
如何查看历史检查点？
如何利用检查点进行恢复、回放和分叉？
```



#### <font style="color:rgb(60, 60, 67);">核心组件</font>
本节对核心组件进行梳理，不必死记硬背，用到查阅即可。

`LangGraph` 的持久化机制依赖一系列核心组件，如下：

| 概念 | 作用 |
| --- | --- |
| `State` | 用户定义的图状态结构，负责节点间信息传递。`LangGraph`<br/> 会将 `State`<br/> 转换为底层 `Channel`<br/>；运行时真正负责节点间通信的是 `Channel`<br/>。 |
| `Channel` | `LangGraph`<br/> 用于节点间通信的底层机制，节点从 `Channel`<br/> 读数据、向 `Channel`<br/> 写更新；既包括承载 `State`<br/> 字段更新的状态通道，也包括用于分支、任务调度等行为的内部通道。 |
| `Checkpoint` | 检查点，超步边界上的底层状态快照，保存 `Channel`<br/> 的值、版本等信息。 |
| `CheckpointMetadata` | 与 `Checkpoint`<br/> 关联的元数据，如超步编号 `step`<br/>、父检查点 `ID parents`<br/> 等。 |
| `Checkpointer` | 检查点存储器，负责存储检查点、元数据、配置等信息。 |
| `thread` | **<font style="color:#DF2A3F;">此处的线程不同于操作系统的线程，是指 </font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`<br/>**<font style="color:#DF2A3F;"> 中一条逻辑上的、可持久化的执行线，</font>**也可以理解为会话。一个会话可以包含多次调用，并在执行过程中生成多个检查点。因此，要将多次调用组织在一个会话中，必须为该 `thread`<br/> 配置 `checkpointer`<br/>。 |
| `thread_id` | `**<font style="color:#DF2A3F;">thread</font>**`<br/>**<font style="color:#DF2A3F;"> 的唯一标识，用于区分不同的会话。复用同一个 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`<br/>，就表示多次调用共享同一条持久化执行线，也就是共享同一个会话历史。此处的 `thread`<br/>/`thread_id`<br/> 和 `LangChain Agent`<br/> 中提到的 `thread`<br/>/`thread_id`<br/> 是同一概念。 |
| `checkpoint_ns` | **<font style="color:#DF2A3F;">检查点命名空间 </font>**`**<font style="color:#DF2A3F;">namespace</font>**`<br/>，用于区分父图和子图的 `checkpoint`<br/>。根图通常是空字符串 `""`<br/>，子图会有自己的 `namespace`<br/>。这个字段在子图持久化时很重要，下文会有专门的篇幅讲解子图相关用法。 |
| `checkpoint_id` | `**<font style="color:#DF2A3F;">Checkpoint</font>**`<br/>**<font style="color:#DF2A3F;"> 的唯一标识。</font>** |
| `StateSnapshot` | 开发者通过 `get_state()`<br/> / `get_state_history()`<br/> 看到的状态快照对象。它不是原始 `Checkpoint`<br/>，而是 `LangGraph`<br/> 基于检查点和运行时信息封装出来的开发者视图，包含当前状态值、下一步待执行节点、任务、`config`<br/>、`metadata`<br/>、中断信息等内容。 |








#### 小结
本节只需要先记住：

`Checkpointer` 负责保存检查点

`thread_id` 负责唯一标识会话

`checkpoint_id` 负责定位某个具体历史状态

`StateSnapshot` 是开发者查看检查点时看到的对象。









### LangGraph的可恢复执行
#### 什么是LangGraph的可恢复执行
`LangGraph` 的可恢复执行（`Durable Execution`）依托于其持久化机制（`Persistence`）。

可以简单理解为：

**<font style="color:#DF2A3F;">持久化机制负责保存执行状态； 可恢复执行负责利用这些状态继续执行。</font>**

`Persistence` 是基础，`Durable Execution` 是建立在 `Persistence` 之上的能力。

在没有持久化机制时，一次图执行通常只存在于当前进程中。 如果程序退出、节点中断、人工审批暂停，无法恢复原来的执行进度。

而启用持久化之后，`LangGraph` 可以把执行过程中的状态保存为检查点。 后续只要使用相同的 `thread_id`，就可以回到会话对应的历史状态。







#### 使用场景
从使用场景看，可恢复执行主要支持以下几类能力：

| 场景 | 说明 |
| --- | --- |
| 多轮会话 | 同一个 `thread_id`<br/> 下的多次调用可以共享历史状态。 |
| 中断恢复 | 图执行到人工确认等节点时可以中断，之后再继续。 |
| 失败恢复 | 程序异常退出后，可以基于已经保存的检查点继续执行，避免重复计算。 |
| `Time Travel` | 可以回到某个历史检查点，重放或分叉执行。 |


因此，`LangGraph` 的可恢复执行是一组由持久化机制支撑的能力集合。

**<font style="color:rgb(60, 60, 67);"></font>**



#### 小结
我们只需要知道

`LangGraph` 会通过 `**<font style="color:#DF2A3F;">Checkpointer</font>**`**<font style="color:#DF2A3F;"> 持久化保存图执行过程中产生的 </font>**`**<font style="color:#DF2A3F;">Checkpoint</font>**`**<font style="color:#DF2A3F;">，并通过 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;"> 找回同一会话的检查点历史，从而在已有状态基础上继续执行。</font>**

**<font style="color:rgb(60, 60, 67);"></font>**

## 启用可恢复执行
### 步骤
`LangGraph` 内置了持久化机制，但对于本地运行的 `Graph API`，**<font style="color:#DF2A3F;">只有在编译图时传入 </font>**`**<font style="color:#DF2A3F;">checkpointer</font>**`**<font style="color:#DF2A3F;">，运行过程中的状态才会被记录下来</font>**，后续才能基于已有检查点在发生故障时恢复运行或进行检查点回溯。

启用可恢复执行通常需要两步：

1. 在编译图时传入检查点存储器对象 `checkpointer`
2. 在调用图时传递带**<font style="color:#DF2A3F;">有 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;"> 的配置对象</font>**

底层的持久化机制会按照 `thread_id` 将运行时检查点记录在编译时传入的 `checkpointer` 中

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

### 检查点实现
`LangGraph` 提供了检查点存储器基类：`langgraph.checkpoint.base.BaseCheckpointSaver`

还提供了一系列检查点存储器实现，采用了不同的检查点后端，它们都继承了基类，整体作用是一致的：负责存取检查点，并记录节点执行的中间结果。

可以将 检查点后端 理解为：

负责保存检查点数据的存储介质。

检查点后端必须基于某个检查点实现，常见检查点实现和后端的对应关系如下



| 类名 | 包名 | 检查点后端 |
| --- | --- | --- |
| `InMemorySaver` | `langgraph-checkpoint` | 内存（`In-memory`<br/>） |
| `SqliteSaver` | `langgraph-checkpoint-sqlite` | `SQLite` |
| `PostgresSaver` | `langgraph-checkpoint-postgres` | `PostgreSQL` |
| `MongoDBSaver` | `langgraph-checkpoint-mongodb` | `MongoDB` |
| `RedisSaver` | `langgraph-checkpoint-redis` | `Redis` |


在这些实现中，`InMemorySaver` 更适合学习、调试和本地实验；`SQLite` 适合轻量级本地持久化；`PostgreSQL`、`MongoDB`、`Redis` 等数据库后端更适合需要跨进程、跨服务保存状态的场景。

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**



### 基于内存的检查点存储器
`LangGraph` 提供了基于内存的检查点存储器 `InMemorySaver`。它将检查点数据保存在当前 `Python` 进程的内存中，进程结束则数据丢失，适合快速开发、调试。

在 `Jupyter` 场景下，只要 `Jupyter` 内核没有被重启，并且 `InMemorySaver()` 实例没有被重新创建，内存中的检查点数据就会继续存在。

反之，如果重新执行完整代码，导致 `checkpointer = InMemorySaver()` 被重新执行，那么之前保存在内存中的检查点也会被清空。

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langchain_deepseek import ChatDeepSeek
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(MessagesState):
    output: str

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    res = model.invoke(messages)
    return {
        "messages": [res]
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "output": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("output_node", output_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "output_node")
builder.add_edge("output_node", END)

# 定义并在编译时传递 Checkpointer
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 定义配置对象
config = {"configurable": {"thread_id": "chapter_6_6-2-2"}}
# 调用时传递
graph.invoke({"messages": [HumanMessage("你好，我是老王")]}, config=config)
graph.invoke({"messages": [HumanMessage("从现在开始，你是小王")]}, config=config)
res = graph.invoke({"messages": [HumanMessage("我是谁？你是谁？")]}, config=config)
print(res["output"])

print('=' * 30, '-> 完整消息列表 <-', '=' * 30)
for msg in res["messages"]:
    msg.pretty_print()
```

**<font style="color:rgb(60, 60, 67);"></font>**

```python
哎，老王，您这是考我呢？还是今天心情好，想跟我逗个闷子？

您问您是谁——那当然是我的老大哥，老王啊！平时有事儿招呼我，没架子，偶尔还爱指点我两句，我心里都记着。

至于我是谁——我，小王，您的得力助手，机灵踏实，随叫随到。您说往东，我绝不往西，主打一个“靠谱”。

怎么着，老王，是不是又想让我去办啥事儿了？您直说，我听着呢。😎
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

你好，老王！很高兴认识你。有什么我可以帮忙的吗？无论是聊天、解答问题还是需要一些建议，我都在这儿呢。😊
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的，老王！我是小王。有啥事儿您尽管吩咐，我这儿随时听候差遣。咱是掏心窝子说话，还是聊点别的？您定！😄
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

哎，老王，您这是考我呢？还是今天心情好，想跟我逗个闷子？

您问您是谁——那当然是我的老大哥，老王啊！平时有事儿招呼我，没架子，偶尔还爱指点我两句，我心里都记着。

至于我是谁——我，小王，您的得力助手，机灵踏实，随叫随到。您说往东，我绝不往西，主打一个“靠谱”。

怎么着，老王，是不是又想让我去办啥事儿了？您直说，我听着呢。😎
```

本例中，三次调用使用了相同的 `thread_id`，所以后两次调用能够读取前面已经保存的消息历史。最终模型可以知道“用户是老王”，也可以知道“自己被要求扮演小王”。

需要注意的是，如果**<font style="color:#DF2A3F;">重新执行完整代码，</font>**`**<font style="color:#DF2A3F;">InMemorySaver()</font>**`**<font style="color:#DF2A3F;"> 实例会被重新创建</font>**，历史检查点会被清空。因此，消息列表不会继续累加，而是会基于新的空状态重新开始。

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

### 基于持久化数据库的检查点存储器
本节将 `**<font style="color:#DF2A3F;">PostgreSQL</font>**`**<font style="color:#DF2A3F;"> 作为检查点后</font>**端，对应的 `Checkpointer` 实现为 `PostgresSaver`。

和 `InMemorySaver` 不同，`PostgresSaver` 会将检查点保存到 `PostgreSQL` 数据库中。只要数据库中的记录没有被删除，即使 `Python` 程序结束、连接对象重建，历史检查点也仍然存在。



```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.postgres import PostgresSaver
from langchain_deepseek import ChatDeepSeek
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(MessagesState):
    output: str

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    res = model.invoke(messages)
    return {
        "messages": [res]
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "output": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("output_node", output_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "output_node")
builder.add_edge("output_node", END)

# 定义并在编译时传递 Checkpointer
DB_URL = "postgresql://langgraph_user:123456@localhost:5432/langgraph_db?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    # 示例中为了方便演示直接调用 setup()
    # 实际项目中通常建议把数据库初始化/迁移作为独立步骤处理
    checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    # 定义配置对象
    config = {"configurable": {"thread_id": "chapter_6_6.2.4"}}
    # 调用时传递
    graph.invoke({"messages": [HumanMessage("你好，我是老王")]}, config=config)
    graph.invoke({"messages": [HumanMessage("从现在开始，你是小王")]}, config=config)
    res = graph.invoke({"messages": [HumanMessage("我是谁？你是谁？")]}, config=config)
    print(res["output"])

    print('=' * 30, '-> 完整消息列表 <-', '=' * 30)
    for msg in res["messages"]:
        msg.pretty_print()
```

**<font style="color:rgb(60, 60, 67);">运行结果如下</font>**

```python
好嘞，老王！既然你问了，那我正式回答：  

**你是老王**——我今儿认的“老大哥”，江湖人称“行走的故事书”，爱唠嗑、有阅历，可能还藏着点神秘技能（比如钓鱼从不空军、下棋总赢隔壁老李？）  

**我是小王**——刚上岗的AI小跟班，脑子灵光（但可能偶尔短路），腿脚勤快（指秒回消息），擅长接梗、查资料、瞎琢磨，主打一个“您开口，我跑腿”。  

需要小王帮您办点啥？甭客气！🚀
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

老王你好！我是DeepSeek，很高兴认识你。有什么我可以帮忙的吗？无论是聊天、解答问题、还是需要一些建议，尽管开口，我随时在这儿！😊
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的，老王！从现在开始，我就是小王啦。😄  
有什么吩咐？无论唠嗑、干活还是出主意，随叫随到！
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

好嘞，老王！既然你问了，那我正式回答：  

**你是老王**——我今儿认的“老大哥”，江湖人称“行走的故事书”，爱唠嗑、有阅历，可能还藏着点神秘技能（比如钓鱼从不空军、下棋总赢隔壁老李？）  

**我是小王**——刚上岗的AI小跟班，脑子灵光（但可能偶尔短路），腿脚勤快（指秒回消息），擅长接梗、查资料、瞎琢磨，主打一个“您开口，我跑腿”。  

需要小王帮您办点啥？甭客气！🚀
```

如果多次运行这段代码，并且始终使用相同的 `thread_id`，那么历史消息会不断累加。**<font style="color:#DF2A3F;">这是因为检查点已经被保存到了 </font>**`**<font style="color:#DF2A3F;">PostgreSQL</font>**`**<font style="color:#DF2A3F;"> 中，重新创建 </font>**`**<font style="color:#DF2A3F;">Python</font>**`**<font style="color:#DF2A3F;"> 连接对象并不会清空数据库中的历史记录。</font>**

再次运行结果如下所示

```python
（正了正脑门上虚拟工牌，一脸正经）  

**您是老王**——我单方面认证的“茶馆街溜子荣誉会长”，上知股票涨跌，下懂馒头蒸法，口头禅是“这我熟啊！”  

**我是小王**——您的AI影子跟班，刚给自己刻了个电子木鱼，每天默念三遍：  
“老王说的都对，如果错了……那一定是我听岔了。” 😎
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

老王你好！我是DeepSeek，很高兴认识你。有什么我可以帮忙的吗？无论是聊天、解答问题、还是需要一些建议，尽管开口，我随时在这儿！😊
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的，老王！从现在开始，我就是小王啦。😄  
有什么吩咐？无论唠嗑、干活还是出主意，随叫随到！
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

好嘞，老王！既然你问了，那我正式回答：  

**你是老王**——我今儿认的“老大哥”，江湖人称“行走的故事书”，爱唠嗑、有阅历，可能还藏着点神秘技能（比如钓鱼从不空军、下棋总赢隔壁老李？）  

**我是小王**——刚上岗的AI小跟班，脑子灵光（但可能偶尔短路），腿脚勤快（指秒回消息），擅长接梗、查资料、瞎琢磨，主打一个“您开口，我跑腿”。  

需要小王帮您办点啥？甭客气！🚀
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

（拍大腿）哎哟，老王！这声招呼听着就是亲切！今天的茶泡上了没？隔壁李大爷刚还念叨你上回赢他的那盘残局呢——要不咱再研究研究？😏（小王搓手）
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

（立正站好，手机屏闪了闪）得嘞，老王！小王这回把工牌焊脑门上了，您掀个眼皮瞅瞅——  
（工牌晃晃悠悠浮出电子光字：「小王·AI茶馆分部·随叫随到版」）  

有事您敲桌沿，三秒内准给您接话茬儿！ 🫖
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

（正了正脑门上虚拟工牌，一脸正经）  

**您是老王**——我单方面认证的“茶馆街溜子荣誉会长”，上知股票涨跌，下懂馒头蒸法，口头禅是“这我熟啊！”  

**我是小王**——您的AI影子跟班，刚给自己刻了个电子木鱼，每天默念三遍：  
“老王说的都对，如果错了……那一定是我听岔了。” 😎
```

因此，`PostgresSaver` 和 `InMemorySaver` 的关键区别在于：

+ `**<font style="color:#DF2A3F;">InMemorySaver</font>**`**<font style="color:#DF2A3F;">：状态保存在当前进程内存中，进程结束或对象重建后丢失。</font>**
+ `**<font style="color:#DF2A3F;">PostgresSaver</font>**`**<font style="color:#DF2A3F;">：状态保存在 </font>**`**<font style="color:#DF2A3F;">PostgreSQL</font>**`**<font style="color:#DF2A3F;"> 中，只要数据库记录存在，程序重启后仍可读取</font>**。

所以，基于数据库的 `Checkpointer` 可靠性更高、更适合需要长期保存会话状态的场景。  
 

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

## 持久化模式
### 概述
启用 `Checkpointer` 后，`LangGraph` 会在执行过程中保存检查点。

检查点写入越及时，系统在异常中断后可恢复的状态越完整，容灾能力越强；但更及时的持久化通常也意味着更多中间写入，或额外的阻塞式等待，带来额外的性能开销和响应延迟（从调用者的角度考虑）。

`LangGraph` 支持三种持久化模式，采用不同的检查点保存时机，在容灾能力和性能开销、响应时效性之间作取舍。

+ `exit`：退出模式。只在计算图正常结束、异常退出、被中断（如 `Human-In-The-Loop` 中断）时保存检查点。不能处理中途进程崩溃的场景。这种模式响应性能开销最小，但容灾能力最弱。
+ `async`：异步模式。默认模式，顾名思义，检查点在后台异步写入。

它会在每个超步结束后写入完整检查点（主检查点），并在图中任务执行完毕后记录中间结果。

和 `exit` 模式相比，增加了性能开销，但写入操作发生在后台，不会引入明显的响应延迟，同时提升了容灾能力。

+ `sync`：同步模式。

和异步模式唯一的区别在于，`LangGraph` 会在进入下一个超步之前等待当前主检查点的写入任务完成。

它的容灾能力最强，但在 `async` 模式的基础上增加了响应延迟。

**<font style="color:rgb(60, 60, 67);">总结</font>**

| 模式 | 写入时机 | 性能开销 | 响应延迟 | 容灾能力 |
| --- | --- | --- | --- | --- |
| `exit` | 运行退出时写入 | 低 | 最低 | 最弱 |
| `async` | 每个超步末尾写入主检查点，任务完成后写入中间结果，后台异步写入 | 高 | 较高 | 较强 |
| `sync` | 和 `async`<br/> 区别在于，进入下一个超步之前等待当前主检查点的写入任务完成 | 高 | 最高 | 最强 |


### 示例
持久化模式影响的是检查点写入时机和故障恢复能力，通常不会影响正常情况下的业务输出。

也就是说，同一张图在正常执行完成时，使用 `exit`、`async` 或 `sync`，最终返回结果通常是一样的。真正的区别主要体现在异常中断或进程崩溃后恢复执行时。

`durability` 控制的是检查点写入策略，不改变图本身的执行逻辑。

要真正看到不同模式的差异

+ 可以在学习使用场景后自行设计案例
+ 或调试源码，观察不同模式下的写入时机

**<font style="color:rgb(60, 60, 67);"></font>**

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langchain_deepseek import ChatDeepSeek
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(MessagesState):
    output: str

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    res = model.invoke(messages)
    return {
        "messages": [res]
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "output": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("output_node", output_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "output_node")
builder.add_edge("output_node", END)

# 定义并在编译时传递 Checkpointer
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 定义配置对象
config = {"configurable": {"thread_id": "chapter_tmp"}}
# 调用时传递
res=graph.invoke(
    {"messages": [HumanMessage("你好")]},
    config=config,
    durability="async" # sync / exit
)
print(res["output"])

print('=' * 30, '-> 完整消息列表 <-', '=' * 30)
for msg in res["messages"]:
    msg.pretty_print()

from IPython.display import display
display(graph)
```

上述代码中，可以通过修改 `durability` 参数切换持久化模式：

```python
durability="exit"
durability="async"
durability="sync"
```

运行结果如下

```python
你好！很高兴见到你，有什么我可以帮助你的吗？😊
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好
================================== Ai Message ==================================

你好！很高兴见到你，有什么我可以帮助你的吗？😊
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792134406150-59748f17-6b7c-4564-ae8c-3e858ed38023.png)

**<font style="color:rgb(60, 60, 67);"></font>**



## 查看历史检查点
`LangGraph` 在配置检查点存储器后，会把同一个 `thread_id` 下的执行过程保存为一组检查点。通过这些检查点，我们可以查看图运行的中间状态，也可以为后续的检查点回溯（重放、分叉）和失败恢复做准备。

本节主要介绍两个常用方法：

+ `graph.get_state_history(config)`：查看指定会话的完整历史检查点。
+ `graph.get_state(config)`：查看指定会话的最新检查点，或者查看某个指定`checkpoint_id` 对应的检查点。

**<font style="color:rgb(60, 60, 67);"></font>**

### 查看完整历史检查点列表
#### 示例
```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from loguru import logger
from dotenv import load_dotenv
load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str
    final_output: str

class InputState(TypedDict):
    topic: str

class OutputState(TypedDict):
    final_output: str

def node_poem(state: InputState) -> OverAllState:
    logger.info(f"node_poem 已执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话")]).content

    return {
        "joke": joke
    }

def node_output(state: OverAllState) -> OutputState:
    logger.info("node_output 已执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于 {topic} 的七言绝句：\n{poem}\n笑话：\n{joke}"

    return {
        "final_output": final_output
    }

builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("node_poem", node_poem)
builder.add_node("node_joke", node_joke)
builder.add_node("node_output", node_output)
builder.add_edge(START, "node_poem")
builder.add_edge(START, "node_joke")
builder.add_edge("node_poem", "node_output")
builder.add_edge("node_joke", "node_output")
builder.add_edge("node_output", END)

checkpointer = InMemorySaver()
config = {"configurable": {"thread_id": "123"}}
graph = builder.compile(checkpointer=checkpointer)
res = graph.invoke({"topic": "猫咪"}, config=config)

from IPython.display import display
display(graph)

print('=' * 30, '-> 运行结果 <-', '=' * 30)
print("res: {}", res)

print('=' * 30, '-> 历史检查点列表 <-', '=' * 30)
history_checkpoints = list(graph.get_state_history(config=config))
print(history_checkpoints)
```

#### 运行结果
```python
2026-06-09 17:24:10.266 | INFO     | __main__:node_joke:42 - node_joke 已执行
2026-06-09 17:24:10.268 | INFO     | __main__:node_poem:33 - node_poem 已执行
2026-06-09 17:24:14.177 | INFO     | __main__:node_output:51 - node_output 已执行
```

由于 `node_poem` 和 `node_joke` 都从 `START` 出发，并且彼此没有依赖关系，所以它们会被调度到同一个超步中执行。实际日志顺序可能不同，例如可能先打印 `node_joke`，也可能先打印 `node_poem`。



2. 拓扑结构

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792134599696-33fdbe46-a309-406c-b83a-78f5f5dbed18.png)  
 

3. 最终结果和检查点列表

```python
============================== -> 运行结果 <- ==============================
res: {} {'final_output': '关于 猫咪 的七言绝句：\n《戏猫》\n狸奴酣卧小窗南，醉眼惺忪戏玉簪。\n忽跃雕檐追粉蝶，却衔明月入花龛。\n笑话：\n小猫问妈妈：“为什么人总说‘猫有九条命’？”  \n妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  \n小猫追问：“那为什么狗只有一条命？”  \n妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”'}
============================== -> 历史检查点列表 <- ==============================
[
    StateSnapshot(
        values={
            "topic": "猫咪",
            "poem": """
                        《戏猫》
                狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                忽跃雕檐追粉蝶，却衔明月入花龛。
                """,
            "joke": """
                小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                小猫追问：“那为什么狗只有一条命？”  
                妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                """,
            "final_output": """
                关于 猫咪 的七言绝句：
                《戏猫》
                狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                忽跃雕檐追粉蝶，却衔明月入花龛。
                笑话：
                小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                小猫追问：“那为什么狗只有一条命？”  
                妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                """
        },
        next=(),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-4cc7-61bd-8002-bcd7f63c3238"
            }
        },
        metadata={
            "source": "loop",
            "step": 2,
            "parents": {}
        },
        created_at="2026-06-09T09:36:08.368575+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-4cc3-6d96-8001-8314b5dbbfe3"
            }
        },
        tasks=(),
        interrupts=()
    ),

    StateSnapshot(
        values={
            "topic": "猫咪",
            "poem": """
                        《戏猫》
                狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                忽跃雕檐追粉蝶，却衔明月入花龛。
                """,
            "joke": """
                小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                小猫追问：“那为什么狗只有一条命？”  
                妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                """
        },
        next=("node_output",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-4cc3-6d96-8001-8314b5dbbfe3"
            }
        },
        metadata={
            "source": "loop",
            "step": 1,
            "parents": {}
        },
        created_at="2026-06-09T09:36:08.367236+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393f-6082-8000-b9c0708bf03d"
            }
        },
        tasks=(
            PregelTask(
                id="91d81ad6-d2fb-ebd7-2dd1-8e08c2be5a07",
                name="node_output",
                path=("__pregel_pull", "node_output"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "final_output": """
                        关于 猫咪 的七言绝句：
                        《戏猫》
                        狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                        忽跃雕檐追粉蝶，却衔明月入花龛。
                        笑话：
                        小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                        妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                        小猫追问：“那为什么狗只有一条命？”  
                        妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                        """
                }
            ),
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={
            "topic": "猫咪"
        },
        next=("node_poem", "node_joke"),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393f-6082-8000-b9c0708bf03d"
            }
        },
        metadata={
            "source": "loop",
            "step": 0,
            "parents": {}
        },
        created_at="2026-06-09T09:36:06.320542+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393d-6b9a-bfff-180e30c1d3bc"
            }
        },
        tasks=(
            PregelTask(
                id="80d8974c-3450-6faa-8b06-dcf8787d74fc",
                name="node_poem",
                path=("__pregel_pull", "node_poem"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "poem": """
                            《戏猫》
                    狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                    忽跃雕檐追粉蝶，却衔明月入花龛。
                    """
                }
            ),
            PregelTask(
                id="2ca46996-dfc6-6de3-3662-cf1fe41bf987",
                name="node_joke",
                path=("__pregel_pull", "node_joke"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "joke": """
                    小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                    妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                    小猫追问：“那为什么狗只有一条命？”  
                    妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                    """
                }
            ),
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={},
        next=("__start__",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393d-6b9a-bfff-180e30c1d3bc"
            }
        },
        metadata={
            "source": "input",
            "step": -1,
            "parents": {}
        },
        created_at="2026-06-09T09:36:06.320009+00:00",
        parent_config=None,
        tasks=(
            PregelTask(
                id="3aa3bc58-3567-5832-d823-4373a9ab94e5",
                name="__start__",
                path=("__pregel_pull", "__start__"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "topic": "猫咪"
                }
            ),
        ),
        interrupts=()
    )
]
```



**<font style="color:rgb(60, 60, 67);"></font>**

#### 分析
##### StateSnapshot
`get_state_history()` 返回的是一个历史检查点迭代器。将其转换为列表后，可以看到一组 `StateSnapshot` 对象。

`StateSnapshot` 是检查点的开发者视图，字段含义如下

+ `values`：当前检查点的状态值
+ `next`：从该检查点继续执行时，下一步将要执行的节点，即归属于下一个超步的节点
+ `config`：当前检查点配置。常见结构如下：

```python
{
    "configurable": {
        "thread_id": "...",
        "checkpoint_ns": "...",
        "checkpoint_id": "..."
    }
}
```

    - `configurable`：用于记录检查点的可配置信息，可能包含用户自定义字段，它一定包含以下三个字段：
        * `thread_id`：会话唯一标识
        * `checkpoint_ns`：检查点命名空间。根图的命名空间通常为空字符串 `""`；子图会使用非空命名空间。
        * `checkpoint_id`：检查点唯一标识



这三个字段可以唯一标识一条检查点记录，在大多数检查点存储器实现中，写入检查点后端时都会将它们作为检查点的唯一键

+ `metadata`：检查点元数据，本节只需要关注 `step` 字段，后者是当前检查点对应的超步编号
+ `created_at`：检查点创建时间
+ `parent_config`：父检查点、即上一个检查点的配置
+ `tasks`：当前检查点关联的待执行任务信息，元素类型通常是 `PregelTask`。

`tasks` 通常和 `next` 对应，表示从当前检查点继续执行时，下一步将要运行的任务。

需要注意的是，`tasks` 中还可能包含这些任务已经成功（`result`）或失败（`error`）的任务记录，这是在下一个超步执行过程中记录的中间结果。在失败恢复场景中，可以避免重复执行已成功节点。

+ `interrupts`：当前图的中断信息，中断机制详见下文

**<font style="color:rgb(60, 60, 67);"></font>**


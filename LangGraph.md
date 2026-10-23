

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

```python
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

**<font style="color:#DF2A3F;"></font>**

## 持久化模式
### 概述
启用 `Checkpointer` 后，`LangGraph` 会在执行过程中保存检查点。

检查点写入越及时，**<font style="color:#DF2A3F;">系统在异常中断后可恢复的状态越完整，容灾能力越强</font>****；**但更及时的持久化通常也意味着更多中间写入，或额外的阻塞式等待，带来额外的性能开销和响应延迟（从调用者的角度考虑）。

`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 支持三种持久化模式，采用不同的检查点保存时机</font>**，在容灾能力和性能开销、响应时效性之间作取舍。



+ `**<font style="color:#DF2A3F;">exit</font>**`**<font style="color:#DF2A3F;">：退出模式。只在计算图正常结束、异常退出、被中断（如 </font>**`**<font style="color:#DF2A3F;">Human-In-The-Loop</font>**`**<font style="color:#DF2A3F;"> 中断）时保存检查点</font>**。不能处理中途进程崩溃的场景。这种模式响应性能开销最小，但容灾能力最弱。
+ `**<font style="color:#DF2A3F;">async</font>**`**<font style="color:#DF2A3F;">：异步模式。默认模式，顾名思义，检查点在后台异步写入</font>**。
    - 它会在每个超步结束后写入完整检查点（主检查点），并在图中任务执行完毕后记录中间结果。
    - 和 `exit` 模式相比，增加了性能开销，但写入操作发生在后台，不会引入明显的响应延迟，同时提升了容灾能力。
+ `sync`：同步模式。
    - 和异步模式唯一的区别在于，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 会在进入下一个超步之前等待当前主检查点的写入任务完成</font>**。它的容灾能力最强，但在 `async` 模式的基础上增加了响应延迟。

**<font style="color:rgb(60, 60, 67);">总结</font>**

| 模式 | 写入时机 | 性能开销 | 响应延迟 | 容灾能力 |
| --- | --- | --- | --- | --- |
| `exit` | **<font style="color:#DF2A3F;">运行退出时写入</font>** | 低 | 最低 | 最弱 |
| `async` | **<font style="color:#DF2A3F;">每个超步末尾写入主检查点，任务完成后写入中间结果，后台异步写入</font>** | 高 | 较高 | 较强 |
| `sync` | **<font style="color:#DF2A3F;">和 </font>**`**<font style="color:#DF2A3F;">async</font>**`<br/>**<font style="color:#DF2A3F;"> 区别在于，进入下一个超步之前等待当前主检查点的写入任务完成 ,这里是同步阻塞写入</font>** | 高 | 最高 | 最强 |






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
`LangGraph` 在配置检查点存储器后，**<font style="color:#DF2A3F;">会把同一个 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;"> 下的执行过程保存为一组检查点。</font>**通过这些检查点，我们可以查看图运行的中间状态，**<font style="color:#DF2A3F;">也可以为后续的检查点回溯（重放、分叉）和失败恢复做准备。</font>**

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

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

**<font style="color:rgb(60, 60, 67);"></font>**

##### 检查点主要信息说明
本例中检查点按照如下顺序返回

```python
Step 2 → Step 1 → Step 0 → Step -1
```

即

```python
最新检查点 → 最旧检查点
```

其中：

1. `step=-1`

这是输入检查点，`metadata["source"] == "input"`，这表示该检查点是在运行时初始化时被记录的，尚未进入图计算循环。

此时用户输入还没有真正变成普通业务节点可消费的状态，因此

```python
values={}
```

从这个检查点继续执行，下一步是内部启动节点 `__start__`，因此：

```python
next=("__start__",)
```

2. `step=0`

这是图计算循环的第一个检查点，`metadata["source"] == "loop"`，这表示检查点是在图计算循环中被写入的。

此时，`__start__` 已经把用户输入写入状态，所以 `values` 中已经可以看到状态信息，如下：

```python
values={
    "topic": "猫咪"
}
```

从这个检查点继续执行，`node_poem` 和 `node_joke` 将在下一个超步执行，因此：

```python
next=("node_poem", "node_joke")
```

3. `step=1`

这是图计算循环的第二个检查点，同样 `metadata["source"] == "loop"`。

同时，它对应第一个执行图节点计算逻辑的超步。

此时，`node_poem` 和 `node_joke` 的输出已经到状态中，因此：

```python
values={
    "topic": "猫咪",
    "poem": "...",
    "joke": "..."
}
```

从这个检查点开始，下游的汇总节点 `node_output` 满足执行条件，因此：

```python
next=("node_output",)
```

4. `step=2`

这是图计算循环的第三个检查点，同样 `metadata["source"] == "loop"`

此时，`node_output` 已经执行完成，最终输出也已经写入状态，因此：

```python
values={
    "topic": "猫咪",
    "poem": "...",
    "joke": "...",
    "final_output": "..."
}
```

此时，图已经执行结束，所以：

```python
next=()
```







##### `tasks`字段说明
`tasks` 是一个元组，其中的每个元素都是一个 `PregelTask` 实例

`tasks` 表示：如果从当前检查点继续执行，下一步要运行哪些任务。比如在 `step=0` 的检查点中：

```python
next=("node_poem", "node_joke")
tasks=(
    PregelTask(name="node_poem", ...),
    PregelTask(name="node_joke", ...),
)
```

这表示从该检查点继续执行时，下一步会运行 `node_poem` 和 `node_joke` 节点，底层执行的是它们的同名任务。

不过，查看历史检查点时，`PregelTask` 实例的 `result` 字段可能非空，如：

```python
PregelTask(
    name="node_poem",
    result={
        "poem": "..."
    }
)
```

这看起来像是“上一个检查点中提前保存了下一个超步的执行结果”。实际上：

+ **<font style="color:#DF2A3F;">完整的检查点（主检查点）是在超步末尾保存的</font>**
+ 但在一个超步内部，`LangGraph` 还会记录节点级别的计算结果
+ `checkpoint["id"]` 会在保存主检查点前由 `create_checkpoint()` 推进为新的 `ID`，然后用这个 `ID` 写入主检查点（假设当前超步为 `S1`）。下一轮（超步为 `S2`）计算循环中，任务执行完毕后的中间结果会用相同的 `ID` 写入。

因此，下一轮（超步为 `S2`）任务的计算结果绑定到了“当前超步（超步为 `S1`）的主检查点”。

+ 从检查点后端检索时，会按照 `checkpoint_id` 将主检查点和中间结果组织起来
+ 因此，检索到的历史检查点列表中，主检查点（超步为 `S1`）和下一个超步（超步为 `S2`）的中间结果会被放在同一个 `StateSnapshot` 中
+ 如果同一个超步里有多个并行任务，其中部分任务已经成功，另一个任务失败，那么成功任务的结果可以被保存下来；
+ 后续恢复时，`LangGraph` 可以复用这些已成功任务的结果，避免重复执行已经成功的节点。









### 查看最新检查点
如果只想查看当前线程的最新检查点，可以使用：

```python
latest_history_checkpoint = graph.get_state(config=config)
print(latest_history_checkpoint)
```

完整示例如下

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
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句，不要赏析，只给诗句即可")]).content

    return {
        "poem": poem
    }

def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话，尽可能简单，不超过一百字")]).content

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

print('=' * 30, '-> 最新历史检查点 <-', '=' * 30)
latest_history_checkpoint = graph.get_state(config=config)
print(latest_history_checkpoint)
```

运行结果如下

1. 运行日志

```python
2026-06-10 11:23:59.610 | INFO     | __main__:node_poem:33 - node_poem 已执行
2026-06-10 11:23:59.611 | INFO     | __main__:node_joke:42 - node_joke 已执行
2026-06-10 11:24:01.424 | INFO     | __main__:node_output:51 - node_output 已执行
```

2. 拓扑结构

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792143313676-2f0c40a9-6b83-4385-8e72-d86fab8697e8.png)







运行结果及检查点信息

```python
============================== -> 运行结果 <- ==============================
res: {} {'final_output': '关于 猫咪 的七言绝句：\n《戏题狸奴》\n昼眠锦褥夜巡檐，碧眼圆瞳映月纤。\n捕鼠本非真本领，得鱼时复近妆奁。\n笑话：\n小猫咪第一次照镜子，看到里面的自己，吓得炸毛：“你是谁？！” 它绕到镜子后面找了一圈，没找到，又回来对着镜子哈气。 最后它恍然大悟，对主人说：“我明白了，那是个隐藏的摄像头！”'}
============================== -> 最新历史检查点 <- ==============================
StateSnapshot(
    values={
        "topic": "猫咪",
        "poem": (
            "《戏题狸奴》\n"
            "昼眠锦褥夜巡檐，碧眼圆瞳映月纤。\n"
            "捕鼠本非真本领，得鱼时复近妆奁。"
        ),
        "joke": (
            "小猫咪第一次照镜子，看到里面的自己，吓得炸毛：“你是谁？！” "
            "它绕到镜子后面找了一圈，没找到，又回来对着镜子哈气。 "
            "最后它恍然大悟，对主人说：“我明白了，那是个隐藏的摄像头！”"
        ),
        "final_output": (
            "关于 猫咪 的七言绝句：\n"
            "《戏题狸奴》\n"
            "昼眠锦褥夜巡檐，碧眼圆瞳映月纤。\n"
            "捕鼠本非真本领，得鱼时复近妆奁。\n"
            "笑话：\n"
            "小猫咪第一次照镜子，看到里面的自己，吓得炸毛：“你是谁？！” "
            "它绕到镜子后面找了一圈，没找到，又回来对着镜子哈气。 "
            "最后它恍然大悟，对主人说：“我明白了，那是个隐藏的摄像头！”"
        ),
    },
    next=(),
    config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_ns": "",
            "checkpoint_id": "1f1647bd-3512-631a-8002-4e48b5ecda6d",
        }
    },
    metadata={
        "source": "loop",
        "step": 2,
        "parents": {},
    },
    created_at="2026-06-10T03:24:01.426084+00:00",
    parent_config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_ns": "",
            "checkpoint_id": "1f1647bd-350d-62c7-8001-34cd7fa012ed",
        }
    },
    tasks=(),
    interrupts=(),
)
```

`**<font style="color:#DF2A3F;">graph.get_state(config=config)</font>**`**<font style="color:#DF2A3F;"> 只返回当前会话的最新检查点。</font>**

在本例中，图已经执行完成，所以最新检查点具有以下特征：

```python
next=()
tasks=()
```

这表示没有后续节点需要继续执行。







### 根据ID查看指定检查点


如果希望查看某个历史检查点，可以在 **`configurable`** 中额外传入 **`checkpoint_id`**：

```python
target_config = {
    "configurable": {
        "thread_id": "123",
        "checkpoint_id": "某个历史 checkpoint_id"
    }
}

snapshot = graph.get_state(config=target_config)
print(snapshot)
```

这样返回的就不是最新检查点，而是指定 **`checkpoint_id`** 对应的历史检查点。





#### <font style="color:rgb(51, 51, 51);">获取历史检查点列表</font>
**<font style="color:rgb(51, 51, 51);">示例如下</font>**

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
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句，不要赏析，只给诗句即可")]).content

    return {
        "poem": poem
    }

def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话，尽可能简单，不超过一百字")]).content

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





**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

```python
2026-06-11 18:28:29.126 | INFO     | __main__:node_poem:33 - node_poem 已执行
2026-06-11 18:28:29.128 | INFO     | __main__:node_joke:42 - node_joke 已执行
2026-06-11 18:28:31.021 | INFO     | __main__:node_output:51 - node_output 已执行
```



<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792143446538-4f3b1f13-19c0-491b-a229-fd829a770717.png)

```python
============================== -> 运行结果 <- ==============================
res: {} {'final_output': '关于 猫咪 的七言绝句：\n《戏猫》\n狸奴日午卧青毡，蝶影翩然忽跃前。\n捕得飞花轻似梦，独摇银尾弄春烟。\n笑话：\n猫咪走进咖啡店，点了一杯牛奶。  \n店员问：“要不要加糖？”  \n猫咪摇头：“不用，我最近在减肥。”  \n店员看了看它圆滚滚的肚子：“真的吗？”  \n猫咪叹了口气：“唉，都是猫粮的错。”'}
============================== -> 历史检查点列表 <- ==============================
[
    StateSnapshot(
        values={
            "topic": "猫咪",
            "poem": (
                "《戏猫》\n"
                "狸奴日午卧青毡，蝶影翩然忽跃前。\n"
                "捕得飞花轻似梦，独摇银尾弄春烟。"
            ),
            "joke": (
                "猫咪走进咖啡店，点了一杯牛奶。  \n"
                "店员问：“要不要加糖？”  \n"
                "猫咪摇头：“不用，我最近在减肥。”  \n"
                "店员看了看它圆滚滚的肚子：“真的吗？”  \n"
                "猫咪叹了口气：“唉，都是猫粮的错。”"
            ),
            "final_output": (
                "关于 猫咪 的七言绝句：\n"
                "《戏猫》\n"
                "狸奴日午卧青毡，蝶影翩然忽跃前。\n"
                "捕得飞花轻似梦，独摇银尾弄春烟。\n"
                "笑话：\n"
                "猫咪走进咖啡店，点了一杯牛奶。  \n"
                "店员问：“要不要加糖？”  \n"
                "猫咪摇头：“不用，我最近在减肥。”  \n"
                "店员看了看它圆滚滚的肚子：“真的吗？”  \n"
                "猫咪叹了口气：“唉，都是猫粮的错。”"
            ),
        },
        next=(),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-acae-618c-8002-5a8b9e940205",
            }
        },
        metadata={
            "source": "loop",
            "step": 2,
            "parents": {},
        },
        created_at="2026-06-11T10:28:31.022523+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-acab-66b3-8001-d74912e4cb07",
            }
        },
        tasks=(),
        interrupts=(),
    ),

    StateSnapshot(
        values={
            "topic": "猫咪",
            "poem": (
                "《戏猫》\n"
                "狸奴日午卧青毡，蝶影翩然忽跃前。\n"
                "捕得飞花轻似梦，独摇银尾弄春烟。"
            ),
            "joke": (
                "猫咪走进咖啡店，点了一杯牛奶。  \n"
                "店员问：“要不要加糖？”  \n"
                "猫咪摇头：“不用，我最近在减肥。”  \n"
                "店员看了看它圆滚滚的肚子：“真的吗？”  \n"
                "猫咪叹了口气：“唉，都是猫粮的错。”"
            ),
        },
        next=("node_output",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-acab-66b3-8001-d74912e4cb07",
            }
        },
        metadata={
            "source": "loop",
            "step": 1,
            "parents": {},
        },
        created_at="2026-06-11T10:28:31.021422+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-9a92-67bc-8000-dd5ab888e9d5",
            }
        },
        tasks=(
            PregelTask(
                id="66dedbf9-3fbb-c539-5f4e-0e3f604a9c56",
                name="node_output",
                path=("__pregel_pull", "node_output"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "final_output": (
                        "关于 猫咪 的七言绝句：\n"
                        "《戏猫》\n"
                        "狸奴日午卧青毡，蝶影翩然忽跃前。\n"
                        "捕得飞花轻似梦，独摇银尾弄春烟。\n"
                        "笑话：\n"
                        "猫咪走进咖啡店，点了一杯牛奶。  \n"
                        "店员问：“要不要加糖？”  \n"
                        "猫咪摇头：“不用，我最近在减肥。”  \n"
                        "店员看了看它圆滚滚的肚子：“真的吗？”  \n"
                        "猫咪叹了口气：“唉，都是猫粮的错。”"
                    )
                },
            ),
        ),
        interrupts=(),
    ),

    StateSnapshot(
        values={
            "topic": "猫咪",
        },
        next=("node_poem", "node_joke"),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-9a92-67bc-8000-dd5ab888e9d5",
            }
        },
        metadata={
            "source": "loop",
            "step": 0,
            "parents": {},
        },
        created_at="2026-06-11T10:28:29.123773+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-9a90-6db5-bfff-d0eb857db9d8",
            }
        },
        tasks=(
            PregelTask(
                id="8b59cf5a-16b4-c734-41c8-56ac9b7614c9",
                name="node_poem",
                path=("__pregel_pull", "node_poem"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "poem": (
                        "《戏猫》\n"
                        "狸奴日午卧青毡，蝶影翩然忽跃前。\n"
                        "捕得飞花轻似梦，独摇银尾弄春烟。"
                    )
                },
            ),
            PregelTask(
                id="388c08f3-e341-3c76-cfc4-a9825ec97353",
                name="node_joke",
                path=("__pregel_pull", "node_joke"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "joke": (
                        "猫咪走进咖啡店，点了一杯牛奶。  \n"
                        "店员问：“要不要加糖？”  \n"
                        "猫咪摇头：“不用，我最近在减肥。”  \n"
                        "店员看了看它圆滚滚的肚子：“真的吗？”  \n"
                        "猫咪叹了口气：“唉，都是猫粮的错。”"
                    )
                },
            ),
        ),
        interrupts=(),
    ),

    StateSnapshot(
        values={},
        next=("__start__",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f165804-9a90-6db5-bfff-d0eb857db9d8",
            }
        },
        metadata={
            "source": "input",
            "step": -1,
            "parents": {},
        },
        created_at="2026-06-11T10:28:29.123104+00:00",
        parent_config=None,
        tasks=(
            PregelTask(
                id="63a24d9c-ea6d-0420-4b5a-b978c6eaeb90",
                name="__start__",
                path=("__pregel_pull", "__start__"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "topic": "猫咪",
                },
            ),
        ),
        interrupts=(),
    ),
]
```









#### 根据ID查看特定检查点
获取历史检查点列表后，可以从某个历史快照中取出它的 `checkpoint_id`，然后调用 `graph.get_state()` 查看该检查点。

```python
print('=' * 30, '-> 根据ID查看特定检查点 <-', '=' * 30)
checkpointe_id = history_checkpoints[3].config["configurable"]["checkpoint_id"]
print(f"checkpointe_id: {checkpointe_id}")

target_config = {
    "configurable": {
        "thread_id": config["configurable"]["thread_id"],
        "checkpoint_id": checkpointe_id
    }
}

specific_history_checkpoint = graph.get_state(config=target_config)
print(specific_history_checkpoint)
```

运行结果如下

```python
============================== -> 根据ID查看特定检查点 <- ==============================
checkpointe_id: 1f165804-9a90-6db5-bfff-d0eb857db9d8
StateSnapshot(
    values={},
    next=("__start__",),
    config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_id": "1f165804-9a90-6db5-bfff-d0eb857db9d8",
        }
    },
    metadata={
        "source": "input",
        "step": -1,
        "parents": {},
    },
    created_at="2026-06-11T10:28:29.123104+00:00",
    parent_config=None,
    tasks=(
        PregelTask(
            id="63a24d9c-ea6d-0420-4b5a-b978c6eaeb90",
            name="__start__",
            path=("__pregel_pull", "__start__"),
            error=None,
            interrupts=(),
            state=None,
            result={
                "topic": "猫咪",
            },
        ),
    ),
    interrupts=(),
)
```

当前场景下，我们已经获取了完整检查点列表，然后从中取了某个检查点的 `checkpoint_id`，然后调用 `get_state()` 查到了它对应的快照。

这看起来有些“绕”，因为我们本来就已经拿到了完整的历史快照。但在实际场景中，这个方法仍然很有意义。

例如，在后续的 检查点回溯-分叉 场景中，我们经常会基于历史检查点创建新的检查点，拿到该检查点的、带有 `checkpoint_id` 的配置信息 `config`。此时就可以将 `config` 传递给 `graph.get_state()`，从而便捷而精确地查看该检查点的快照。













## 使用场景
### 多轮对话
最基础的使用场景是多轮对话。前文示例已经实现了一个简单的多轮对话流程，这里不再展开。







### 中断恢复
中断恢复需要结合 `LangGraph` 的中断机制理解，见下文。

### 失败后恢复运行


#### 用法说明
任务失败的原因可能是偶发的外部因素，如网络波动、秘钥过期、第三方服务异常；也可能是程序自身的BUG。

如果是前者，故障发生后，修复外部问题，直接重试即可；如果是后者，需要更改代码修复BUG后，再基于历史检查点恢复运行。

要在失败后基于历史检查点恢复运行，需要满足以下条件：

1. **<font style="color:#DF2A3F;">启用检查点存储器</font>**
    - 如果是 `InMemorySaver`，不要重建检查点存储器对象，否则历史检查点丢失，无法恢复
    - 如果希望跨进程、服务重启后仍可恢复，应使用基于 `SQLite、Postgres` 等持久化检查点后端的存储器
2. **<font style="color:#DF2A3F;">再次运行时用 </font>**`**<font style="color:#DF2A3F;">None</font>**`**<font style="color:#DF2A3F;"> 作为计算图的状态输入</font>**
3. **<font style="color:#DF2A3F;">传递的配置信息应包含 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;"> 而不能包含 </font>**`**<font style="color:#DF2A3F;">checkpoint_id</font>**`

此时，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 会根据 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;"> 从 </font>**`**<font style="color:#DF2A3F;">checkpointer</font>**`**<font style="color:#DF2A3F;"> 读取该会话的最新检查点，并从该检查点继续推进。</font>**

如果某个超步中有多个并行任务，其中一部分任务成功，另一部分任务失败，那么已经成功完成的任务结果会作为 `pending_writes` 被保存。恢复运行时，已经成功完成的任务不会重复执行。





#### <font style="color:rgb(51, 51, 51);">示例</font>
##### <font style="color:rgb(51, 51, 51);">模拟失败</font>
<font style="color:rgb(51, 51, 51);">本例使用如下结构：</font>

```python
START
  -> node_change_topic
      -> node_poem
      -> node_joke
  -> node_output
  -> END
```

<font style="color:rgb(51, 51, 51);">运行时，</font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_change_topic</font>**`<font style="color:rgb(51, 51, 51);"> 会把输入主题 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">"猫咪"</font>**`<font style="color:rgb(51, 51, 51);"> 改写为 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">"猫咪: 布偶猫"</font>**`<font style="color:rgb(51, 51, 51);">，然后并行执行 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_poem</font>**`<font style="color:rgb(51, 51, 51);"> 和 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_joke</font>**`<font style="color:rgb(51, 51, 51);">，最后由 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_output</font>**`<font style="color:rgb(51, 51, 51);"> 汇总输出。</font>

**<font style="color:rgb(51, 51, 51);">示例如下</font>**

```python
from typing import TypedDict

from dotenv import  load_dotenv
from langchain_core.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph,START,END
from loguru import logger
load_dotenv(override=True)

model = ChatDeepSeek(
    model = "deepseek-v4-flash",
    extra_body={
        "thinking":{
            "type":"disabled"
        }
    }
)

#1. 声明状态
class OverAllState(TypedDict):
    topic:str
    poem:str
    joke:str
    final_output:str

#1.1 输入状态
class InputState(TypedDict):
    topic:str

#1.2 输出状态
class OutputState(TypedDict):
    final_output:str

topics = ["布偶猫","狸花猫","金渐层"]
topic_index = 0
#2. 定义节点
def node_change_topic(state:InputState)->OverAllState:
    global topic_index
    logger.info("topic_index:{}",topic_index)
    sub_topic = topics[topic_index]
    topic_index += 1
    topic_index %= len(topics)

    return {
        "topic":f"{state["topic"]}:{sub_topic}"
    }
#2.1 同一个超步的位置 成功运行的节点
def node_poem(state:OverAllState) -> OverAllState:
    logger.info("node_poem正在执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于{topic}主题的七言绝句")]).content
    return {
        "poem":poem
    }
import  time
#2.2 同一个超步的位置 运行失败的节点
def node_joke(state:OverAllState) -> OverAllState:
    logger.info("node_joke正在执行")
    topic = state["topic"]
    time.sleep(5)
    raise Exception("人为抛异常")
    joke = model.invoke([HumanMessage(f"写一首关于{topic}主题的笑话")]).content
    return {
        "joke":joke
    }

def node_output(state:OverAllState) -> OutputState:
    logger.info("node_output正在执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于{topic}的七言绝句:{poem}\n 笑话:{joke}\n"
    return {
        "final_output":final_output
    }

#3. 构建图
builder = StateGraph(state_schema=OverAllState,input_schema=InputState,output_schema=OutputState)

#3.1 添加节点
builder.add_node("node_change_topic",node_change_topic)
builder.add_node("node_poem",node_poem)
builder.add_node("node_joke",node_joke)
builder.add_node("node_output",node_output)

#3.2 添加边
builder.add_edge(START,"node_change_topic")
builder.add_edge("node_change_topic","node_poem")
builder.add_edge("node_change_topic","node_joke")
builder.add_edge("node_poem","node_output")
builder.add_edge("node_joke","node_output")
builder.add_edge("node_output",END)

#4. 添加检查点后端
DB_URL = "postgresql://langgraph_user:123456@localhost:5432/langgraph_db?sslmode=disable"
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    #5. 第一次使用PostgresSaver作为检查点 需要调用方法 setup()
    #checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    from IPython.display import display
    display(graph)

    config = {
        "configurable":{
            "thread_id":"chapter03-05"
        }
    }

    res = graph.invoke({"topic":"猫"},config=config)
    print(res)
```

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792143688316-bf9ece51-6d67-407d-a60c-98de23023aff.png)

```python
2026-06-10 16:17:43.544 | INFO     | __main__:node_change_topic:45 - topic_idx: 0
2026-06-10 16:17:43.545 | INFO     | __main__:node_joke:65 - node_joke 已执行
2026-06-10 16:17:43.548 | INFO     | __main__:node_poem:54 - node_poem 已执行
>Traceback...
Exception: 人为中断
During task with name 'node_joke' and id 'edb59019-4808-ed51-68eb-d5fba8696b13'
```

本例在 `node_joke` 中抛出异常，从而模拟计算图执行失败的场景。

为了观察恢复执行时哪些节点会被重新执行，示例中额外定义了全局变量 `topics` 和 `topic_idx`。`node_change_topic` 每执行一次，都会从 `topics` 中取出一个新的子主题，作为状态字段 `topic` 的值。

如果恢复运行时 `node_change_topic` 没有再次执行，则最终结果仍然会使用第一次生成的主题，即 `猫咪: 布偶猫`。如果该节点被重新执行，则主题可能会变成 `猫咪: 狸花猫` 或 `猫咪: 金渐层`。



##### 查看历史检查点列表
示例如下

list(graph.get_state_history(config=config))

运行结果如下

```python
[
    StateSnapshot(
        values={
            "topic": "猫咪: 布偶猫"
        },
        next=(
            "node_poem",
            "node_joke"
        ),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f164a86-e60d-6881-8001-c717582e8893"
            }
        },
        metadata={
            "source": "loop",
            "step": 1,
            "parents": {}
        },
        created_at="2026-06-10T08:43:19.431987+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f164a86-e608-6596-8000-ed45af11cea7"
            }
        },
        tasks=(
            PregelTask(
                id="66287e00-4228-a985-fc09-6791a14e26ce",
                name="node_poem",
                path=(
                    "__pregel_pull",
                    "node_poem"
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "poem": "《戏题布偶猫》\n雪翼云裘自绝尘，琉璃碧眼若含春。\n行来莲步生兰气，卧处梨花是此身。"
                }
            ),
            PregelTask(
                id="b2632b70-562b-7d5c-0dae-0f83bd6c2912",
                name="node_joke",
                path=(
                    "__pregel_pull",
                    "node_joke"
                ),
                error="Exception('人为中断')",
                interrupts=(),
                state=None,
                result=None
            )
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={
            "topic": "猫咪"
        },
        next=(
            "node_change_topic",
        ),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f164a86-e608-6596-8000-ed45af11cea7"
            }
        },
        metadata={
            "source": "loop",
            "step": 0,
            "parents": {}
        },
        created_at="2026-06-10T08:43:19.429860+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f164a86-e606-6a42-bfff-9f09d8a3aa96"
            }
        },
        tasks=(
            PregelTask(
                id="273fe47f-b4cc-93d0-0ab7-57a0db4bad56",
                name="node_change_topic",
                path=(
                    "__pregel_pull",
                    "node_change_topic"
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "topic": "猫咪: 布偶猫"
                }
            ),
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={},
        next=(
            "__start__",
        ),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f164a86-e606-6a42-bfff-9f09d8a3aa96"
            }
        },
        metadata={
            "source": "input",
            "step": -1,
            "parents": {}
        },
        created_at="2026-06-10T08:43:19.429167+00:00",
        parent_config=None,
        tasks=(
            PregelTask(
                id="ec8e6e32-6969-6798-9c5f-aade41e9211a",
                name="__start__",
                path=(
                    "__pregel_pull",
                    "__start__"
                ),
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

<font style="color:rgb(51, 51, 51);">可以看到，失败后的检查点中（</font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">step=1</font>**`<font style="color:rgb(51, 51, 51);">）记录了当前超步相关任务的执行情况：</font>

+ `**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">node_poem</font>**`**<font style="color:#DF2A3F;"> 执行成功，结果记录在 </font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">result</font>**`**<font style="color:#DF2A3F;"> 字段中。</font>**
+ `**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">node_joke</font>**`**<font style="color:#DF2A3F;"> 执行失败，</font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">result</font>**`**<font style="color:#DF2A3F;"> 为空，异常信息记录在 </font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">error</font>**`**<font style="color:#DF2A3F;"> 字段中。</font>**

<font style="color:rgb(51, 51, 51);">本例中，</font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_poem</font>**`<font style="color:rgb(51, 51, 51);"> 和 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_joke</font>**`<font style="color:rgb(51, 51, 51);"> 属于同一个超步。虽然 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_joke</font>**`<font style="color:rgb(51, 51, 51);"> 失败了，但 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_poem</font>**`<font style="color:rgb(51, 51, 51);"> 已成功，其计算结果会被持久化机制保存下来。因此，后续恢复运行时，</font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">node_poem</font>**`<font style="color:rgb(51, 51, 51);"> 不会重复执行。</font>





##### 修复BUG之后恢复运行
修复 `node_joke` 中的人为异常后，重新编译计算图，并基于最新检查点恢复运行。

示例如下

```python
def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]

    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话")]).content

    return {
        "joke": joke
    }


builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("node_change_topic", node_change_topic)
builder.add_node("node_poem", node_poem)
builder.add_node("node_joke", node_joke)
builder.add_node("node_output", node_output)
builder.add_edge(START, "node_change_topic")
builder.add_edge("node_change_topic", "node_poem")
builder.add_edge("node_change_topic", "node_joke")
builder.add_edge("node_poem", "node_output")
builder.add_edge("node_joke", "node_output")
builder.add_edge("node_output", END)

new_graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(new_graph)

res = new_graph.invoke(None, config=config)
print(res)
```

这里需要注意两点：

1. 重新编译计算图时，仍然使用原来的 `checkpointer` 对象。
2. 恢复运行时，输入参数传入 `None`，并且配置中只传入 `thread_id`。

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792143773780-3cb2ff3d-fe62-46bb-87a5-5aca663eb063.png)

```python
2026-06-10 16:37:12.985 | INFO     | __main__:node_joke:2 - node_joke 已执行
2026-06-10 16:37:18.878 | INFO     | __main__:node_output:76 - node_output 已执行
{
    'final_output': '关于 猫咪: 布偶猫 的七言绝句：\n《戏题布偶猫》\n雪翼云裘自绝尘，琉璃碧眼若含春。\n行来莲步生兰气，卧处梨花是此身。\n笑话：\n一只布偶猫走进咖啡馆，优雅地跳上吧台，对老板说：“给我来一杯最贵的猫屎咖啡。”  \n老板愣住：“可……您本身就是猫啊？”  \n布偶猫舔了舔爪子，翻了个白眼：“所以呢？我自己产的屎，你们人类不是喝得挺香吗？今天我倒要尝尝，你们拿我的‘周边产品’能泡出什么花样来。”'
}
```

运行结果中可以观察到：

+ `node_joke` 被重新执行。
+ `node_output` 被执行。
+ `node_poem` 没有被重新执行。

最终输出中的七言绝句和历史检查点中 `node_poem.result` 记录的内容一致，说明 `node_poem` 的执行结果被成功复用了。





### Time Travel
`Time Travel` 直译为 时间旅行，在当前场景下太过生硬，从技术实现和使用场景来看，译为<font style="background-color:#FBDE28;"> </font>**<font style="color:#DF2A3F;background-color:#FBDE28;">检查点回溯</font>** 更为合理。

所谓**<font style="color:#DF2A3F;">检查点回溯，是指基于某个历史检查点，重新执行后续流程，或者在该检查点基础上修改状态并创建新的执行分支。</font>**

检查点回溯有两种形式，根据是否更改历史状态区分：

+ `**<font style="color:#DF2A3F;background-color:#FBDE28;">Replay</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">：检查点重放，回到某个历史检查点，沿着原先的执行路径重新执行后续节点。</font>**
+ `**<font style="color:#DF2A3F;background-color:#FBDE28;">Fork</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">：检查点分叉，回到某个历史检查点，修改状态，从该位置创建一条新的执行分支。</font>**

二者的共同点是：

检查点之前的节点不会重新执行，检查点之后的节点会重新执行。

二者的区别是：

`Replay` 不修改历史状态；`Fork` 会基于历史检查点应用新的状态更新，并创建新的检查点分支。





#### Replay
`Replay` 模式和失败恢复很接近，但二者的触发方式和语义不同。

+ 失败恢复通常基于最新检查点继续运行，配置中只包含 `thread_id`，不包含 `checkpoint_id`。
+ `Replay` 则是显式传入某个历史检查点的配置，配置中包含 `checkpoint_id`。`LangGraph` 会据此从该历史检查点开始重放后续步骤。

需要注意：

`Replay` 不是简单读取历史缓存，**<font style="color:#DF2A3F;">而是重新执行该检查点之后的节点。</font>**

因此，检查点之后的 `LLM 调用、API 请求、工具调用、中断` 等都会重新触发，最终结果可能和原始结果不同。







##### 正常运行的任务
本例仍然使用如下结构：

```python
START
-> node_change_topic
-> node_poem
-> node_joke
-> node_output
-> END
```

示例如下

```python
from typing import TypedDict

from dotenv import  load_dotenv
from langchain_core.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph,START,END
from loguru import logger
load_dotenv(override=True)

model = ChatDeepSeek(
    model = "deepseek-v4-flash",
    extra_body={
        "thinking":{
            "type":"disabled"
        }
    }
)

#1. 声明状态
class OverAllState(TypedDict):
    topic:str
    poem:str
    joke:str
    final_output:str

#1.1 输入状态
class InputState(TypedDict):
    topic:str

#1.2 输出状态
class OutputState(TypedDict):
    final_output:str

topics = ["布偶猫","狸花猫","金渐层"]
topic_index = 0
#2. 定义节点
def node_change_topic(state:InputState)->OverAllState:
    global topic_index
    logger.info("topic_index:{}",topic_index)
    sub_topic = topics[topic_index]
    topic_index += 1
    topic_index %= len(topics)

    return {
        "topic":f"{state["topic"]}:{sub_topic}"
    }
#2.1 同一个超步的位置 成功运行的节点
def node_poem(state:OverAllState) -> OverAllState:
    logger.info("node_poem正在执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于{topic}主题的七言绝句")]).content
    return {
        "poem":poem
    }
import  time
#2.2 同一个超步的位置 运行失败的节点
def node_joke(state:OverAllState) -> OverAllState:
    logger.info("node_joke正在执行")
    topic = state["topic"]
    # time.sleep(5)
    # raise Exception("人为抛异常")
    joke = model.invoke([HumanMessage(f"写一首关于{topic}主题的笑话")]).content
    return {
        "joke":joke
    }

def node_output(state:OverAllState) -> OutputState:
    logger.info("node_output正在执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于{topic}的七言绝句:{poem}\n 笑话:{joke}\n"
    return {
        "final_output":final_output
    }

#3. 构建图
builder = StateGraph(state_schema=OverAllState,input_schema=InputState,output_schema=OutputState)

#3.1 添加节点
builder.add_node("node_change_topic",node_change_topic)
builder.add_node("node_poem",node_poem)
builder.add_node("node_joke",node_joke)
builder.add_node("node_output",node_output)

#3.2 添加边
builder.add_edge(START,"node_change_topic")
builder.add_edge("node_change_topic","node_poem")
builder.add_edge("node_change_topic","node_joke")
builder.add_edge("node_poem","node_output")
builder.add_edge("node_joke","node_output")
builder.add_edge("node_output",END)

#4. 添加检查点后端
DB_URL = "postgresql://langgraph_user:123456@localhost:5432/langgraph_db?sslmode=disable"
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    #5. 第一次使用PostgresSaver作为检查点 需要调用方法 setup()
    #checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    from IPython.display import display
    display(graph)

    config = {
        "configurable":{
            "thread_id":"chapter03-08"
        }
    }

    res = graph.invoke({"topic":"猫咪"},config=config)
    print(res)
```

**运行结果如下**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792143907272-a14d0ad5-c1a7-406f-b786-97256348965d.png)

```python
2026-06-12 14:32:42.019 | INFO     | __main__:node_change_topic:45 - topic_idx: 0
2026-06-12 14:32:42.022 | INFO     | __main__:node_joke:63 - node_joke 已执行
2026-06-12 14:32:42.026 | INFO     | __main__:node_poem:54 - node_poem 已执行
2026-06-12 14:32:43.585 | INFO     | __main__:node_output:72 - node_output 已执行
2026-06-12 14:32:43.588 | INFO     | __main__:<module>:102 - res: 
{
    'final_output': '关于 猫咪: 布偶猫 的七言绝句：\n《布偶猫》\n雪团云絮落仙踪，碧眼盈盈醉九重。\n静若琼瑶闲卧月，清姿一步一玲珑。\n笑话：\n一只布偶猫照镜子，惊呼：“我这么美，怎么还没当上国王？” 主人笑道：“因为你没有‘喵’民拥护。” 布偶猫委屈：“我有啊，铲屎官你不是天天跪着拥护我吗？”'
}
```

  
 





##### 查看历史检查点列表
示例如下

```python
history_checkpoints = list(graph.get_state_history(config=config))
print(history_checkpoints)
```

运行结果如下

```python
[
    StateSnapshot(
        values={
            'topic': '猫咪: 布偶猫',
            'poem': '《布偶猫》\n雪团云絮落仙踪，碧眼盈盈醉九重。\n静若琼瑶闲卧月，清姿一步一玲珑。',
            'joke': '一只布偶猫照镜子，惊呼：“我这么美，怎么还没当上国王？” 主人笑道：“因为你没有‘喵’民拥护。” 布偶猫委屈：“我有啊，铲屎官你不是天天跪着拥护我吗？”',
            'final_output': '关于 猫咪: 布偶猫 的七言绝句：\n《布偶猫》\n雪团云絮落仙踪，碧眼盈盈醉九重。\n静若琼瑶闲卧月，清姿一步一玲珑。\n笑话：\n一只布偶猫照镜子，惊呼：“我这么美，怎么还没当上国王？” 主人笑道：“因为你没有‘喵’民拥护。” 布偶猫委屈：“我有啊，铲屎官你不是天天跪着拥护我吗？”'
        },
        next=(),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-4ad3-67ef-8003-01d021de9403'
            }
        },
        metadata={
            'source': 'loop',
            'step': 3,
            'parents': {}
        },
        created_at='2026-06-12T06:32:43.586547+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-4acf-62cf-8002-aed204b5115d'
            }
        },
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'topic': '猫咪: 布偶猫',
            'poem': '《布偶猫》\n雪团云絮落仙踪，碧眼盈盈醉九重。\n静若琼瑶闲卧月，清姿一步一玲珑。',
            'joke': '一只布偶猫照镜子，惊呼：“我这么美，怎么还没当上国王？” 主人笑道：“因为你没有‘喵’民拥护。” 布偶猫委屈：“我有啊，铲屎官你不是天天跪着拥护我吗？”'
        },
        next=('node_output',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-4acf-62cf-8002-aed204b5115d'
            }
        },
        metadata={
            'source': 'loop',
            'step': 2,
            'parents': {}
        },
        created_at='2026-06-12T06:32:43.584774+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-3be4-6523-8001-dd86f05c34b8'
            }
        },
        tasks=(
            PregelTask(
                id='b9042203-0ddf-f638-aeeb-e0501601381a',
                name='node_output',
                path=('__pregel_pull', 'node_output'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'final_output': '关于 猫咪: 布偶猫 的七言绝句：\n《布偶猫》\n雪团云絮落仙踪，碧眼盈盈醉九重。\n静若琼瑶闲卧月，清姿一步一玲珑。\n笑话：\n一只布偶猫照镜子，惊呼：“我这么美，怎么还没当上国王？” 主人笑道：“因为你没有‘喵’民拥护。” 布偶猫委屈：“我有啊，铲屎官你不是天天跪着拥护我吗？”'
                }
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'topic': '猫咪: 布偶猫'
        },
        next=('node_poem', 'node_joke'),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-3be4-6523-8001-dd86f05c34b8'
            }
        },
        metadata={
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        created_at='2026-06-12T06:32:42.020569+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-3bde-6373-8000-9750b4be0665'
            }
        },
        tasks=(
            PregelTask(
                id='eab6bf51-4986-a60f-753c-e493b78c56b8',
                name='node_poem',
                path=('__pregel_pull', 'node_poem'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'poem': '《布偶猫》\n雪团云絮落仙踪，碧眼盈盈醉九重。\n静若琼瑶闲卧月，清姿一步一玲珑。'
                }
            ),
            PregelTask(
                id='a228a7bf-4c3b-2ad3-039f-12ac0107c87b',
                name='node_joke',
                path=('__pregel_pull', 'node_joke'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'joke': '一只布偶猫照镜子，惊呼：“我这么美，怎么还没当上国王？” 主人笑道：“因为你没有‘喵’民拥护。” 布偶猫委屈：“我有啊，铲屎官你不是天天跪着拥护我吗？”'
                }
            )
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'topic': '猫咪'
        },
        next=('node_change_topic',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-3bde-6373-8000-9750b4be0665'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-12T06:32:42.018069+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-3bdc-60a4-bfff-d4fdcf6e6a9e'
            }
        },
        tasks=(
            PregelTask(
                id='e0034c08-dd19-0ae9-7059-f8414745d212',
                name='node_change_topic',
                path=('__pregel_pull', 'node_change_topic'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'topic': '猫咪: 布偶猫'
                }
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={},
        next=('__start__',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f166288-3bdc-60a4-bfff-d4fdcf6e6a9e'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-12T06:32:42.017182+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='1a1ac312-152a-dcc3-2aa7-c2588f651448',
                name='__start__',
                path=('__pregel_pull', '__start__'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'topic': '猫咪'
                }
            ),
        ),
        interrupts=()
    )
]
```

##### 重放
`get_state_history()` 返回的是历史检查点列表，并且默认按照时间倒序排列。

`history_checkpoints` 结构如下：

```python
history_checkpoints[0] -> 最终完成后的检查点
next=()
history_checkpoints[1] -> node_output 执行前的检查点
next=('node_output',)
history_checkpoints[2] -> node_poem、node_joke 执行前的检查点
next=('node_poem', 'node_joke')
history_checkpoints[3] -> node_change_topic 执行前的检查点
next=('node_change_topic',)
history_checkpoints[4] -> __start__ 执行前的输入检查点
next=('__start__',)
```

因此，如果希望从写诗写笑话开始重放，选择 `next == ('node_poem', 'node_joke')` 的检查点即可。

示例如下

```python
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    #5. 第一次使用PostgresSaver作为检查点 需要调用方法 setup()
    #checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable":{
            "thread_id":"chapter03-08"
        }
    }
    # 获取到检查点历史
    history_checkpoints = list(graph.get_state_history(config=config))

    new_checkpoint = None
    #next = ('node_poem', 'node_joke')
    for checkpoint in history_checkpoints:
        if checkpoint.next == ('node_poem', 'node_joke'):
            new_checkpoint = checkpoint
            break

    # 如果想要实现replay的效果 状态填写None  config填写为之前某一个检查点的config
    res = graph.invoke(None,config=new_checkpoint.config)
    print(res)
```

运行结果如下

```python
2026-07-07 18:31:32.019 | INFO     | __main__:node_joke:59 - node_joke正在执行
2026-07-07 18:31:32.023 | INFO     | __main__:node_poem:50 - node_poem正在执行
2026-07-07 18:31:38.158 | INFO     | __main__:node_output:69 - node_output正在执行
{'final_output': '关于猫咪:狸花猫的七言绝句:《狸花猫》\n玄文斑驳隐花光，夜半巡檐气自昂。\n踏雪无痕轻似梦，一窗明月一炉香。\n\n注：我的创作思路是捕捉狸花猫的神秘与灵动。首句“玄文斑驳”描绘其毛发纹理，如暗夜中的花纹；次句“夜半巡檐”展现其夜行习性，气度不凡。后两句以“踏雪无痕”喻其轻盈，结句“明月炉香”营造温馨画面，暗合狸花猫守护家园的意象，使全诗动静相生，虚实相映。\n 笑话:# 狸花猫的烦恼\n\n有只狸花猫去宠物医院看病，医生问它：“哪里不舒服？”\n\n狸花猫叹了口气说：“医生，我总觉得自己不够‘花’。”\n\n医生疑惑：“你可是标准的狸花猫啊，条纹多漂亮！”\n\n狸花猫委屈地说：“可是隔壁那只三花猫，人家身上有黑、白、橘三种颜色，比我缤纷多了！我整天就这一身灰黑条纹，太单调了。”\n\n医生笑了：“那你想要怎么办？”\n\n狸花猫眼睛一亮：“医生，您能不能给我开点染色剂？我想在背上加几块橘色斑点，尾巴再来点白的，耳朵尖染成黑的…”\n\n医生打断它：“等等，那样你就变成‘四不像’了！”\n\n狸花猫思考了一会，突然骄傲地挺起胸膛：“你说得对！我这一身条纹可是老祖宗传下来的迷彩服，抓老鼠时特别管用！三花猫再好看，还得靠我帮忙抓家里的老鼠呢！”\n\n医生欣慰地说：“这就对了。”\n\n狸花猫走出诊室，正好遇到一只胖橘猫。胖橘猫嘲笑它：“哟，这不是那只会抓老鼠却不会卖萌的狸花猫吗？”\n\n狸花猫淡定地回了一句：“我会抓老鼠，你会什么？”\n\n胖橘猫想了想，尴尬地说：“我…我会吃。”\n\n狸花猫笑道：“那不就成了？你负责吃，我负责抓，咱们各有各的活法！”\n\n—— **适合自己的，才是最好的。** 🐱\n'}
```

本例中：

1. `node_poem` 和 `node_joke` 被重新执行。
2. `node_output` 被重新执行。

由于没有执行change_topic,所以可以一直写同一只猫的内容









#### Fork
##### 用法说明
`Fork` 依赖状态图的 `update_state()` 方法

示例如下

```python
change_input_config = graph.update_state(
    config=before_router_config.config,
    values={"user_input": "帮我写一个关于布偶猫的笑话"},
    as_node=START
)
```

其中：

+ `config`：历史检查点的配置信息，通常包含 `thread_id`、`checkpoint_ns` 和 `checkpoint_id`。`LangGraph` 会据此检索历史检查点
+ `values`：要应用到该检查点上的状态更新
+ `as_node`：指定这次状态更新应当被视为哪个节点产生的输出。



`LangGraph` 会按照该节点的写入逻辑，将 `values` 写入状态通道，并据此决定后续从哪些节点继续执行。

**<font style="color:#DF2A3F;">基于分叉检查点运行时，并不是重新执行 </font>**`**<font style="color:#DF2A3F;">as_node</font>**`**<font style="color:#DF2A3F;"> 本身，而是从它的后继节点继续推进</font>**

如 `START` 所属超步编号为 `0`，则恢复时从编号为 `1` 的超步开始运行。

注意

`update_state()` 不会修改原来的历史检查点，而是基于某个历史检查点创建一个新的检查点。这个新检查点相当于一条新的执行分支。













##### 准备检查点历史
###### 准备运行图并运行一次
示例如下

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langgraph.checkpoint.memory import InMemorySaver
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
    username: str
    user_input: str
    output: str


class StructuredOutputState(TypedDict):
    topic:Annotated[str,"主题"]
    mode:Annotated[Literal["poem","joke"],"模式"]



model_with_structure=model.with_structured_output(schema=StructuredOutputState)


def router_node(state: OverAllState) -> StructuredOutputState:
    logger.info("路由节点已执行")
    user_input = state["user_input"]
    res = model_with_structure.invoke(
        [
            HumanMessage(user_input)
        ]
    )
    logger.info("路由结果-res: {}", res)

    return res

def router(state: StructuredOutputState) -> Literal["node_poem", "node_joke", "node_default"]:
    logger.info("路由函数已执行")
    if state["mode"] == "poem":
        logger.info("路由至 node_poem 节点")
        return "node_poem"
    elif state["mode"] == "joke":
        logger.info("路由至 node_joke 节点")
        return "node_joke"

    logger.info("路由至兜底节点")
    return "node_default"

def node_poem(state: StructuredOutputState) -> OverAllState:
    logger.info(f"node_poem 已执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句，不要赏析，只给诗句即可")]).content

    return {
        "output": poem
    }

def node_joke(state: StructuredOutputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话，尽可能简单，不超过一百字")]).content

    return {
        "output": joke
    }

def node_default(state: StructuredOutputState) -> OverAllState:
    logger.info(f"node_default 已执行")

    return {
        "output": "无法处理的任务类型"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("router_node", router_node)
builder.add_node("node_poem", node_poem)
builder.add_node("node_joke", node_joke)
builder.add_node("node_default", node_default)

builder.add_edge(START, "router_node")
builder.add_conditional_edges(
    "router_node",
    router,
    path_map=["node_poem", "node_joke", "node_default"]
)
builder.add_edge("node_poem", END)
builder.add_edge("node_joke", END)
builder.add_edge("node_default", END)

checkpointer = InMemorySaver()
config = {"configurable": {"thread_id": "123"}}
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

res = graph.invoke({
    "username": "小黄",
    "user_input": "帮我写一首关于布偶猫的七言绝句"
}, config=config)
print(res)
```

执行逻辑如下：

1. `router_node` 根据用户输入提取结构化结果，例如 `{"topic": "布偶猫", "mode": "poem"}`。
2. `router` 根据 `mode` 决定后续分支。
3. 如果 `mode == "poem"`，进入 `node_poem`。
4. 如果 `mode == "joke"`，进入 `node_joke`。
5. 如果无法识别，则进入 `node_default`。



运行结果如下

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792144167235-9adf0dc2-623e-451a-8d4b-09aad126eaca.png)

```python
2026-06-11 14:26:48.353 | INFO     | __main__:router_node:31 - 路由节点已执行
2026-06-11 14:26:49.618 | INFO     | __main__:router_node:38 - 路由结果-res: {'topic': '布偶猫', 'mode': 'poem'}
2026-06-11 14:26:49.619 | INFO     | __main__:router:43 - 路由函数已执行
2026-06-11 14:26:49.619 | INFO     | __main__:router:45 - 路由至 node_poem 节点
2026-06-11 14:26:49.620 | INFO     | __main__:node_poem:55 - node_poem 已执行
{
    'username': '小黄', 
    'user_input': '帮我写一首关于布偶猫的七言绝句', 
    'output': '《布偶猫》\n蓝眼澄明玉作团，云踪雪影卧雕栏。\n无端一跃梅花落，犹带娇声唤月寒。'
}
```







###### 获取历史检查点列表
示例如下

```python
history_checkpoints = list(graph.get_state_history(config=config))
print(history_checkpoints)
```

运行结果如下

```python
[
    StateSnapshot(
        values={
            "username": "小黄",
            "user_input": "帮我写一首关于布偶猫的七言绝句",
            "output": (
                "《布偶猫》\n"
                "蓝眼澄明玉作团，云踪雪影卧雕栏。\n"
                "无端一跃梅花落，犹带娇声唤月寒。"
            ),
            "topic": "布偶猫",
            "mode": "poem",
        },
        next=(),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-8784-65e9-8002-e21663d55c94",
            }
        },
        metadata={
            "source": "loop",
            "step": 2,
            "parents": {},
        },
        created_at="2026-06-11T06:26:51.611071+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-7488-66c4-8001-f46b7c2d9a4e",
            }
        },
        tasks=(),
        interrupts=(),
    ),

    StateSnapshot(
        values={
            "username": "小黄",
            "user_input": "帮我写一首关于布偶猫的七言绝句",
            "topic": "布偶猫",
            "mode": "poem",
        },
        next=("node_poem",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-7488-66c4-8001-f46b7c2d9a4e",
            }
        },
        metadata={
            "source": "loop",
            "step": 1,
            "parents": {},
        },
        created_at="2026-06-11T06:26:49.620436+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-6871-6073-8000-ddcbe680da21",
            }
        },
        tasks=(
            PregelTask(
                id="cb3a8901-3a87-d0b0-3bab-d1adda5563d8",
                name="node_poem",
                path=("__pregel_pull", "node_poem"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "output": (
                        "《布偶猫》\n"
                        "蓝眼澄明玉作团，云踪雪影卧雕栏。\n"
                        "无端一跃梅花落，犹带娇声唤月寒。"
                    )
                },
            ),
        ),
        interrupts=(),
    ),

    StateSnapshot(
        values={
            "username": "小黄",
            "user_input": "帮我写一首关于布偶猫的七言绝句",
        },
        next=("router_node",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-6871-6073-8000-ddcbe680da21",
            }
        },
        metadata={
            "source": "loop",
            "step": 0,
            "parents": {},
        },
        created_at="2026-06-11T06:26:48.352564+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-686b-66c7-bfff-24e0e8d44f4c",
            }
        },
        tasks=(
            PregelTask(
                id="fb88436e-b163-6308-6341-7d8368e041f9",
                name="router_node",
                path=("__pregel_pull", "router_node"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "topic": "布偶猫",
                    "mode": "poem",
                },
            ),
        ),
        interrupts=(),
    ),

    StateSnapshot(
        values={},
        next=("__start__",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f1655e8-686b-66c7-bfff-24e0e8d44f4c",
            }
        },
        metadata={
            "source": "input",
            "step": -1,
            "parents": {},
        },
        created_at="2026-06-11T06:26:48.350273+00:00",
        parent_config=None,
        tasks=(
            PregelTask(
                id="a1b5e5fb-ee60-fd7b-68bc-475fa9cb66e2",
                name="__start__",
                path=("__pregel_pull", "__start__"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "username": "小黄",
                    "user_input": "帮我写一首关于布偶猫的七言绝句",
                },
            ),
        ),
        interrupts=(),
    ),
]
```

###### 获取 `router_node` 之前的检查点
示例如下

`router_node` 的上一个检查点的 `next` 字段为 `('router_node',)`，据此筛选检查点

```python
before_router_checkpoint = next(h for h in history_checkpoints if h.next == ('router_node',))
before_router_checkpoint
```

运行结果如下

```python
StateSnapshot(
    values={
        "username": "小黄",
        "user_input": "帮我写一首关于布偶猫的七言绝句",
    },
    next=("router_node",),
    config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_ns": "",
            "checkpoint_id": "1f1655e8-6871-6073-8000-ddcbe680da21",
        }
    },
    metadata={
        "source": "loop",
        "step": 0,
        "parents": {},
    },
    created_at="2026-06-11T06:26:48.352564+00:00",
    parent_config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_ns": "",
            "checkpoint_id": "1f1655e8-686b-66c7-bfff-24e0e8d44f4c",
        }
    },
    tasks=(
        PregelTask(
            id="fb88436e-b163-6308-6341-7d8368e041f9",
            name="router_node",
            path=("__pregel_pull", "router_node"),
            error=None,
            interrupts=(),
            state=None,
            result={
                "topic": "布偶猫",
                "mode": "poem",
            },
        ),
    ),
    interrupts=(),
)
```

该检查点表示：

用户输入已经写入状态，下一步将要执行 `router_node`。

从这个检查点分叉，本文采用两种方案（还可以有别的方案）：

1. 修改输入，让 `router_node` 重新执行。见 `6.5.4.2.3.` 节
2. 直接伪造 `router_node` 的输出，从而跳过 `router_node`。见 `6.5.4.2.4.` 节







##### 只改状态
###### 更新配置
示例如下

```python
change_input_config = graph.update_state(
    config=before_router_checkpoint.config,
    values={"user_input": "帮我写一个关于布偶猫的笑话"},
    as_node=START
)
change_input_config
```

这里把 `as_node` 设置为 `START`，表示：

把这次状态更新视为 `START` 的输出。

因此，后续运行时会从 `START` 的后继节点继续推进，也就是重新执行 `router_node`。

上述代码运行结果如下

```python
{
    "configurable": {
        "thread_id": "123",
        "checkpoint_ns": "",
        "checkpoint_id": "1f1655f8-86fb-6dd1-8001-9e7808dfaf1a",
    }
}
```

执行 `update_state()` 后会返回一个新的配置，其中包含新的 `checkpoint_id`。这个新的 `checkpoint_id` 不同于原有历史检查点，说明 `update_state()` 创建了一个新的检查点分支。

需要注意：

`update_state()` 返回的是新检查点的配置，不是新状态本身。

新的状态已经被记录在检查点后端，运行时生效。可以执行以下代码查看新建的检查点快照：

graph.get_state(change_input_config)

此处不再演示。









###### 重新运行
示例如下

```python
graph.invoke(None, change_input_config)
```



运行结果如下

```python
2026-06-11 14:13:44.267 | INFO     | __main__:router_node:31 - 路由节点已执行
2026-06-11 14:13:45.222 | INFO     | __main__:router_node:38 - 路由结果-res: {'topic': '布偶猫', 'mode': 'joke'}
2026-06-11 14:13:45.223 | INFO     | __main__:router:43 - 路由函数已执行
2026-06-11 14:13:45.223 | INFO     | __main__:router:48 - 路由至 node_joke 节点
2026-06-11 14:13:45.225 | INFO     | __main__:node_joke:64 - node_joke 已执行
{
    "username": "小黄",
    "user_input": "帮我写一个关于布偶猫的笑话",
    "output": (
        "一只布偶猫照镜子，问：“魔镜魔镜，谁是世界上最美的猫？”  \n"
        "镜子没说话。  \n"
        "猫急了：“快说呀！不然我卖萌把你甜碎。”  \n"
        "镜子终于开口：“……我死机了，重启中。”"
    ),
}
```

由运行结果可知：

1. `user_input` 已经从“帮我写一首关于布偶猫的七言绝句”变成“帮我写一个关于布偶猫的笑话”。而 `username` 没有被更新，仍然保持原值 `"小黄"`。说明 `values` 是 状态更新 而非完整的 状态替换。
2. `router_node` 开始的节点全部重新运行。由于新的输入被识别为笑话任务，因此后续进入 `node_joke`。









##### 改状态且跳过某些节点
###### 更新配置
如果不希望重新执行 `router_node`，而是直接指定 `router_node` 的输出，可以把 `as_node` 设置为 `"router_node"`。

示例如下

```python
skip_router_config = graph.update_state(
    config=before_router_checkpoint.config,
    values={"topic": "狸花猫", "mode": "笑话"},
    as_node="router_node"
)
skip_router_config
```

1. 此处把 `{"topic": "狸花猫", "mode": "笑话"}` 当作 `router_node` 的输出。
2. 因此，后续运行时，`router_node` 本身不会被重新执行，从它的后继节点继续推进。
3. 由于 `router_node` 挂载了条件边：

```python
builder.add_conditional_edges(
    "router_node",
    router,
    path_map=["node_poem", "node_joke", "node_default"]
)
```

所以当 `update_state()` 把 `values` 视为 `router_node` 的输出时，条件路由函数 `router` 也会根据这次更新后的状态计算后续分支。

这里把 `mode` 设置为 `"笑话"`，`router` 函数命中兜底分支 `node_default`。

运行结果如下

```python
2026-06-11 14:14:02.223 | INFO     | __main__:router:43 - 路由函数已执行
2026-06-11 14:14:02.223 | INFO     | __main__:router:51 - 路由至兜底节点
{
    "configurable": {
        "thread_id": "123",
        "checkpoint_ns": "",
        "checkpoint_id": "1f1655cb-de15-6c4e-8001-ca03b011e993",
    }
}
```

可以看到：

1. `update_state()` 创建了一个新的检查点，返回的 `skip_router_config` 指向这个新检查点。
2. 日志中出现了：

```python
路由函数已执行
路由至兜底节点
```

这说明 `router_node` 的节点函数没有被重新执行，但与 `router_node` 条件边相关的路由函数被执行了。

3. `update_state()` 底层会根据 `as_node` 找到对应图节点的 `writers`。

对于 `router_node`，`writers` 包括状态写入器和条件边对应的路由逻辑。

4. 执行 `update_state()` 时，传入的 `values` 会被当作 `router_node` 的输出，交给该节点的 `writers` 处理。

状态写入器会把 `values` 转换为对状态 `Channel` 的写入；条件边对应的路由逻辑会根据当前状态计算后继分支，并生成类似 `branch:to:node_default` 这样的触发通道写入。

随后，`update_state()` 会把这些写入应用到检查点中，并保存一个新的检查点。

因此，状态更新和路由结果已经记录在 `skip_router_config` 指向的新检查点中。后续再调用：

5. graph.invoke(None, skip_router_config)

时，运行图会基于这个新检查点继续执行，被触发的后继节点也会继续运行。

综上，`graph.update_state()` 的作用可以理解为：

不执行指定节点的节点函数本身，而是把传入的 `values` 当作该节点已经产生的输出，执行该节点对应的 `writers`，应用状态写入和可能存在的路由写入，然后生成一个新的检查点。

因此，在本例中可以看到：`router_node` 的节点逻辑没有被执行，但它挂载的路由函数被执行，状态更新也已经生效。









###### 重新运行
示例如下

```python
graph.invoke(None, skip_router_config)
```



运行结果如下

```python
2026-06-11 16:49:42.501 | INFO     | __main__:node_default:73 - node_default 已执行
{
    "username": "小黄",
    "user_input": "帮我写一首关于布偶猫的七言绝句",
    "output": "无法处理的任务类型"
}
```

和分析一致，触发了兜底分支。

这种方式常用于测试场景。例如：

+ 跳过不稳定的大模型路由节点。
+ 手动指定路由结果，测试不同分支。
+ 修改历史状态，验证后续节点能否正常运行。
+ 基于同一个历史检查点探索多条可能的执行路径。

不过在正式业务流程中，应谨慎使用这种方式。因为它是在伪造某个节点的输出，如果伪造的状态不符合后续节点的预期，可能导致逻辑混乱，甚至抛出运行时异常。











# <font style="color:rgb(51, 51, 51);">图记忆管理</font>
`Agent` 的三种记忆：

+ **<font style="color:#DF2A3F;">短期记忆：通过运行时状态 </font>**`**<font style="color:#DF2A3F;">State</font>**`**<font style="color:#DF2A3F;"> 访问，并由检查点存储器 </font>**`**<font style="color:#DF2A3F;">Checkpointer</font>**`**<font style="color:#DF2A3F;"> 保存</font>**，它按照 `thread_id` 组织，可以实现线程内的记忆共享。
+ **<font style="color:#DF2A3F;">长期记忆：通过长期记忆存储器 </font>**`**<font style="color:#DF2A3F;">Store</font>**`**<font style="color:#DF2A3F;"> 访问和存储。</font>**数据通常按照元组类型的命名空间组织，以键值对的形式存储，提供跨会话的记忆共享。
    - 可以按照命名空间+可选的过滤条件精确检索
    - 也可以通过语义模糊匹配。
+ **<font style="color:#DF2A3F;">运行时上下文：通过上下文对象 </font>**`**<font style="color:#DF2A3F;">Context</font>**`**<font style="color:#DF2A3F;"> 访问</font>**，只对本次调用生效，不会被持久化。

它更适合传递本次运行所需的外部依赖或调用参数，如用户名、模型配置、数据库连接、权限信息等。

我们不止一次提到，`**<font style="color:#DF2A3F;">Agent</font>**`**<font style="color:#DF2A3F;"> 底层就是一个简易的 </font>**`**<font style="color:#DF2A3F;">ReAct</font>**`**<font style="color:#DF2A3F;"> 架构的 </font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 状态图，所以 </font>**`**<font style="color:#DF2A3F;">Agent</font>**`**<font style="color:#DF2A3F;"> 的记忆机制本质上就是 </font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 状态图的记忆机制。</font>**

生产环境建议用基于持久化数据库的记忆存储器，如 `PostgresSaver` 和 `PostgresStore`。

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

## 短期记忆
只要编译图时启用了 `Checkpointer`，并且调用图时复用同一个 `thread_id`，`LangGraph` 就可以在多次调用之间共享同一会话线程下的状态。

我们一直在用，不在赘述。

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

## 长期记忆
长期记忆存储器在编译图时通过 `store` 参数传递，图节点中可以通过 `Runtime` 对象访问。

本节会在一个案例中同时用到短期记忆和长期记忆，两者均使用 `PostgreSQL` 作为持久化后端——短期记忆通过 `PostgresSaver`，长期记忆通过 `PostgresStore`，数据不会随进程退出而丢失。 



### 准备长期记忆
示例如下

```python
from typing import Final, Tuple
from langgraph.store.postgres import PostgresStore

# 与 6.2.4 节使用同一数据库
DB_URL = "postgresql://langgraph_user:123456@localhost:5432/langgraph_db?sslmode=disable"
with PostgresStore.from_conn_string(DB_URL) as store:
    # setup() 创建长期记忆所需的表结构，幂等操作
    store.setup()

    # 命名空间 - 层级结构：("users", "用户名")
    # Final 的作用是告诉静态类型检查器，该变量不应该被重新赋值，但 python 运行时不会阻止重新赋值
    USERS_NS: Final[Tuple[str]] = ("users", )
    PREFERENCES_KEY: Final[str] = "preferences"

    # 三层: (领域, 用户实体)
    # 每个 key 存储该用户的不同类型数据
    namespace1 = (*USERS_NS, "Alice")
    value1 = {
        "course": "计算机组成原理",
        "sports": "跑步",
        "food": "紫光园奶皮子酸奶"
    }

    namespace2 = (*USERS_NS, "Bob")
    value2 = {
        "course": "数字电路与模拟电路",
        "sports": "跑步",
        "food": "奶皮子糖葫芦"
    }

    namespace3 = (*USERS_NS, "Black")
    value3 = {
        "course": "数字电路与模拟电路",
        "sports": "羽毛球",
        "food": "紫光园奶皮子酸奶"
    }

    store.put(namespace1, PREFERENCES_KEY, value1)
    store.put(namespace2, PREFERENCES_KEY, value2)
    store.put(namespace3, PREFERENCES_KEY, value3)

    for item in store.search(USERS_NS):
        print(item)
```

运行结果如下

```python
Item(namespace=['users', 'Alice'], key='preferences', value={'course': '计算机组成原理', 'sports': '跑步', 'food': '紫光园奶皮子酸奶'}, created_at='2026-06-12T09:54:50.593300+00:00', updated_at='2026-06-12T09:54:50.593303+00:00', score=None)
Item(namespace=['users', 'Bob'], key='preferences', value={'course': '数字电路与模拟电路', 'sports': '跑步', 'food': '奶皮子糖葫芦'}, created_at='2026-06-12T09:54:50.593329+00:00', updated_at='2026-06-12T09:54:50.593329+00:00', score=None)
Item(namespace=['users', 'Black'], key='preferences', value={'course': '数字电路与模拟电路', 'sports': '羽毛球', 'food': '紫光园奶皮子酸奶'}, created_at='2026-06-12T09:54:50.593348+00:00', updated_at='2026-06-12T09:54:50.593348+00:00', score=None)
```

这里使用 `USERS_NS` 作为顶层命名空间，`store.search(USERS_NS)` 会查询所有以 `("users",)` 开头的记忆数据。返回的每条数据都是一个 `Item` 实例。

需要注意：

1. `namespace` 使用元组是硬性约束，天然表达层级结构。本例使用两层命名空间 `("users", "Alice")`，清晰区分了领域（`users`）和实体（用户名），而具体存储什么类型的数据则由 `key` 参数表达（如 `"preferences"`）。
2. `(*USERS_NS, "Alice")` 的写法表示对已有元组进行解包，再拼接新的命名空间片段。
3. `PostgresStore` 将长期记忆持久化到 `PostgreSQL` 中，程序重启后数据不会丢失，适合生产环境。本节使用和 6.2.4节、6.5.4.1节 相同的数据库实例。
4. 如果希望支持语义检索，需要在 `Store` 中配置索引和 `embedding` 函数。未配置索引时，`search()` 只能按照命名空间和过滤条件检索。

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

### 访问长期记忆
长期记忆存储器可以通过节点函数中的 `runtime.store` 访问。

示例如下

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore
from langgraph.graph.message import MessagesState
from langgraph.runtime import Runtime
from langchain.messages import SystemMessage, HumanMessage, AIMessage, ToolMessage
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

class OverAllState(MessagesState):
    username: str
    user_input: str
    output: str
    preferences: dict[str, str] # 用户偏好

def check_preferences_node(state: OverAllState, runtime: Runtime) -> OverAllState:
    # 从短期记忆中获取用户名
    username = state["username"]
    # 获取长期记忆存储器 store
    store = runtime.store

    # 根据用户信息拼接命名空间，查询用户偏好
    namespace = (*USERS_NS, username)
    key = PREFERENCES_KEY
    item = store.get(namespace, key)

    if not item:
        logger.warning("长期记忆中没有 {} 的偏好数据", username)
        return {}

    logger.info("长期记忆中保存的用户偏好: {}", item.value)
    return {
        "preferences": item.value
    }

# 路由函数，如果状态中没有用户偏好，则从长期记忆中查询，否则直接执行模型节点
def router(state: OverAllState) -> Literal["check_preferences_node", "llm_node"]:
    if not state.get("preferences"):
        logger.info("需要从长期记忆中查询用户偏好")
        return "check_preferences_node"
    logger.info("用户偏好已存在，不必查询")
    return "llm_node"

def llm_node(state: OverAllState) -> OverAllState:
    # 长期记忆中未必有用户偏好
    preferences = state.get("preferences", {})
    user_input = state["user_input"]
    human_prompt = (f"这是用户的偏好: \n{preferences}\n"
              f"这是用户的需求: \n{user_input}\n")
    system_prompt = "请根据用户偏好解决用户需求"

    # 获取历史消息列表，如果历史为空则初始化系统提示词
    messages: list[SystemMessage | HumanMessage | AIMessage | ToolMessage] = [
        SystemMessage(content=system_prompt)
    ] if not state.get("messages", []) else state["messages"]

    model_response = model.invoke(messages + [HumanMessage(content=human_prompt)])
    output = model_response.content

    return {
        "messages": messages + [HumanMessage(content=human_prompt), model_response],
        "output": output
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("check_preferences_node", check_preferences_node)
builder.add_node("llm_node", llm_node)
builder.add_conditional_edges(START, router, path_map=["check_preferences_node", "llm_node"])
builder.add_edge("check_preferences_node", "llm_node")
builder.add_edge("llm_node", END)

# DB_URL 与 6.2.4 节一致
DB_URL = "postgresql://langgraph_user:123456@localhost:5432/langgraph_db?sslmode=disable"
# 编译时传递短期记忆和长期记忆存储器（均使用 PostgreSQL）
with PostgresSaver.from_conn_string(DB_URL) as checkpointer, \
     PostgresStore.from_conn_string(DB_URL) as store:
    checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer, store=store)
    from IPython.display import display
    display(graph)

    config = {"configurable": {"thread_id": "345"}}
    res = graph.invoke({"username": "Alice", "user_input": "我有点无聊，和我聊聊天吧"}, config=config)

    print('=' * 30, '->  <-', '=' * 30)
    print(res)

    # 第二次调用：复用同一个 thread_id，观察短期记忆缓存效果
    new_res = graph.invoke({"user_input": "推荐一下酸奶"}, config=config)
    new_messages = new_res.pop("messages")
    print('=' * 30, '->  <-', '=' * 30)
    print(new_res)
```

运行结果如下

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792144637253-33f08534-2ab4-4bec-91b3-a60b82bcdbf6.png)

```python
2026-06-12 18:53:39.351 | INFO     | __main__:router:51 - 需要从长期记忆中查询用户偏好
2026-06-12 18:53:39.356 | INFO     | __main__:check_preferences_node:43 - 长期记忆中保存的用户偏好: {'course': '计算机组成原理', 'sports': '跑步', 'food': '紫光园奶皮子酸奶'}
============================== -> 不包含消息列表的运行结果 <- ==============================
{
    'username': 'Alice',
    'user_input': '我有点无聊，和我聊聊天吧', 
    'output': '哈哈，好啊！正好你学了《计算机组成原理》，我有个脑洞问题想请教一下：如果让你把“跑步的节奏感”比喻成计算机里的一个部件（比如CPU、内存、流水线），你会怎么对应呢？😄 顺便，我推荐你边慢跑边思考这个问题，跑完再来份紫光园奶皮子酸奶犒劳自己，简直完美！', 
    'preferences': {
        'course': '计算机组成原理', 
        'sports': '跑步', 
        'food': '紫光园奶皮子酸奶'
    }
}
============================== -> 消息列表 <- ==============================
================================ System Message ================================

请根据用户偏好解决用户需求
================================ Human Message =================================

这是用户的偏好: 
{
    'course': '计算机组成原理', 
    'sports': '跑步', 
    'food': '紫光园奶皮子酸奶'
}
这是用户的需求: 
我有点无聊，和我聊聊天吧

================================== Ai Message ==================================

哈哈，好啊！正好你学了《计算机组成原理》，我有个脑洞问题想请教一下：如果让你把“跑步的节奏感”比喻成计算机里的一个部件（比如CPU、内存、流水线），你会怎么对应呢？😄 顺便，我推荐你边慢跑边思考这个问题，跑完再来份紫光园奶皮子酸奶犒劳自己，简直完美！

```

运行流程如下：

1. 第一次调用时，状态中还没有 `preferences`。
2. 路由函数会路由至 `check_preferences_node`。
3. `check_preferences_node` 从长期记忆存储器中读取用户偏好。
4. 读取到的偏好会写入图状态，成为当前会话线程中的短期记忆。
5. 后续节点可以直接从状态中读取 `preferences`。









### 再次调用
在同一个 `with` 块中再次调用计算图，并复用同一个 `thread_id`，可以看到短期记忆的效果。该代码已包含在上节的 `with` 代码块末尾。

运行结果如下

```python
2026-06-12 18:54:12.446 | INFO     | __main__:router:53 - 用户偏好已存在，不必查询
    ============================== -> 不包含消息列表的运行结果 <- ==============================
    {
        'username': 'Alice', 
        'user_input': '推荐一下酸奶', 
        'output': '哈哈，看来你对紫光园奶皮子酸奶是真爱啊！不过既然你点名要推荐酸奶，我得先问一句——你是想换换口味试试别的，还是想让我帮你吹爆紫光园这款？（毕竟按你的偏好，它简直是“计算机组成原理”级别的完美搭配：**奶皮子像CPU缓存一样浓缩精华，酸甜度像流水线调度一样平衡，吃完跑步时还能提供“数据预取”般的能量**😂）\n\n如果是**忠于原味**，那必须再安利一次紫光园：浓稠拉丝、奶香爆炸，跑完步来一罐，简直像给肌肉做了个“中断响应”！如果是**想尝鲜**，我推荐：\n1. **兰格格天边的额吉**——蒙古风味，酸度低、像RISC精简指令集一样纯净；\n2. **简爱父爱配方**——无添加糖，适合跑步后控糖，像低功耗处理器；\n3. **卡士原态酪乳**——口感像固态硬盘般扎实绵密，适合当夜宵。\n\n不过……你确定要换掉紫光园吗？🤔', 
        'preferences': {
            'course': '计算机组成原理', 
            'sports': '跑步', 
            'food': '紫光园奶皮子酸奶'
        }
    }
    ============================== -> 消息列表 <- ==============================
    ================================ System Message ================================

    请根据用户偏好解决用户需求
    ================================ Human Message =================================

    这是用户的偏好: 
    {
        'course': '计算机组成原理', 
        'sports': '跑步', 
        'food': '紫光园奶皮子酸奶'
    }
    这是用户的需求: 
    我有点无聊，和我聊聊天吧

    ================================== Ai Message ==================================

    哈哈，好啊！正好你学了《计算机组成原理》，我有个脑洞问题想请教一下：如果让你把“跑步的节奏感”比喻成计算机里的一个部件（比如CPU、内存、流水线），你会怎么对应呢？😄 顺便，我推荐你边慢跑边思考这个问题，跑完再来份紫光园奶皮子酸奶犒劳自己，简直完美！
    ================================ Human Message =================================

    这是用户的偏好: 
    {
        'course': '计算机组成原理', 
        'sports': '跑步', 
        'food': '紫光园奶皮子酸奶'
    }
    这是用户的需求: 
    推荐一下酸奶

    ================================== Ai Message ==================================

    哈哈，看来你对紫光园奶皮子酸奶是真爱啊！不过既然你点名要推荐酸奶，我得先问一句——你是想换换口味试试别的，还是想让我帮你吹爆紫光园这款？（毕竟按你的偏好，它简直是“计算机组成原理”级别的完美搭配：**奶皮子像CPU缓存一样浓缩精华，酸甜度像流水线调度一样平衡，吃完跑步时还能提供“数据预取”般的能量**😂）

    如果是**忠于原味**，那必须再安利一次紫光园：浓稠拉丝、奶香爆炸，跑完步来一罐，简直像给肌肉做了个“中断响应”！如果是**想尝鲜**，我推荐：
        1. **兰格格天边的额吉**——蒙古风味，酸度低、像RISC精简指令集一样纯净；
            2. **简爱父爱配方**——无添加糖，适合跑步后控糖，像低功耗处理器；
                3. **卡士原态酪乳**——口感像固态硬盘般扎实绵密，适合当夜宵。

    不过……你确定要换掉紫光园吗？🤔
```

第二次运行前，状态中已经包含：

+ `username`
+ `preferences`
+ `messages`
+ 上一次调用生成的 `output`

因此路由函数会直接判断：

logger.info("用户偏好已存在，不必查询")

因此，本次调用不会重新访问长期记忆存储器，而是直接复用当前会话状态中缓存的用户偏好。

这正是短期记忆和长期记忆协作的典型模式：

第一次调用从长期记忆中读取数据，并写入短期记忆；后续同一会话线程内优先复用短期记忆，避免重复查询长期记忆。

<font style="color:rgb(119, 119, 119);"></font>



## 运行时上下文
在某些场景下，我**<font style="color:#DF2A3F;background-color:#FBDE28;">们希望在调用图时传入一些仅对当次调用有效的信息</font>**，比如当前登录用户、请求来源、调用方标识等。这些信息不适合放入图状态（图状态会被持久化并在同一会话中跨调用共享），而应该通过运行时上下文（Runtime Context）传递。

**<font style="color:#DF2A3F;">运行时上下文的特点是：</font>**

**<font style="color:#DF2A3F;">仅对本次调用生效，不会被持久化，也不会在同一会话的下一次调用中自动恢复。</font>**

运行时上下文的使用方式：

1. **<font style="color:#DF2A3F;">初始化状态图时，使用 </font>**`**<font style="color:#DF2A3F;">context_schema</font>**`**<font style="color:#DF2A3F;"> 定义上下文类型。</font>**
2. **<font style="color:#DF2A3F;">调用图时通过 </font>**`**<font style="color:#DF2A3F;">context</font>**`**<font style="color:#DF2A3F;"> 参数传入上下文对象。</font>**
3. **<font style="color:#DF2A3F;">节点或路由函数中通过 </font>**`**<font style="color:#DF2A3F;">runtime.context</font>**`**<font style="color:#DF2A3F;"> 访问上下文。</font>**

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

### <font style="color:rgb(51, 51, 51);">访问运行时上下文</font>
<font style="color:rgb(51, 51, 51);">下面通过一个简单的客服机器人案例演示运行时上下文的用法。</font>

**<font style="color:rgb(51, 51, 51);">场景</font>**<font style="color:rgb(51, 51, 51);">：不同的用户调用同一个机器人，机器人根据运行时上下文中的</font>**<font style="color:rgb(51, 51, 51);">用户名</font>**<font style="color:rgb(51, 51, 51);">和</font>**<font style="color:rgb(51, 51, 51);">会员等级</font>**<font style="color:rgb(51, 51, 51);">，生成不同风格的回复。</font>

图结构：

```python
START -> llm_node -> END
```



示例如下

```python
from dataclasses import dataclass
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.runtime import Runtime
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

# 1. 定义运行时上下文类型
@dataclass
class UserContext:
    username: str
    membership_level: str  # "普通用户" / "VIP"

# 2. 定义图状态
class OverAllState(MessagesState):
    user_input: str
    output: str

# 3. 定义节点：通过 runtime.context 访问运行时上下文
def llm_node(state: OverAllState, runtime: Runtime[UserContext]) -> OverAllState:
    runtime_context = runtime.context

    if runtime_context:
        username = runtime_context.username
        level = runtime_context.membership_level
        logger.info(f"当前用户: {username}, 会员等级: {level}")

        if level == "VIP":
            system_prompt = f"你是高级客服助理。当前VIP用户是{username}，请使用尊称'您'，语气热情周到，回复末尾加上'🎖️VIP专属服务'。"
        else:
            system_prompt = f"你是普通客服助理。当前用户是{username}，请友好简洁地回复。"
    else:
        logger.warning("运行时上下文为空，使用默认风格")
        system_prompt = "你是客服助理，请友好简洁地回复。"

    user_input = state["user_input"]
    messages = state.get("messages", [])
    response = model.invoke(
        [HumanMessage(content=system_prompt)] +
        messages +
        [HumanMessage(content=user_input)]
    )

    return {
        "messages": [response],
        "output": response.content
    }

# 4. 构建图，传入 context_schema
builder = StateGraph(state_schema=OverAllState, context_schema=UserContext)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", END)

graph = builder.compile()

# === 第一次调用：传入运行时上下文（VIP用户） ===
print("=" * 30, "第一次调用：VIP用户", "=" * 30)
config = {"configurable": {"thread_id": "demo-7.3"}}
res = graph.invoke(
    {"user_input": "你好，帮我查一下最近有什么优惠活动"},
    config=config,
    context=UserContext(username="Alice", membership_level="VIP")
)

print(f"output: {res['output']}")
print()

# === 第二次调用：不传运行时上下文 ===
print("=" * 30, "第二次调用：不传上下文", "=" * 30)
res2 = graph.invoke(
    {"user_input": "再帮我看看有没有新品"},
    config=config
)

print(f"output: {res2['output']}")

from IPython.display import display
display(graph)
```

运行结果如下

```python
2026-07-09 17:40:27.208 | INFO     | __main__:llm_node:39 - 当前用户: Alice, 会员等级: VIP
    ============================== 第一次调用：VIP用户 ==============================
    2026-07-09 17:40:29.116 | WARNING  | __main__:llm_node:46 - 运行时上下文为空，使用默认风格
        output: 尊敬的Alice您好！非常感谢您的咨询🎉 我们近期为VIP用户准备了多项专属优惠活动，包括但不限于：1）满额立减（消费满888元立减88元）；2）积分翻倍（指定商品享3倍积分）；3）限时闪购（每日10:00-12:00精选好物低至5折）。如需了解具体参与商品或为您定制个性化推荐，请随时告知，我将全力为您服务！🎖️VIP专属服务

    ============================== 第二次调用：不传上下文 ==============================
    output: 您好！目前新品信息尚未更新，建议您关注官方渠道获取最新动态。如有其他问题，欢迎随时咨询，我会尽力为您解答！😊
```

分析

1. 第一次调用传入了 `context=UserContext(username=”Alice”, membership_level=”VIP”)`，节点通过 `runtime.context` 读取到用户名和会员等级，生成了VIP风格的回复，末尾带上了 `🎖️VIP专属服务` 标识。
2. 第二次调用使用了相同的 `thread_id`，但没有传入 `context`。此时 `runtime.context` 为 `None`，节点走入了默认风格分支，回复中不再有VIP标识。

这个对比可以清晰地看到：

运行时上下文仅对当次调用生效，不会被持久化。即使使用相同的 `thread_id`，下一次调用也不会自动继承上一次的上下文。

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

### 小结
运行时上下文适合传递以下信息：

| **<font style="color:#DF2A3F;">适合放入运行时上下文</font>** | 不适合（应放入图状态） |
| --- | --- |
| **<font style="color:#DF2A3F;">当前登录用户信息</font>** | 需要在多轮对话间共享的数据 |
| **<font style="color:#DF2A3F;">请求来源（Web / API / 小程序）</font>** | 需要在检查点中恢复的执行进度 |
| **<font style="color:#DF2A3F;">调用方标识、Trace ID</font>** | 需要跨调用持久化的业务数据 |
| **<font style="color:#DF2A3F;">本次调用的功能开关</font>** | 节点间需要传递的计算结果 |


简单来说：

> **<font style="color:#DF2A3F;">跨调用共享、需要持久化的数据 → 放入 State；仅当次调用有效的信息 → 放入 Runtime Context。</font>**
>

## Node总结：LangGraph的节点
### 节点的本质
从用户视角看，`LangGraph` 的节点通常是一个可调用对象，最常见的是同步或异步 `Python` 函数：

```python
def sync_node(state: State) -> State:
    ...

async def async_node(state: State) -> State:
    ...
```

除此之外，节点也可以是符合要求的其他可调用对象或 `**<font style="color:#DF2A3F;">Runnable</font>**`**<font style="color:#DF2A3F;"> 实例。</font>**

从源码实现角度看，节点最终会被转换或包装为可运行对象，**<font style="color:#DF2A3F;">并作为 </font>**`**<font style="color:#DF2A3F;">PregelNode.bound</font>**`**<font style="color:#DF2A3F;"> 保存</font>**





### 节点函数的完整形态
状态图节点的第一个位置参数用于接收图状态。

此外，`LangGraph` 还可以在运行时注入运行时配置、运行时对象和流式写入器等参数。

普通图节点通常使用以下四个参数：

+ `**<font style="color:#DF2A3F;">state</font>**`**<font style="color:#DF2A3F;">：输入节点的图状态。</font>**`**<font style="color:#DF2A3F;">state</font>**`**<font style="color:#DF2A3F;"> 是位置传参，因此参数名称不重要，但通常约定命名为 </font>**`**<font style="color:#DF2A3F;">state</font>**`
+ `**<font style="color:#DF2A3F;">config</font>**`**<font style="color:#DF2A3F;">：状态图的运行时配置，它是一个 </font>**`**<font style="color:#DF2A3F;">RunnableConfig</font>**`**<font style="color:#DF2A3F;"> 实例，底层运行时在节点函数执行前通过关键字传参动态注入。</font>**

可以通过 `config`**<font style="color:#DF2A3F;"> 访问当前线程的 </font>**`**<font style="color:#DF2A3F;">thread_id</font>**`**<font style="color:#DF2A3F;">、超步序号、递归限制等配置信息。</font>**如下

```python
config["configurable"]["thread_id"] 
config["recursion_limit"]
config["metadata"]
```

+ `runtime`：状态图的运行时对象，可以通过它访问运行时上下文、长期记忆存储器等信息。底层运行时在节点函数执行前通过关键字传参动态注入。
+ `writer`：流式写入器，**<font style="color:#DF2A3F;">通常用于自定义流式输出</font>**。详见流处理章节。





典型写法如下： 

```python
def node(state: State, config: RunnableConfig, runtime: Runtime[Context]) -> State:
    ...
```

+ 编译图时传入 `checkpointer`，**<font style="color:#DF2A3F;">并在调用时传入 </font>**`**<font style="color:#DF2A3F;">config</font>**`，可以在节点中通过 `config` 间接访问当前调用的配置信息，如 `thread_id`。
+ 编译图时传入 `store` 后，**<font style="color:#DF2A3F;">可以在节点中通过 </font>**`**<font style="color:#DF2A3F;">runtime.store</font>**`**<font style="color:#DF2A3F;"> 访问长期记忆存储器。</font>**
+ 初始化状态图时传入 `context_schema`，并在调用时传入 `context` 后，可以在节点中通过 `**<font style="color:#DF2A3F;">runtime.context</font>**`**<font style="color:#DF2A3F;"> 访问运行时上下文。</font>**









### 节点的触发和执行
#### 节点的触发
##### 用户视角
从用户层面看，节点由图中的控制流关系（普通边、条件边和动态控制指令）触发：

+ 普通边决定**<font style="color:#DF2A3F;">固定的后继节点。</font>**
+ 条件边根据路由函数的返回值决定后继节点。
    - **<font style="color:#DF2A3F;">可以返回普通节点名称</font>**
    - **<font style="color:#DF2A3F;">也可以返回 </font>**`**<font style="color:#DF2A3F;">Send</font>**`**<font style="color:#DF2A3F;"> 实例，从而动态派发多个并行任务</font>**，常用于 `Map-Reduce` 场景。
+ `**<font style="color:#DF2A3F;">Command.goto</font>**`**<font style="color:#DF2A3F;"> 可以在节点返回时动态指定后继节点</font>**。





##### 源码视角
从源码实现角度看，`LangGraph` 会将图状态和节点之间的触发关系组织为通道。

其中，状态字段通常对应状态通道，节点之间的控制流关系则通过**<font style="color:#DF2A3F;">触发通道或屏障通道表示</font>**。对于普通的单节点触发关系，目标节点通常会订阅如下形式的内部通道：

```python
branch:to:<node_name>
```



例如，对于普通边：

```python
builder.add_edge("node_a", "node_b")
```

运行时 `node_a` 的写入器会向以下通道写入数据：

```python
branch:to:node_b
```

而 `node_b` 会将该通道注册为自己的触发通道。

条件边和 `Command.goto` 最终也会根据目标节点生成相应的触发通道写入。`Send` 则会生成动态派发任务所需的 `Send` 数据包。

对于多个前驱节点共同汇聚到同一个节点的情况，底层还可能使用类似下面的屏障通道：

```python
join:<node_a>+<node_b>:<node_c>
```

用于等待指定的前驱节点全部完成。

当某个节点订阅的触发通道在当前超步中产生新的写入后，该节点会在下一超步被调度。





#### 节点的执行
##### 用户视角
从用户层面看，节点执行过程可以理解为：

1. `LangGraph` 从当前图状态中读取节点需要的字段，并构造节点输入。
2. 调用节点函数，执行其业务逻辑。
3. 节点函数返回状态更新、`Command` 或其他受支持的结果。
4. `LangGraph` 将节点返回值转换为状态通道和控制通道的写入条目。
5. **<font style="color:#DF2A3F;">如果当前节点挂载了条件边，则执行条件边路由逻辑</font>**，根据包含当前节点更新的状态决定后继节点。
6. **<font style="color:#DF2A3F;">当前超步的所有任务执行完毕后，</font>**`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 汇总这些写入</font>**，并根据各状态字段的 `Reducer` 规则更新图状态。

节点通常只需要返回发生变化的字段，而不需要返回完整状态。例如：

```python
def node_a(state: State) -> State:
    return {"count": state["count"] + 1}
```

其中：

{"count": state["count"] + 1}

表示对状态的更新，而不是一份必须包含所有字段的完整状态。







##### 源码视角
从源码实现角度看，节点业务逻辑、状态写入逻辑和控制流写入逻辑会被组合成一个可顺序执行的可运行对象。

编译节点时，`LangGraph` 会创建一个 `PregelNode`。

相关源码位于 `CompiledStateGraph` 类的 `attach_node()` 方法中，如下

```python
self.nodes[key] = PregelNode(
    triggers=[branch_channel],
    # read state keys and managed values
    channels=("__root__" if is_single_input else input_channels),
    # coerce state dict to schema class (eg. pydantic model)
    mapper=mapper,
    # publish to state keys
    writers=[ChannelWrite(write_entries)],
    metadata=node.metadata,
    retry_policy=node.retry_policy,
    cache_policy=node.cache_policy,
    bound=node.runnable,  # type: ignore[arg-type]
)
```

其中：

+ `bound` 保存节点的业务逻辑；
+ `writers` 保存节点执行完成后需要调用的写入器；
+ `triggers` 保存能够触发该节点的通道。

可以近似表示为：

```python
PregelNode
├── bound：节点业务逻辑
├── writers：状态和控制流写入器
└── triggers：能够触发该节点的通道
```

因此，从整体上看，节点执行并不是简单地”调用一个函数并返回结果”，而是：

执行业务逻辑，生成状态和控制流写入，并通过目标节点触发通道或动态任务写入，为下一超步生成待调度任务，持续推动计算图运行。





# 中断
`LangGraph` 提供了两种中断机制：

+ **<font style="color:#DF2A3F;">动态中断：在图的任意节点中调用 </font>**`**<font style="color:#DF2A3F;">interrupt()</font>**`**<font style="color:#DF2A3F;"> 函数实现</font>**它可以放在代码的任意位置，并且可以根据应用逻辑设置条件触发，所以是动态的。动态中断提供了人机交互接口，使得调用者可以人为干预计算图的运行，是业务逻辑的一部分。
+ **<font style="color:#DF2A3F;">静态中断：在编译或调用状态图时通过 </font>**`**<font style="color:#DF2A3F;">interrupt_before</font>**`**<font style="color:#DF2A3F;"> 和 </font>**`**<font style="color:#DF2A3F;">interrupt_after</font>**`**<font style="color:#DF2A3F;"> 参数设置断点</font>**它是在运行前确定的，不能根据业务逻辑条件触发，所以是静态的。静态中断主要用于调试，不是业务逻辑的一部分。





## 动态中断
### 概述
+ `**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 的动态中断机制允许用户在图执行过程中设置暂停点</font>**<font style="color:#DF2A3F;">，</font>等待外部输入后再继续执行，提供了人机交互接口。
+ 当中断触发时，`**<font style="color:#DF2A3F;">LangGraph</font>**`**<font style="color:#DF2A3F;"> 会通过持久化机制保存当前图状态，并无限期等待直到用户恢复执行。</font>**
+ 中断通过在任意图节点中调用 `interrupt()` 函数实现，该函数可以接收任何可 `JSON` 序列化的值，并将该值暴露给调用方。
+ 中断触发后，调用方根据 `interrupt()` 暴露的信息生成反馈，重新调用图时通过 `Command` 将反馈传递给计算图，恢复图的运行。
+ 恢复运行后，调用方通过 `Command` 传递的反馈会作为 `interrupt()` 函数的返回值，参与后续计算。













### 启用中断
1. 配置检查点存储器
2. 设置 `thread_id`：如果要确保中断后可以恢复执行，就需要保存运行图的完整状态，**<font style="color:#DF2A3F;">所以必须启用可恢复执行。</font>**
3. **<font style="color:#DF2A3F;">在需要中断的位置调用 </font>**`**<font style="color:#DF2A3F;">interrupt()</font>**`











### 恢复中断
满足以下要求：

+ **<font style="color:#DF2A3F;">基于相同的配置再次调用计算图</font>**
+ **<font style="color:#DF2A3F;">将输入替换为 </font>**`**<font style="color:#DF2A3F;">Command()</font>**`**<font style="color:#DF2A3F;"> 实例即可恢复运行。</font>**

`LangGraph` **<font style="color:#DF2A3F;">会从中断节点继续运行，该节点会被再次执行</font>**。

通过 `Command` 实例的 `resume` 属性将用户反馈传递给计算图，中断节点重新运行时，传递给 `resume` 属性的值将会作为 `interrupt()` 函数的返回值。









### 常见使用模式
根据中断发生的位置、业务场景以及执行结构，动态中断可以衍生出多种常见的使用模式。本节将介绍以下模式：

1. **<font style="color:#DF2A3F;">基础 HITL 模式：状态图触发一次中断，获取人类输入后继续执行。</font>**
2. 多个并行中断：多个并行任务分别产生中断，并根据 中断 `ID` 接收各自的恢复数据。
3. 审批模式：根据人类审批（批准、拒绝或其它处理方式），决定后续执行路径。
4. 审核与编辑模式：将模型生成的内容交给人类检查，并允许人类直接修改后继续处理。
5. 工具执行审批模式：在调用工具或执行具有副作用的操作之前，由人类确认是否允许执行。
6. 单节点串行中断模式：在同一个节点内多次触发中断，上一个中断恢复后才能触发下一个。
7. 人类输入验证模式：对人类输入进行校验；当输入不符合要求时，再次中断并要求重新输入。

需要注意，这些模式并不是完全互斥的。例如，工具执行审批本质上也是审批模式的一种，只是它专门应用于工具调用场景。





#### 基础 HITL 模式
##### 什么是`HITL`
`**<font style="color:#DF2A3F;">HITL(Human In The Loop)</font>**`**<font style="color:#DF2A3F;"> 即人在环，是一种经典的人机协同设计模式，它允许计算图在运行过程中暂停，等待人工审批、修正意见或补充信息，</font>**然后从暂停位置恢复执行。





##### 基础`HITL`模式的实现
调用 `interrupt()` 可以暂停状态图的执行，并等待外部输入后恢复运行。因此它是实现 `HITL` 的核心机制。

不过，仅仅调用 `interrupt()` 并不一定构成 `HITL`。只有当中断后的决策、输入或修改确实由人类完成时，才能称为 `HITL`；如果恢复数据完全由程序自动提供，本质上仍然是普通的中断与恢复机制。



```python
from typing import TypedDict

from langgraph.graph import StateGraph,START,END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver


#1. 声明状态
class OverAllState(TypedDict):
    username:str

#2. 声明节点
def node_a(state:OverAllState)->OverAllState:
    username = interrupt("请输入您的姓名")
    return {
        "username":username
    }

#3. 构建图
builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a",node_a)
builder.add_edge(START,"node_a")
builder.add_edge("node_a",END)

#4. 想要使用中断 => 必须配置检查点
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

config = {"configurable":{"thread_id":"123"}}
interrupt_res = graph.invoke({},config=config)
print(interrupt_res)

prompt = interrupt_res['__interrupt__'][0].value
print(prompt)
username = input(prompt)
# 5. 恢复图的中断执行
resume_res = graph.invoke(Command(resume=username),config=config)
print(resume_res)
```

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792403497584-3474cff2-01d1-45f7-8505-9a274b13c74f.png)

```python
============================== -> interrupt_res <- ==============================

{'__interrupt__': [Interrupt(value='请输入您的姓名', id='3543beca616d469fe1ccd35250ef5acf')]}

============================== -> resumed_res <- ==============================

{'username': '小黄'}
```











#### <font style="color:rgb(51, 51, 51);">多个并行中断</font>
**<font style="color:rgb(51, 51, 51);">示例如下</font>**

```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import InMemorySaver

class OverAllState(TypedDict):
    username: str
    age: int

def node_a(state: OverAllState) -> OverAllState:
    username = interrupt("请输入您的姓名")
    return {
        "username": username
    }

def node_b(state: OverAllState) -> OverAllState:
    age = interrupt("请输入您的年龄")
    return {
        "age": age
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}
interrupted_res = graph.invoke({}, config=config)
print('=' * 30, '-> interrupt_res <-', '=' * 30)
print(interrupted_res)

# 构建 中断ID -> 恢复指令 的映射
resume_map = {}
for i in interrupted_res['__interrupt__']:
    user_input = input(f"{i.value}: ")
    if "年龄" in i.value:
        resume_map[i.id] = int(user_input)
    else:
        resume_map[i.id] = user_input

resumed_res = graph.invoke(Command(resume=resume_map), config=config)
print('=' * 30, '-> resumed_res <-', '=' * 30)
print(resumed_res)
```

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

```python
============================== -> interrupt_res <- ==============================

{'__interrupt__': [Interrupt(value='请输入您的年龄', id='2770f00729006c756d481fffa712bb27'), Interrupt(value='请输入您的姓名', id='db7e81e1d4f5cab611f07e0469e99854')]}

============================== -> resumed_res <- ==============================

{'username': '小黄', 'age': 19}
```

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>







#### <font style="color:rgb(51, 51, 51);">审批模式</font>
> 在 Agent 执行敏感或高风险操作前，强制暂停流程，等待人类审查并做出决策后，才决定是否继续。它将人类的监督从被动的“事后监控”转变为主动的“事前控制”，是构建可靠、安全 AI 应用的关键一环。
>





**<font style="color:rgb(51, 51, 51);">示例如下</font>**

**<font style="color:rgb(51, 51, 51);"></font>**

**<font style="color:rgb(51, 51, 51);"></font>**

```python
from time import sleep
from typing import TypedDict, Literal

from langchain_core.messages import HumanMessage
from langgraph.graph import StateGraph,START,END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv
load_dotenv(override=True)

model =ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking":{
            "type":"disabled"
        }
    }
)

#1. 声明状态
class OverAllState(TypedDict):
    topic:str
    poem:str
    is_approved:bool

#2. 声明节点
def approve_node(state:OverAllState) -> Command[Literal["llm_node","default_node"]]:
    is_approved = interrupt("是否同意调用模型?")
    goto = "llm_node" if is_approved else "default_node"
    return Command(
        goto=goto,
        update={"is_approved":is_approved}
    )

def llm_node(state:OverAllState) -> OverAllState:
    topic = state["topic"]
    res = model.invoke([HumanMessage(content=f"帮我写一首关于{topic}主题的七言绝句,只写诗句,不需要赏析")]).content

    return {
        "poem":res
    }

def default_node(state:OverAllState) -> OverAllState:
    return {
        "poem":"请求被拒绝"
    }

#3. 构建图
builder = StateGraph(state_schema=OverAllState)

builder.add_node("approve_node",approve_node)
builder.add_node("llm_node",llm_node)
builder.add_node("default_node",default_node)
builder.add_edge(START,"approve_node")
builder.add_edge("llm_node",END)
builder.add_edge("default_node",END)

#4. 添加检查点后端
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

#5. 首次执行 -> 触发中断
config = {"configurable":{"thread_id":"234"}}
interrupt_res = graph.invoke({"topic":"菊花"},config=config)
print(interrupt_res)
#6. 人工审核
user_approved = input("是否同意调用模型?(y/n)").strip().lower() == 'y'
# 重新调用图
approved_res = graph.invoke(Command(resume=user_approved),config=config)
print(approved_res)

#7. 审核不通过
config1 = {"configurable":{"thread_id":"456"}}
interrupt_res = graph.invoke({"topic":"牡丹花"},config=config1)
print(interrupt_res)

user_approved = input("是否同意调用模型?(y/n)").strip().lower() == 'y'
# 重新调用图
approved_res = graph.invoke(Command(resume=user_approved),config=config1)
print(approved_res)
```

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

**<font style="color:rgb(51, 51, 51);"></font>**

```python
{'topic': '菊花', '__interrupt__': [Interrupt(value='是否同意调用模型?', id='91edf41ecf736a242c18fd61d40ad5b4')]}

{'topic': '菊花', 'poem': '《咏菊》\n西风猎猎卷霜华，独抱秋心隐士家。\n冷艳不争桃李色，寒香偏入野人茶。', 'is_approved': True}

{'topic': '牡丹花', '__interrupt__': [Interrupt(value='是否同意调用模型?', id='47958afc1277b7ce6bf850a5f27f2e46')]}
{'topic': '牡丹花', 'poem': '请求被拒绝', 'is_approved': False}
```









#### <font style="color:rgb(51, 51, 51);">审核与编辑模式</font>
**<font style="color:rgb(51, 51, 51);">示例如下</font>**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek
from langgraph.checkpoint.memory import InMemorySaver
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
    reviewed_poem: str

def llm_node(state: OverAllState) -> OverAllState:
    topic = state['topic']

    res = model.invoke([HumanMessage(content=f"帮我写一首关于 {topic} 的七言绝句，只给出诗句，不要赏析")]).content
    return {
        "poem": res
    }

def review_node(state: OverAllState) -> OverAllState:
    reviewed_poem = interrupt({
        "instruction": "请审核并修改大模型生成的七言绝句",
        "poem": state['poem']
    })
    return {
        "reviewed_poem": reviewed_poem
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("review_node", review_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "review_node")
builder.add_edge("review_node", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "review_test"}}
interrupted_res = graph.invoke({"topic": "布偶猫"}, config=config)
print('=' * 30, '-> interrupt_res <-', '=' * 30)
print(interrupted_res)

print("原始诗句：")
print(interrupted_res['__interrupt__'][0].value['poem'])
user_review = input("请审核并修改诗句（直接回车保留原诗）: ")
if not user_review.strip():
    user_review = interrupted_res['__interrupt__'][0].value['poem']
reviewed_res = graph.invoke(Command(resume=user_review), config=config)
print('=' * 30, '-> reviewed_res <-', '=' * 30)
print(reviewed_res)
```

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

```python
============================== -> interrupt_res <- ==============================

{'topic': '布偶猫', 'poem': '《布偶猫》\n冰眸玉骨雪团身，蓝影衔春卧月轮。\n偶启朱唇呼入梦，云鬟一枕醉红尘。', '__interrupt__': [Interrupt(value={'instruction': '请审核并修改大模型生成的七言绝句', 'poem': '《布偶猫》\n冰眸玉骨雪团身，蓝影衔春卧月轮。\n偶启朱唇呼入梦，云鬟一枕醉红尘。'}, id='de13581234c57fb8b5d9620295b8239e')]}

============================== -> reviewed_res <- ==============================

{'topic': '布偶猫', 'poem': '《布偶猫》\n冰眸玉骨雪团身，蓝影衔春卧月轮。\n偶启朱唇呼入梦，云鬟一枕醉红尘。', 'reviewed_poem': '已审核：《布偶猫》\n冰眸玉骨雪团身，蓝影衔春卧月轮。\n偶启朱唇呼入梦，云鬟一枕醉红尘。'}
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792403693802-8e63703f-e9f9-4a8d-a1a0-9f290b8a02bf.png)





#### <font style="color:rgb(51, 51, 51);">工具执行审批模式</font>
**<font style="color:rgb(51, 51, 51);">示例如下</font>**

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langchain.messages import HumanMessage, ToolMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from loguru import logger
from dotenv import load_dotenv
load_dotenv(override=True)

@tool(parse_docstring=True)
def get_weather(city: str) -> str:
    """
    查询指定城市的当日天气

    Args:
        city: 城市名称
    """
    is_approved = interrupt({
        "action": "get_weather",
        "question": "是否同意查询天气？"
    })

    logger.info("is_approved: {}", is_approved)
    if is_approved:
        return f"{city} 今天天气不错"
    else:
        return "用户拒绝查询天气"

tools_by_name = {"get_weather": get_weather}

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)
model_with_tools = model.bind_tools([get_weather])

def llm_node(state: MessagesState) -> MessagesState:
    messages = state['messages']
    response = model_with_tools.invoke(messages)

    return {
        "messages": [response]
    }

def tool_node(state: MessagesState) -> MessagesState:
    last_msg = state['messages'][-1]

    tool_msgs = []
    for tool_call in last_msg.tool_calls:
        tool = tools_by_name[tool_call["name"]]
        logger.info("工具 {} 被调用, 对应的 tool_call: {}", tool_call["name"], tool_call)
        tool_res = tool.invoke(tool_call["args"])
        tool_msg = ToolMessage(
            name = tool_call["name"],
            content = tool_res,
            tool_call_id = tool_call["id"]
        )
        tool_msgs.append(tool_msg)

    return {
        "messages": tool_msgs
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    if state['messages'][-1].tool_calls:
        return "tool_node"
    return END

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", tool_node)
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "tool_test"}}
interrupted_res = graph.invoke({"messages": [HumanMessage("今天北京天气如何？")]}, config=config)
print('=' * 30, '-> interrupt_res <-', '=' * 30)
for msg in interrupted_res['messages']:
    msg.pretty_print()
print('=' * 30, '-> interrupt_info <-', '=' * 30)
print(interrupted_res['__interrupt__'])

user_approved = input("是否同意查询天气？(y/n): ").strip().lower() == 'y'
approved_res = graph.invoke(Command(resume=user_approved), config=config)
print('=' * 30, '-> approved_res <-', '=' * 30)
for msg in approved_res['messages']:
    msg.pretty_print()
```

**<font style="color:rgb(51, 51, 51);">运行结果如下</font>**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792403703942-5fee7e9a-80d2-4017-b384-a3ed79e0d3a4.png)

```python
2026-06-16 14:51:50.662 | INFO     | __main__:tool_node:59 - 工具 get_weather 被调用, 对应的 tool_call: {'name': 'get_weather', 'args': {'city': '北京'}, 'id': 'call_00_EGxsgveRV6mFkXsTtN4H9369', 'type': 'tool_call'}
============================== -> interrupt_res <- ==============================
================================ Human Message =================================

今天北京天气如何？
================================== Ai Message ==================================

好的，我来查询一下今天北京的天气情况。
Tool Calls:
  get_weather (call_00_EGxsgveRV6mFkXsTtN4H9369)
 Call ID: call_00_EGxsgveRV6mFkXsTtN4H9369
  Args:
    city: 北京
============================== -> interrupt_info <- ==============================
[Interrupt(value={'action': 'get_weather', 'question': '是否同意查询天气？'}, id='907128d6bef87a8473833ec0c90589ea')]
2026-06-16 14:51:50.675 | INFO     | __main__:tool_node:59 - 工具 get_weather 被调用, 对应的 tool_call: {'name': 'get_weather', 'args': {'city': '北京'}, 'id': 'call_00_EGxsgveRV6mFkXsTtN4H9369', 'type': 'tool_call'}
2026-06-16 14:51:50.677 | INFO     | __main__:get_weather:27 - is_approved: True
============================== -> approved_res <- ==============================
================================ Human Message =================================

今天北京天气如何？
================================== Ai Message ==================================

好的，我来查询一下今天北京的天气情况。
Tool Calls:
  get_weather (call_00_EGxsgveRV6mFkXsTtN4H9369)
 Call ID: call_00_EGxsgveRV6mFkXsTtN4H9369
  Args:
    city: 北京
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================== Ai Message ==================================

今天北京的天气**不错**哦！☀️ 是个适合出行的好天气。如果需要更详细的天气信息（如温度、风力等），可以随时告诉我，我帮你进一步查询！
```







#### 单节点串行中断模式
示例如下

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command

class OverAllState(TypedDict):
    username: str # 姓名
    age: int # 年龄
    gender: Literal["male", "female"] # 性别

def get_info_node(state: OverAllState) -> OverAllState:
    username = interrupt("请输入您的用户名：")
    age = interrupt("请输入您的年龄：")
    gender = interrupt("请输入您的性别：(male/female)")

    return {
        "username": username,
        "age": age,
        "gender": gender
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("get_info_node", get_info_node)
builder.add_edge(START, "get_info_node")
builder.add_edge("get_info_node", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "seq_interrupt_test"}}
username_interrupted_res = graph.invoke({}, config=config)
print('=' * 30, '-> username_interrupted_res <-', '=' * 30)
print(username_interrupted_res)

user_name = input("请输入您的用户名：")
age_interrupted_res = graph.invoke(Command(resume=user_name), config=config)
print('=' * 30, '-> age_interrupted_res <-', '=' * 30)
print(age_interrupted_res)

user_age = input("请输入您的年龄：")
gender_interrupted_res = graph.invoke(Command(resume=int(user_age)), config=config)
print('=' * 30, '-> gender_interrupted_res <-', '=' * 30)
print(gender_interrupted_res)

user_gender = input("请输入您的性别：(male/female): ")
resumed_res = graph.invoke(Command(resume=user_gender), config=config)
print('=' * 30, '-> resumed_res <-', '=' * 30)
print(resumed_res)
```

运行结果如下

```python
============================== -> username_interrupted_res <- ==============================
{'__interrupt__': [Interrupt(value='请输入您的用户名：', id='0359f8a0b1ca357d2d092d5c55d80ec1')]}
============================== -> age_interrupted_res <- ==============================
{'__interrupt__': [Interrupt(value='请输入您的年龄：', id='0359f8a0b1ca357d2d092d5c55d80ec1')]}
============================== -> gender_interrupted_res <- ==============================
{'__interrupt__': [Interrupt(value='请输入您的性别：(male/female)', id='0359f8a0b1ca357d2d092d5c55d80ec1')]}
============================== -> resumed_res <- ==============================
{'username': '小黄', 'age': 15, 'gender': 'male'}
```

中断恢复时，整个被中断的节点函数都会重新运行。

+ **<font style="color:#DF2A3F;">单个节点中串行地多次调用 </font>**`**<font style="color:#DF2A3F;">interrupt()</font>**`**<font style="color:#DF2A3F;"> 函数时，检查点存储器会记录历史的 </font>**`**<font style="color:#DF2A3F;">resume</font>**`**<font style="color:#DF2A3F;"> 信息</font>**，`LangGraph` 运行时会读取这些信息，并在 `interrupt()` 函数中维护索引，按照节点内调用 `interrupt()` 函数的顺序，**<font style="color:#DF2A3F;">逐个取出历史 </font>**`**<font style="color:#DF2A3F;">resume</font>**`**<font style="color:#DF2A3F;"> 的值，并将它们作为 </font>**`**<font style="color:#DF2A3F;">interrupt()</font>**`**<font style="color:#DF2A3F;"> 函数的返回值。</font>**
+ 所以，已经被恢复的 `interrupt()` 不会被重复触发。并且，由此可以推断，我们要保证历史 `resume` 可以被正确应用，**<font style="color:#DF2A3F;">就应保证中断恢复前后的多次 </font>**`**<font style="color:#DF2A3F;">interrupt()</font>**`**<font style="color:#DF2A3F;"> 相对顺序保持不变。</font>**
+ **<font style="color:#DF2A3F;">当历史 </font>**`**<font style="color:#DF2A3F;">resume</font>**`**<font style="color:#DF2A3F;"> 耗尽后，本次恢复运行时传入的 </font>**`**<font style="color:#DF2A3F;">resume</font>**`**<font style="color:#DF2A3F;"> 会作为本次中断的返回值</font>**，然后节点函数继续运行。

所有中断都触发并恢复后，计算图正常结束。







### 使用规范


#### <font style="color:#DF2A3F;">不要用</font>`<font style="color:#DF2A3F;">try/catch</font>`<font style="color:#DF2A3F;">包裹</font>`<font style="color:#DF2A3F;">interrupt()</font>`<font style="color:#DF2A3F;">调用</font>
中断的触发是通过抛出 `GraphInterrupt` 异常实现的，如果用 `try/catch` 包裹，则**<font style="color:#DF2A3F;">底层运行时无法感知断点，不会中断计算图。</font>**



```python
def node_a(state: State):
    # ✅ 正确用法：先打断点，再处理异常
    interrupt("What's your name?")
    try:
        fetch_data()  # 这一步可能失败
    except Exception as e:
        print(e)
    return state
```



```python
def node_a(state: State):
    # ❌ 错误用法，用 try/catch 包裹 interrupt()，这将会导致 GraphInterrupt 被捕获而无法触发中断
    try:
        interrupt("What's your name?")
        fetch_data()  # 这一步可能失败
    except Exception as e:
        print(e)
    return state
```

####  


#### <font style="color:#DF2A3F;">不要更改单个节点内</font>`<font style="color:#DF2A3F;">interrupt</font>`<font style="color:#DF2A3F;">的调用顺序</font>
上文提到：

1. 恢复运行时整个节点函数都会重新运行，而非精确地从断点继续
2. 同一节点中存在多个断点时：历史恢复记录被记录在检查点中，并在恢复运行时按顺序加载

一旦中断恢复前后的断点顺序、数量不完全一致，将会导致语义混乱，程序执行结果无法满足预期。

这条规范就是为了避免这样的情况。



```python
def node_a(state: State):
    # ✅ 正确用法，断点顺序固定
    name = interrupt("What's your name?")
    age = interrupt("What's your age?")
    city = interrupt("What's your city?")

    return {
        "name": name,
        "age": age,
        "city": city
    }
```







**错误用法**

1. **<font style="color:#DF2A3F;">可能跳过其中部分断点</font>**

```python
def node_a(state: State):
    # ❌ 错误用法，以状态为条件，可能跳过部分断点
    name = interrupt("What's your name?")

    # 第一次运行到此处时，可能不会触发分支内的断点
    # 恢复时，resume 本应作为 city，但如果分支内断点触发，将被错误地作为 age，导致语义混乱
    if state.get("needs_age"):
        age = interrupt("What's your age?")

    city = interrupt("What's your city?")

    return {"name": name, "city": city}
```

断点以状态为条件，如果中断恢复前后状态发生了变化，断点的顺序和数量可能发生变化，导致语义混乱。



2.**<font style="color:#DF2A3F;"> 基于不确定的数据循环，并在其中设置断点</font>**

```python
def node_a(state: State):
    # ❌ 错误用法：基于不确定的数据循环
    # 每次调用时的断点数量可能不同
    results = []
    for item in state.get("dynamic_list", []):  # List might change between runs
        result = interrupt(f"Approve {item}?")
        results.append(result)

    return {"results": results}
```

基于状态中可变的值循环，并在循环中设置断点，如果中断恢复前后状态发生变化，断点数量可能变化，导致语义混乱。

<font style="color:rgb(51, 51, 51);"></font>





#### <font style="color:#DF2A3F;">不要在</font>`<font style="color:#DF2A3F;">interrupt()</font>`<font style="color:#DF2A3F;">中传递复杂类型</font>
+ `LangGraph` 运行时会将 `interrupt()` 函**<font style="color:#DF2A3F;">数接收到的参数经过 </font>**`**<font style="color:#DF2A3F;">JSON</font>**`**<font style="color:#DF2A3F;"> 序列化 之后传递给调用者。</font>**
+ 如果传递不支持 `JSON` 序列化 的复杂类型，如函数，将会抛出异常，如下所示

```python
TypeError: Type is not msgpack serializable: Interrupt
```







**正确用法**

1. 简单类型

```python
def node_a(state: State):
    # ✅ 正确用法：传递可以序列化的简单类型数据
    name = interrupt("What's your name?")
    count = interrupt(42)
    approved = interrupt(True)

    return {"name": name, "count": count, "approved": approved}
```

2. 结构化数据

```python
def node_a(state: State):
    # ✅ 正确用法：传递仅包含基本数据类型的可序列化字典
    response = interrupt({
        "question": "Enter user details",
        "fields": ["name", "email", "age"],
        "current_values": state.get("user", {})
    })

    return {"user": response}
```





**错误用法**

1. 函数

```python
def validate_input(value):
    return len(value) > 0

def node_a(state: State):
    # ❌ 错误用法：传递函数
    # 函数不支持 JSON 序列化
    response = interrupt({
        "question": "What's your name?",
        "validator": validate_input  # 此处会导致失败
    })
    return {"name": response}
```

2. 类的实例

```python
class DataProcessor:
    def __init__(self, config):
        self.config = config

def node_a(state: State):
    processor = DataProcessor({"mode": "strict"})

    # ❌ 错误用法：传递类的实例
    # 此处的自定义实例不支持 JSON 序列化
    response = interrupt({
        "question": "Enter data to process",
        "processor": processor  # 这将会导致失败
    })
    return {"result": response}
```





#### <font style="color:#DF2A3F;">断点之前的副作用操作必须是幂等的</font>
+ 副作用操作：修改了外部环境的操作，如
    - 写数据库
    - 写文件
    - 发送邮件或消息
+ 幂等性：**<font style="color:#DF2A3F;">多次执行的结果等价于一次执行。</font>**

**<font style="color:#DF2A3F;"></font>**

我们知道，中断恢复时，**<font style="color:#DF2A3F;">断点所在的函数会被重复执行</font>**，所以，如果断点之前存在不满足幂等性的副作用操作，将会导致多次调用结果不一致。

如：写数据操作不满足幂等性，可能导致写入多条重复记录。





**正确用法**

1. 在断点之前使用满足幂等性的操作

```python
def node_a(state: State):
    # ✅ 正确用法：使用幂等的 upsert 操作
    # 多次运行结果一致
    db.upsert_user(
        user_id=state["user_id"],
        status="pending_approval"
    )

    approved = interrupt("Approve this change?")

    return {"approved": approved}
```

`upsert` 的语义是：写入或更新，如果有相同记录则覆盖更新，否则新增。满足幂等性。





2. 将副作用操作放在断点之后

```python
def node_a(state: State):
    # ✅ 正确用法：将副作用操作放在断点之后
    # 确保副作用操作只运行一次
    approved = interrupt("Approve this change?")

    if approved:
        db.create_audit_log(
            user_id=state["user_id"],
            action="approved"
        )

    return {"approved": approved}
```

上述代码将副作用操作置于断点之后，之后中断触发并恢复运行之后，副作用操作才会被执行，后者只会被执行一次。



3. 将副作用操作和断点置于不同的节点

```python
def approval_node(state: State):
    # ✅ 正确用法：当前节点只做中断
    approved = interrupt("Approve this change?")

    return {"approved": approved}

def notification_node(state: State):
    # ✅ 正确用法：副作用操作发生在独立的节点中
    # 当前节点独立于中断，所以副作用操作，不会因为中断恢复而重复执行
    if (state.approved):
        send_notification(
            user_id=state["user_id"],
            status="approved"
        )

    return state
```

这种方式更彻底，将中断和副作用操作完全隔离。





**错误用法**

1. 在断点之前向数据库中新增数据

```python
def node_a(state: State):
    # ❌ 错误用法：在断点之前向数据库中新增数据
    # 中断恢复会导致数据重复
    audit_id = db.create_audit_log({
        "user_id": state["user_id"],
        "action": "pending_approval",
        "timestamp": datetime.now()
    })

    approved = interrupt("Approve this change?")

    return {"approved": approved, "audit_id": audit_id}
```

2. 将数据追加到已存在的列表

```python
def node_a(state: State):
    # ❌ 错误用法：在断点之前向列表追加数据
    # 这将会在中断恢复时导致列表数据重复
    db.append_to_history(state["user_id"], "approval_requested")

    approved = interrupt("Approve this change?")

    return {"approved": approved}
```







### 中断触发和恢复前后的检查点
##### 示例代码
```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import InMemorySaver

class OverAllState(TypedDict):
    username: str
    age: int

def node_a(state: OverAllState) -> OverAllState:
    username = interrupt("请输入您的姓名")
    age = interrupt("请输入您的年龄")
    return {
        "username": username,
        "age": age
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "123"}}
username_interrupted_res = graph.invoke({}, config=config)
print('=' * 30, '-> username_interrupted_res <-', '=' * 30)
print(username_interrupted_res)
username_interrupted_his = list(graph.get_state_history(config=config))
print('=' * 30, '-> username_interrupted_his <-', '=' * 30)
print(username_interrupted_his)

age_interrupted_res = graph.invoke(Command(resume="小黄"), config=config)
print('=' * 30, '-> age_interrupted_res <-', '=' * 30)
print(age_interrupted_res)
age_interrupted_his = list(graph.get_state_history(config=config))
print('=' * 30, '-> age_interrupted_his <-', '=' * 30)
print(age_interrupted_his)

resumed_res = graph.invoke(Command(resume=123), config=config)
print('=' * 30, '-> resumed_res <-', '=' * 30)
print(resumed_res)
resumed_his = list(graph.get_state_history(config=config))
print('=' * 30, '-> resumed_his <-', '=' * 30)
print(resumed_his)
```

##### 运行结果分析
###### 第一次中断
调用结果

```python
============================== -> username_interrupted_res <- ==============================
{'__interrupt__': [Interrupt(value='请输入您的姓名', id='488d1fa95a4053d41ab09786f5260ae9')]}
```

历史检查点

```python
============================== -> username_interrupted_his <- ==============================
[
    StateSnapshot(
        values={},
        next=('node_a',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4aa0-631a-8000-12a45e5b02a0'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.195818+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4a9d-68e8-bfff-3573d795ed80'
            }
        },
        tasks=(
            PregelTask(
                id='a7e05e86-30c1-a590-a7bf-e3542eb53d4b',
                name='node_a',
                path=('__pregel_pull', 'node_a'),
                error=None,
                interrupts=(
                    Interrupt(
                        value='请输入您的姓名',
                        id='488d1fa95a4053d41ab09786f5260ae9'
                    ),
                ),
                state=None,
                result=None
            ),
        ),
        interrupts=(
            Interrupt(
                value='请输入您的姓名',
                id='488d1fa95a4053d41ab09786f5260ae9'
            ),
        )
    ),

    StateSnapshot(
        values={},
        next=('__start__',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4a9d-68e8-bfff-3573d795ed80'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.194743+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='173d6621-cd7b-3edd-a3d5-8d1c3bb8dc53',
                name='__start__',
                path=('__pregel_pull', '__start__'),
                error=None,
                interrupts=(),
                state=None,
                result={}
            ),
        ),
        interrupts=()
    )
]
```

当前工作流只有一个节点，所以只有一个真正执行用户自定义节点的超步，对应的超步编号为 `1`

中断触发时，检查点存储器中记录的最新检查点超步为 `0`，中断信息只能和这个超步绑定

中断信息存了两份：

+ `**<font style="color:#DF2A3F;">StateSnapshot</font>**`**<font style="color:#DF2A3F;"> 的 </font>**`**<font style="color:#DF2A3F;">tasks</font>**`**<font style="color:#DF2A3F;"> 属性下记录的 </font>**`**<font style="color:#DF2A3F;">PregelTask</font>**`**<font style="color:#DF2A3F;"> 实例的 </font>**`**<font style="color:#DF2A3F;">interrupts</font>**`**<font style="color:#DF2A3F;"> 字段记录了当前任务触发的中断信息。</font>**
+ `**<font style="color:#DF2A3F;">StateSnapshot</font>**`**<font style="color:#DF2A3F;"> 的 </font>**`**<font style="color:#DF2A3F;">interrupts</font>**`**<font style="color:#DF2A3F;"> 属性记录了当前超步发生的所有中断</font>**

当存在多个并行中断时

+ 中断信息被记录在各自所属任务的 `PregelTask` 实例中
+ 同时也会被汇总在 `StateSnapshot` 的 `tasks` 属性中



###### 第一次恢复
调用结果

```python
============================== -> age_interrupted_res <- ==============================
{'__interrupt__': [Interrupt(value='请输入您的年龄', id='488d1fa95a4053d41ab09786f5260ae9')]}
```

历史检查点

```python
============================== -> age_interrupted_his <- ==============================

[
    StateSnapshot(
        values={},
        next=('node_a',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4aa0-631a-8000-12a45e5b02a0'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.195818+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4a9d-68e8-bfff-3573d795ed80'
            }
        },
        tasks=(
            PregelTask(
                id='a7e05e86-30c1-a590-a7bf-e3542eb53d4b',
                name='node_a',
                path=('__pregel_pull', 'node_a'),
                error=None,
                interrupts=(
                    Interrupt(
                        value='请输入您的年龄',
                        id='488d1fa95a4053d41ab09786f5260ae9'
                    ),
                ),
                state=None,
                result={}
            ),
        ),
        interrupts=(
            Interrupt(
                value='请输入您的年龄',
                id='488d1fa95a4053d41ab09786f5260ae9'
            ),
        )
    ),

    StateSnapshot(
        values={},
        next=('__start__',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4a9d-68e8-bfff-3573d795ed80'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.194743+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='173d6621-cd7b-3edd-a3d5-8d1c3bb8dc53',
                name='__start__',
                path=('__pregel_pull', '__start__'),
                error=None,
                interrupts=(),
                state=None,
                result={}
            ),
        ),
        interrupts=()
    )
]
```

`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">checkpoint_id</font>**`**<font style="color:#DF2A3F;">、任务</font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">id</font>**`**<font style="color:#DF2A3F;">、中断</font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">id</font>**`**<font style="color:#DF2A3F;"> 和历史检查点完全一致，只是 </font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">Interrupt</font>**`**<font style="color:#DF2A3F;"> 实例的 </font>**`**<font style="color:#DF2A3F;background-color:rgb(243, 244, 244);">value</font>**`**<font style="color:#DF2A3F;"> 属性变成了第二次中断的值。</font>**

<font style="color:rgb(51, 51, 51);">由此可见，恢复运行时检查点的更新确实是覆盖写入，历史中断信息不会被保留。</font>

<font style="color:rgb(51, 51, 51);">并且，中断恢复时传递的 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">resume</font>**`<font style="color:rgb(51, 51, 51);"> 在 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">graph.get_state_history()</font>**`<font style="color:rgb(51, 51, 51);"> 时也不可见。</font>









###### 第二次恢复
调用结果

```python
============================== -> resumed_res <- ==============================
{'username': '小黄', 'age': 123}
```

历史检查点

```python
============================== -> resumed_his <- ==============================

[
    StateSnapshot(
        values={
            'username': '小黄',
            'age': 123
        },
        next=(),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4ab5-60bc-8001-df8746dc9400'
            }
        },
        metadata={
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.204348+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4aa0-631a-8000-12a45e5b02a0'
            }
        },
        tasks=(),
        interrupts=()
    ),

    StateSnapshot(
        values={},
        next=('node_a',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4aa0-631a-8000-12a45e5b02a0'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.195818+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4a9d-68e8-bfff-3573d795ed80'
            }
        },
        tasks=(
            PregelTask(
                id='a7e05e86-30c1-a590-a7bf-e3542eb53d4b',
                name='node_a',
                path=('__pregel_pull', 'node_a'),
                error=None,
                interrupts=(
                    Interrupt(
                        value='请输入您的年龄',
                        id='488d1fa95a4053d41ab09786f5260ae9'
                    ),
                ),
                state=None,
                result={
                    'username': '小黄',
                    'age': 123
                }
            ),
        ),
        interrupts=(
            Interrupt(
                value='请输入您的年龄',
                id='488d1fa95a4053d41ab09786f5260ae9'
            ),
        )
    ),

    StateSnapshot(
        values={},
        next=('__start__',),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a255-4a9d-68e8-bfff-3573d795ed80'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-17T08:19:59.194743+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='173d6621-cd7b-3edd-a3d5-8d1c3bb8dc53',
                name='__start__',
                path=('__pregel_pull', '__start__'),
                error=None,
                interrupts=(),
                state=None,
                result={}
            ),
        ),
        interrupts=()
    )
]
```

<font style="color:rgb(51, 51, 51);">两个断点全部恢复运行后，编号为 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">1</font>**`<font style="color:rgb(51, 51, 51);"> 的超步终于完成，任务实例的 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">result</font>**`<font style="color:rgb(51, 51, 51);"> 被更新，状态被更新，最后一个超步的 </font>`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">values</font>**`<font style="color:rgb(51, 51, 51);"> 是更新后的状态值，计算图运行完成。</font>













#### 多个并行中断触发时的检查点
#####  示例代码
```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import InMemorySaver

class OverAllState(TypedDict):
    username: str
    age: int

def node_a(state: OverAllState) -> OverAllState:
    username = interrupt("请输入您的姓名")
    return {
        "username": username
    }

def node_b(state: OverAllState) -> OverAllState:
    age = interrupt("请输入您的年龄")
    return {
        "age": age
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}
interrupted_res = graph.invoke({}, config=config)
print('=' * 30, '-> interrupt_res <-', '=' * 30)
print(interrupted_res)

list(graph.get_state_history(config=config))

# 6. 恢复执行
resume_map = {}
for i in interrupted_res['__interrupt__']:
    user_input = input(f"{i.value}:")
    if "年龄" in i.value:
        resume_map[i.id] = int(user_input)
    else:
        resume_map[i.id] = user_input

resumed_res = graph.invoke(Command(resume=resume_map),config=config)
print(resumed_res)

list(graph.get_state_history(config=config))
```

#####  结果分析
计算图返回值

```python
============================== -> interrupt_res <- ==============================

{
    '__interrupt__': [
        Interrupt(
            value='请输入您的姓名',
            id='0cae1932a74087f02711c6f1677b4b17'
        ),
        Interrupt(
            value='请输入您的年龄',
            id='cd216e1ad6907c2432636ff5f32cfcf3'
        )
    ]
}
```

同一个超步中并行触发的中断信息被放在列表中返回。

历史检查点列表

```python
[
    StateSnapshot(
        values={},

        next=(
            'node_a',
            'node_b'
        ),

        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a329-8b50-69e0-8000-7bdce1ba7379'
            }
        },

        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },

        created_at='2026-06-17T09:54:56.810630+00:00',

        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a329-8b4e-6085-bfff-3100dea319cc'
            }
        },

        tasks=(
            PregelTask(
                id='689d8040-4b34-02a2-eb32-d901bbb3c67c',
                name='node_a',
                path=(
                    '__pregel_pull',
                    'node_a'
                ),
                error=None,
                interrupts=(
                    Interrupt(
                        value='请输入您的姓名',
                        id='0cae1932a74087f02711c6f1677b4b17'
                    ),
                ),
                state=None,
                result=None
            ),

            PregelTask(
                id='034cbe3c-e0a6-025a-ce03-46436ce38fc3',
                name='node_b',
                path=(
                    '__pregel_pull',
                    'node_b'
                ),
                error=None,
                interrupts=(
                    Interrupt(
                        value='请输入您的年龄',
                        id='cd216e1ad6907c2432636ff5f32cfcf3'
                    ),
                ),
                state=None,
                result=None
            )
        ),

        interrupts=(
            Interrupt(
                value='请输入您的姓名',
                id='0cae1932a74087f02711c6f1677b4b17'
            ),
            Interrupt(
                value='请输入您的年龄',
                id='cd216e1ad6907c2432636ff5f32cfcf3'
            )
        )
    ),


    StateSnapshot(
        values={},

        next=(
            '__start__',
        ),

        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a329-8b4e-6085-bfff-3100dea319cc'
            }
        },

        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },

        created_at='2026-06-17T09:54:56.809574+00:00',

        parent_config=None,

        tasks=(
            PregelTask(
                id='3df213dc-5fa0-a8aa-b1dc-428da9010be6',
                name='__start__',
                path=(
                    '__pregel_pull',
                    '__start__'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={}
            ),
        ),

        interrupts=()
    )
]
```

如前所述，编号为 `1` 的超步执行时中断触发，此时检查点后端中存储的最新的超步编号为 `0`，因此中断信息记录在超步 `0` 的 `StateSnapshot` 实例中。

+ 快照实例的 `interrupts` 属性记录了并行触发的两个中断的信息。
+ 并且，各自的中断信息被记录在对应的任务实例中。

#### 同一超步部分任务触发中断时的检查点
##### 示例代码
```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import InMemorySaver

class OverAllState(TypedDict):
    username: str
    age: int

def node_a(state: OverAllState) -> OverAllState:
    username = interrupt("请输入您的姓名")
    return {
        "username": username
    }

def node_b(state: OverAllState) -> OverAllState:
    age = 12
    return {
        "age": age
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}
interrupted_res = graph.invoke({}, config=config)
print('=' * 30, '-> interrupt_res <-', '=' * 30)
print(interrupted_res)

list(graph.get_state_history(config=config))
```

##### 结果分析
计算图返回值

```python
============================== -> interrupt_res <- ==============================

{
    'age': 12,
    '__interrupt__': [
        Interrupt(
            value='请输入您的姓名',
            id='bbff88193340fd0ed196be702608e385'
        )
    ]
}
```

历史检查点列表

```python
============================== -> state_history <- ==============================

[
    StateSnapshot(
        values={},
        next=(
            'node_a',
            'node_b'
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a32c-b47b-6ca9-8000-79ee3c65e7ce'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-17T09:56:21.658113+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a32c-b479-6766-bfff-901ed37f684d'
            }
        },
        tasks=(
            PregelTask(
                id='e16690fc-c0f8-e9fc-e30d-706a64f24209',
                name='node_a',
                path=(
                    '__pregel_pull',
                    'node_a'
                ),
                error=None,
                interrupts=(
                    Interrupt(
                        value='请输入您的姓名',
                        id='bbff88193340fd0ed196be702608e385'
                    ),
                ),
                state=None,
                result=None
            ),
            PregelTask(
                id='b628264b-d4ca-c09f-5197-6eb1c953b947',
                name='node_b',
                path=(
                    '__pregel_pull',
                    'node_b'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'age': 12
                }
            )
        ),
        interrupts=(
            Interrupt(
                value='请输入您的姓名',
                id='bbff88193340fd0ed196be702608e385'
            ),
        )
    ),

    StateSnapshot(
        values={},
        next=(
            '__start__',
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16a32c-b479-6766-bfff-901ed37f684d'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-17T09:56:21.657164+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='5dc4ad26-1baa-7c18-2580-153a19d4c175',
                name='__start__',
                path=(
                    '__pregel_pull',
                    '__start__'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={}
            ),
        ),
        interrupts=()
    )
]
```

由上可知，同一个超步中：

+ **<font style="color:#DF2A3F;">发生中断的任务，其中断信息被记录在任务实例和快照的 </font>**`**<font style="color:#DF2A3F;">interrupts</font>**`**<font style="color:#DF2A3F;"> 字段中</font>**
+ **<font style="color:#DF2A3F;">未触发中断的任务正常结束，其结果被记录在任务实例的 </font>**`**<font style="color:#DF2A3F;">result</font>**`**<font style="color:#DF2A3F;"> 字段中</font>**

底层负责执行节点逻辑的是超步的第二阶段，如果存在多个并行任务，则会将任务提交到线程池，在独立的线程中运行，**<font style="color:#DF2A3F;">负责执行第二阶段的 </font>**`**<font style="color:#DF2A3F;">PregelRunner</font>**`**<font style="color:#DF2A3F;"> 实</font>**例会收集提交后的 `future` 对象，并等待所有任务完成。

当中断触发时，对应的线程会抛出 `GraphInterrupt`**<font style="color:#DF2A3F;">，对应的 </font>**`**<font style="color:#DF2A3F;">future</font>**`**<font style="color:#DF2A3F;"> 对象会收集异常</font>**。

`PregelRunner` 实例在 `future` 对应的任务完成后，收集中断信息或任务执行结果，**<font style="color:#DF2A3F;">写入检查点存储器。</font>**

**<font style="color:#DF2A3F;">所以一个超步中某个任务触发中断，不会导致其它并行任务的终止，正常运行的任务结果、中断任务的中断信息都被记录在检查点后端。</font>**











## 静态断点
`LangGraph` 提供了用于调试的 静态断点。

它是在状态图编译或调用时设置的，不会在运行时动态触发，**<font style="color:#DF2A3F;">不属于业务逻辑</font>**，所以称为 静态 断点。

动态断点提供了接收人类反馈的接口，**<font style="color:#DF2A3F;">而静态断点只是暂停计算图的运行，不能向调用者传递信息</font>**，也不能接收调用者的反馈。





### 用法说明
1. 静态断点也需要基于检查点恢复，所以必须配置检查点，启用可恢复运行机制。
2. 计算图会在 `**<font style="color:#DF2A3F;">interrupt_before</font>**`**<font style="color:#DF2A3F;"> 指定的节点执行之前产生中断，暂停计算</font>**
3. 计算图会在 `**<font style="color:#DF2A3F;">interrupt_after</font>**`**<font style="color:#DF2A3F;"> 指定的节点执行之后产生中断，暂停计算</font>**
4. 计算图运行到断点位置会中断，和动态断点不同的是，它会在超步边界而非内部中断。

断点前的超步：三个阶段全部完成；断点后的超步：三个阶段都没有开始。

5. 静态断点不会返回任何中断信息，**<font style="color:#DF2A3F;">只会将当前最新的状态返回。</font>**

因此，我们可以用静态断点查看每个超步边界的中间状态。

6. 传入相同的配置并将 `None` 作为输入，再次调用计算图，会从断点位置继续运行。
7. 支持在两个阶段设置静态断点
    - **<font style="color:#DF2A3F;background-color:#FBDE28;">状态图编译时 ： 推荐在编译的时候把中断写清楚即可</font>**

```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
```

    - **<font style="color:#DF2A3F;">计算图调用时</font>**

```python
first_res = graph.invoke(
    {},
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
```

**<font style="color:#DF2A3F;">断点都是在运行时生效，计算图调用时设置的断点优先级更高。</font>**

如果调用时传入的断点列表不为空，则会覆盖编译时配置。



### 底层原理
1. 编译时设置的静态断点保存在编译图的 `interrupt_before_nodes` 和 `interrupt_after_nodes` 中。

运行时采用调用参数优先、编译配置兜底的规则。

调用时传入非空节点列表会完全覆盖编译配置，而 `None` 或空列表会回退到编译配置。

2. `interrupt_before` 在超步的第一阶段：当前超步的任务列表计算完成后、节点任务开始执行前进行检查。
    - 如果检查点中的状态和上一次静态断点或图的初始状态相比，产生了变化
    - 并且当前超步的任务列表和 `interrupt_before` 列表存在交集

则抛出 `GraphInterrupt`。

3. `interrupt_after` 在超步的第三阶段：当前超步的任务全部执行完成、写入应用到通道并创建检查点之后进行检查。
    - 如果检查点中的状态和上一次静态断点或图的初始状态相比，产生了变化
    - 并且当前超步的任务列表和 `interrupt_after` 列表存在交集

则抛出 `GraphInterrupt`。

如果存在这样的拓扑图

4. node_a -> node_b

那么在 `node_a` 之后 和 `node_b` 之前 设置断点只会中断一次。

因为二者对应同一个超步边界，如果同时配置：

    - `node_a` 的 `after` 先触发；
    - 恢复运行时检查点中记录的上次断点可见状态会被更新，和最新状态保持一致
    - 而产生中断之前已经保存了检查点，恢复运行时会从 `node_b` 的超步开始
    - `node_b` 的超步第一阶段，检查点和上次静态断点相比，没有变化，所以不会中断
    - **<font style="color:#DF2A3F;">因此，</font>**`**<font style="color:#DF2A3F;">node_a</font>**`**<font style="color:#DF2A3F;"> 之后 和 </font>**`**<font style="color:#DF2A3F;">node_b</font>**`**<font style="color:#DF2A3F;"> 之前 设置断点只会中断一次</font>**





### 示例
####  编译时设置断点
##### 顺序结构设置断点
示例如下

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

from loguru import logger

class OverAllState(TypedDict):
    final_res: str

def node_a(state: OverAllState) -> OverAllState:
    logger.info("node_a 被执行")
    return {
        "final_res": "node_a 运行的中间结果"
    }

def node_b(state: OverAllState) -> OverAllState:
    logger.info("node_b 被执行")
    return {
        "final_res": "node_b 运行的中间结果"
    }

def node_c(state: OverAllState) -> OverAllState:
    logger.info("node_c 被执行")
    return {
        "final_res": "node_c 运行的中间结果"
    }


builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_c")
builder.add_edge("node_c", END)

checkpointer = InMemorySaver()
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}

logger.info("{}-> 第一次执行 <-{}", "=" * 10, "=" * 10)
first_res = graph.invoke({}, config=config)
logger.info("第一次执行结果：{}", first_res)

logger.info("{}-> 第二次执行 <-{}", "=" * 10, "=" * 10)
second_res = graph.invoke(None, config=config)
logger.info("第二次执行结果：{}", second_res)

logger.info("{}-> 第三次执行 <-{}", "=" * 10, "=" * 10)
third_res = graph.invoke(None, config=config)
logger.info("第三次执行结果：{}", third_res)

logger.info("{}-> 第四次执行 <-{}", "=" * 10, "=" * 10)
final_res = graph.invoke(None, config=config)
logger.info("第四次执行结果：{}", final_res)

logger.info("{}-> 完毕 <-{}", "=" * 10, "=" * 10)
```

运行结果如下

编译时设置的断点在渲染拓扑结构时被感知，图中对应结点新增了断点标记：

__interrupt = before,after

这表示该节点之前和之后都设置了断点。

日志如下



```python
2026-06-18 11:22:52.094 | INFO     | __main__:<module>:51 - ==========-> 第一次执行 <-==========
2026-06-18 11:22:52.100 | INFO     | __main__:<module>:53 - 第一次执行结果：None
2026-06-18 11:22:52.100 | INFO     | __main__:<module>:55 - ==========-> 第二次执行 <-==========
2026-06-18 11:22:52.102 | INFO     | __main__:node_a:11 - node_a 被执行
2026-06-18 11:22:52.104 | INFO     | __main__:<module>:57 - 第二次执行结果：{'final_res': 'node_a 运行的中间结果'}
    2026-06-18 11:22:52.104 | INFO     | __main__:<module>:59 - ==========-> 第三次执行 <-==========
    2026-06-18 11:22:52.105 | INFO     | __main__:node_b:17 - node_b 被执行
    2026-06-18 11:22:52.107 | INFO     | __main__:<module>:61 - 第三次执行结果：{'final_res': 'node_b 运行的中间结果'}
        2026-06-18 11:22:52.107 | INFO     | __main__:<module>:63 - ==========-> 第四次执行 <-==========
        2026-06-18 11:22:52.108 | INFO     | __main__:node_c:23 - node_c 被执行
        2026-06-18 11:22:52.110 | INFO     | __main__:<module>:65 - 第四次执行结果：{'final_res': 'node_c 运行的中间结果'}
            2026-06-18 11:22:52.111 | INFO     | __main__:<module>:67 - ==========-> 完毕 <-==========
```

由日志可知，`**<font style="color:#DF2A3F;">node_a</font>**`**<font style="color:#DF2A3F;"> 和 </font>**`**<font style="color:#DF2A3F;">node_b</font>**`**<font style="color:#DF2A3F;"> 之间只产生了一次中断，和分析相符</font>**。

##### 存在并行节点时设置断点
为了确定断点是 以节点 为边界还是 以超步 为边界设置的，我们在存在并行节点的计算图中测试

示例如下

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

from loguru import logger

class OverAllState(TypedDict):
    final_res: str

def node_a(state: OverAllState) -> OverAllState:
    logger.info("node_a 被执行")
    return {
        "final_res": "node_a 运行的中间结果"
    }

def node_b(state: OverAllState) -> OverAllState:
    logger.info("node_b 被执行")
    return {
        "final_res": "node_b 运行的中间结果"
    }

def node_c(state: OverAllState) -> OverAllState:
    logger.info("node_c 被执行")
    return {
        "final_res": "node_c 运行的中间结果"
    }

def node_d(state: OverAllState) -> OverAllState:
    logger.info("node_d 被执行")
    return {}

def node_e(state: OverAllState) -> OverAllState:
    logger.info("node_e 被执行")
    return {}

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_node("node_d", node_d)
builder.add_node("node_e", node_e)

builder.add_edge(START, "node_a")
builder.add_edge(START, "node_d")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_d", "node_e")
builder.add_edge(["node_b", "node_e"], "node_c")
builder.add_edge("node_c", END)

checkpointer = InMemorySaver()
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}

logger.info("{}-> 第一次执行 <-{}", "=" * 10, "=" * 10)
first_res = graph.invoke({}, config=config)
logger.info("第一次执行结果：{}", first_res)

logger.info("{}-> 第二次执行 <-{}", "=" * 10, "=" * 10)
second_res = graph.invoke(None, config=config)
logger.info("第二次执行结果：{}", second_res)

logger.info("{}-> 第三次执行 <-{}", "=" * 10, "=" * 10)
third_res = graph.invoke(None, config=config)
logger.info("第三次执行结果：{}", third_res)

logger.info("{}-> 第四次执行 <-{}", "=" * 10, "=" * 10)
final_res = graph.invoke(None, config=config)
logger.info("第四次执行结果：{}", final_res)

logger.info("{}-> 完毕 <-{}", "=" * 10, "=" * 10)
```

运行结果如下

```python
2026-06-18 11:28:49.920 | INFO     | __main__:<module>:62 - ==========-> 第一次执行 <-==========
2026-06-18 11:28:49.922 | INFO     | __main__:<module>:64 - 第一次执行结果：None
2026-06-18 11:28:49.923 | INFO     | __main__:<module>:66 - ==========-> 第二次执行 <-==========
2026-06-18 11:28:49.925 | INFO     | __main__:node_a:11 - node_a 被执行
2026-06-18 11:28:49.927 | INFO     | __main__:node_d:29 - node_d 被执行
2026-06-18 11:28:49.931 | INFO     | __main__:<module>:68 - 第二次执行结果：{'final_res': 'node_a 运行的中间结果'}
    2026-06-18 11:28:49.931 | INFO     | __main__:<module>:70 - ==========-> 第三次执行 <-==========
    2026-06-18 11:28:49.933 | INFO     | __main__:node_b:17 - node_b 被执行
    2026-06-18 11:28:49.935 | INFO     | __main__:node_e:33 - node_e 被执行
    2026-06-18 11:28:49.939 | INFO     | __main__:<module>:72 - 第三次执行结果：{'final_res': 'node_b 运行的中间结果'}
        2026-06-18 11:28:49.940 | INFO     | __main__:<module>:74 - ==========-> 第四次执行 <-==========
        2026-06-18 11:28:49.940 | INFO     | __main__:node_c:23 - node_c 被执行
        2026-06-18 11:28:49.942 | INFO     | __main__:<module>:76 - 第四次执行结果：{'final_res': 'node_c 运行的中间结果'}
            2026-06-18 11:28:49.943 | INFO     | __main__:<module>:78 - ==========-> 完毕 <-==========
```

`node_a` 和 `node_d` 属于同一个超步

`node_b` 和 `node_e` 属于同一个超步

**<font style="color:#DF2A3F;">我们只在 </font>**`**<font style="color:#DF2A3F;">node_a</font>**`**<font style="color:#DF2A3F;"> 和 </font>**`**<font style="color:#DF2A3F;">node_b</font>**`**<font style="color:#DF2A3F;"> 前后设置了超步，但它们的并行节点也被中断了。</font>**

**<font style="color:#DF2A3F;">显然，</font>****<font style="color:#DF2A3F;background-color:#FBDE28;">中断发生在超步边界</font>****<font style="color:#DF2A3F;">，而非节点边界，与分析相符。</font>**

#### 调用时设置断点
调用时的断点配置只对本次调用生效。

##### 正确用法：恢复调用时设置相同的断点
示例如下

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

from loguru import logger

class OverAllState(TypedDict):
    final_res: str

def node_a(state: OverAllState) -> OverAllState:
    logger.info("node_a 被执行")
    return {
        "final_res": "node_a 运行的中间结果"
    }

def node_b(state: OverAllState) -> OverAllState:
    logger.info("node_b 被执行")
    return {
        "final_res": "node_b 运行的中间结果"
    }

def node_c(state: OverAllState) -> OverAllState:
    logger.info("node_c 被执行")
    return {
        "final_res": "node_c 运行的中间结果"
    }


builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_c")
builder.add_edge("node_c", END)

checkpointer = InMemorySaver()
graph = builder.compile(
    checkpointer=checkpointer
)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}

logger.info("{}-> 第一次执行 <-{}", "=" * 10, "=" * 10)
first_res = graph.invoke(
    {},
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
logger.info("第一次执行结果：{}", first_res)

logger.info("{}-> 第二次执行 <-{}", "=" * 10, "=" * 10)
second_res = graph.invoke(
    None,
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
logger.info("第二次执行结果：{}", second_res)

logger.info("{}-> 第三次执行 <-{}", "=" * 10, "=" * 10)
third_res = graph.invoke(
    None,
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
logger.info("第三次执行结果：{}", third_res)

logger.info("{}-> 第四次执行 <-{}", "=" * 10, "=" * 10)
final_res = graph.invoke(
    None,
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
logger.info("第四次执行结果：{}", final_res)

logger.info("{}-> 完毕 <-{}", "=" * 10, "=" * 10)
```

运行结果如下

```python
2026-06-18 11:32:55.457 | INFO     | __main__:<module>:49 - ==========-> 第一次执行 <-==========
2026-06-18 11:32:55.460 | INFO     | __main__:<module>:56 - 第一次执行结果：None
2026-06-18 11:32:55.460 | INFO     | __main__:<module>:58 - ==========-> 第二次执行 <-==========
2026-06-18 11:32:55.461 | INFO     | __main__:node_a:11 - node_a 被执行
2026-06-18 11:32:55.463 | INFO     | __main__:<module>:65 - 第二次执行结果：{'final_res': 'node_a 运行的中间结果'}
    2026-06-18 11:32:55.463 | INFO     | __main__:<module>:67 - ==========-> 第三次执行 <-==========
    2026-06-18 11:32:55.463 | INFO     | __main__:node_b:17 - node_b 被执行
    2026-06-18 11:32:55.465 | INFO     | __main__:<module>:74 - 第三次执行结果：{'final_res': 'node_b 运行的中间结果'}
        2026-06-18 11:32:55.466 | INFO     | __main__:<module>:76 - ==========-> 第四次执行 <-==========
        2026-06-18 11:32:55.467 | INFO     | __main__:node_c:23 - node_c 被执行
        2026-06-18 11:32:55.468 | INFO     | __main__:<module>:83 - 第四次执行结果：{'final_res': 'node_c 运行的中间结果'}
            2026-06-18 11:32:55.468 | INFO     | __main__:<module>:85 - ==========-> 完毕 <-==========
```

我们期望在 `node_a` 和 `node_b` 前后中断，此时运行结果和预期相符。

##### 错误用法：恢复调用时不再设置断点
示例如下

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

from loguru import logger

class OverAllState(TypedDict):
    final_res: str

def node_a(state: OverAllState) -> OverAllState:
    logger.info("node_a 被执行")
    return {
        "final_res": "node_a 运行的中间结果"
    }

def node_b(state: OverAllState) -> OverAllState:
    logger.info("node_b 被执行")
    return {
        "final_res": "node_b 运行的中间结果"
    }

def node_c(state: OverAllState) -> OverAllState:
    logger.info("node_c 被执行")
    return {
        "final_res": "node_c 运行的中间结果"
    }


builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_c")
builder.add_edge("node_c", END)

checkpointer = InMemorySaver()
graph = builder.compile(
    checkpointer=checkpointer
)

from IPython.display import display
display(graph)

config = {"configurable": {"thread_id": "123"}}

logger.info("{}-> 第一次执行 <-{}", "=" * 10, "=" * 10)
first_res = graph.invoke(
    {},
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
logger.info("第一次执行结果：{}", first_res)

logger.info("{}-> 第二次执行 <-{}", "=" * 10, "=" * 10)
second_res = graph.invoke(
    None,
    config=config,
    interrupt_before=["node_a", "node_b"],
    interrupt_after=["node_a", "node_b"]
)
logger.info("第二次执行结果：{}", second_res)

logger.info("{}-> 第三次执行 <-{}", "=" * 10, "=" * 10)
third_res = graph.invoke(None, config=config)
logger.info("第三次执行结果：{}", third_res)

logger.info("{}-> 第四次执行 <-{}", "=" * 10, "=" * 10)
final_res = graph.invoke(None, config=config)
logger.info("第四次执行结果：{}", final_res)

logger.info("{}-> 完毕 <-{}", "=" * 10, "=" * 10)
```

运行结果如下

```python
2026-06-18 11:35:16.236 | INFO     | __main__:<module>:49 - ==========-> 第一次执行 <-==========
2026-06-18 11:35:16.259 | INFO     | __main__:<module>:56 - 第一次执行结果：None
2026-06-18 11:35:16.260 | INFO     | __main__:<module>:58 - ==========-> 第二次执行 <-==========
2026-06-18 11:35:16.261 | INFO     | __main__:node_a:11 - node_a 被执行
2026-06-18 11:35:16.263 | INFO     | __main__:<module>:65 - 第二次执行结果：{'final_res': 'node_a 运行的中间结果'}
    2026-06-18 11:35:16.263 | INFO     | __main__:<module>:67 - ==========-> 第三次执行 <-==========
    2026-06-18 11:35:16.264 | INFO     | __main__:node_b:17 - node_b 被执行
    2026-06-18 11:35:16.265 | INFO     | __main__:node_c:23 - node_c 被执行
    2026-06-18 11:35:16.266 | INFO     | __main__:<module>:69 - 第三次执行结果：{'final_res': 'node_c 运行的中间结果'}
        2026-06-18 11:35:16.267 | INFO     | __main__:<module>:71 - ==========-> 第四次执行 <-==========
        2026-06-18 11:35:16.267 | INFO     | __main__:<module>:73 - 第四次执行结果：{'final_res': 'node_c 运行的中间结果'}
            2026-06-18 11:35:16.268 | INFO     | __main__:<module>:75 - ==========-> 完毕 <-==========
```

本例中，我们只在初次调用和第一次恢复运行时设置断点。

由日志可知，按照期望，第二次恢复运行之后，`node_b` 执行完成后还应中断一次，然而，因为本次没有设置断点，因此计算图不再中断，`node_b` 和 `node_c` 都在本次调用完成，计算图结束。

在第四次调用时，计算图已经运行完毕，恢复运行实际上不会执行任何逻辑，只是将计算图的最终状态返回。











# 项目部署
## 本地部署并对接`LangSmith`
> `LangSmith` 是 `LangChain` 官方提供的 `Agent` 应用工程平台，主要用于对基于 `LangGraph`、`LangChain` 等框架开发的 `Agent` 应用进行运行链路追踪、调试、评估和生产监控。
>

`LangChain` 官方提供了开箱即用的 `LangSmith` 云服务，访问地址为：

[https://smith.langchain.com](https://smith.langchain.com/)

此外，`LangSmith` 也支持混合部署和私有化部署。

除了基本的运行链路追踪，`LangSmith` 云服务 还提供了用于调试的 `studio`，要使用这个模块的功能，需要在本地部署 `LangGraph` 项目，这正是本节要完成的任务。

本节将部署一个经典的 `HITL` 应用，并在 `LangSmith` 云服务 的 `studio` 中调试。





### 补充依赖
本章用命令行工具 `langgraph-cli` 部署项目，需要补充相关依赖，如下所示。

在 `requiremtents_full.txt` 中补充依赖

```python
# LangGraph 本地 Agent Server、Studio、热重载
langgraph-cli[inmem]==0.4.30
```

放在

```python
# =========================================================
# LangGraph / Agent 编排
# 作用：构建 Agent、状态图、多步骤工作流、检查点、PostgreSQL 持久化
# =========================================================
```

模块下

然后命令行启用项目的 `conda` 环境，执行

(langgraph) PS C:\Users\Lenovo\OneDrive\文档\AI\langgraph> pip install -r .\requirements_full.txt

即可。

### 准备本地项目
在项目任意位置创建 `hitl_demo` 目录，作为 `LangGraph` 本地项目的根目录。

####  项目结构
项目结构如下所示

```python
hitl_demo/
├── src
├── __init__.py
│   └── agent.py
├── .env
└── langgraph.json
```

其中：

+ `src`：源码包，存放业务代码
    - `__init__.py`：`Python` 包的初始化文件
    - `agent.py`：源码文件，包含业务逻辑，`langgraph-cli` 会根据配置文件读取其中的 `Agent` 或 `Graph` 对象。
+ `.env`：当前项目的环境变量，本次部署时，`langgraph-cli` 会从中加载环境变量。对接 `LangSmith` 云服务 时，需要确保该文件中包含相关的环境变量。
+ `langgraph.json`：项目配置文件，用于管理依赖并定位源码中的 `agent` 对象。

#### 代码清单
##### `__init__.py`
留空即可。

##### `agent.py`
写入以下内容

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt

class OverAllState(TypedDict):
    username: str # 姓名
    age: int # 年龄
    gender: Literal["male", "female"] # 性别

def get_info_node(state: OverAllState) -> OverAllState:
    username = interrupt("请输入您的用户名：")
    age = interrupt("请输入您的年龄：")
    gender = interrupt("请输入您的性别：(male/female)")

    return {
        "username": username,
        "age": age,
        "gender": gender
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("get_info_node", get_info_node)
builder.add_edge(START, "get_info_node")
builder.add_edge("get_info_node", END)

graph = builder.compile()
```

`graph` 对象是我们希望调试的计算图实例。

注意：`langgraph-cli` 启动的本地服务会帮我们管理检查点，所以代码中不要传递检查点存储器，否则启动本地服务时将报错。

##### `.env`
确保该文件中有 `LANGSMITH_API_KEY` 变量

LANGSMITH_API_KEY=xxx

可以根据需求添加所需的环境变量。

如果希望同时启用 `LangSmith` 的追踪功能，需要添加以下环境变量

```python
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_PROJECT="hitl-demo" # 项目名称，自定义
```

配置后，可以在 `LangSmith` 云服务 的以下界面根据项目名称追踪应用运行的详细信息。

##### `langgraph.json`
写入以下内容

```python
{
    "dependencies": ["."],
    "graphs": {
        "graph": "./src/agent.py:graph"
    },
    "env": ".env"
}
```

+ `dependencies`：`langgraph-cli` 启动的本地服务的依赖项，接收类型为列表的值

列表元素只有 `"."` 表示从本次环境中加载依赖。如果在名为 `langchain` 的 `conda` 环境中加载启动 `LangGraph` 本地服务，则会从该环境中加载依赖。

+ `graphs`：计算图或 `Agent` 对象的映射

这是个字典，其中每个键值对表示一条映射关系

    - `Key`：在 `LangSmith Studio` 中显示的应用名称
    - `Value`：相应的应用实例在项目源码中的路径
        * `:` 之前为应用实例所在的代码
        * `:` 之后为应用实例在代码中的变量名
+ `env`：环境变量文件路径

### 启动项目
在命令行执行 

```shell
langgraph dev
```



该命令将会启动一个轻量的本地服务，用于测试。

```python
#报错
UnicodeDecodeError: 'gbk' codec can't decode byte 0xaf in position 60: illegal multibyte sequence
#原因
agent.py 里有中文注释（# 姓名、# 年龄），在 Windows 下 Python 默认用 GBK 编码读取 .py 文件，遇到 GBK 无法解码的字节就报
UnicodeDecodeError。

解决方法

在 PowerShell 中（你平时用的终端），执行前先设置环境变量：

conda activate langgraph
$env:PYTHONUTF8 = "1"
langgraph dev

或者一行搞定：

conda activate langgraph; $env:PYTHONUTF8 = "1"; langgraph dev

永久修复（推荐）

在 conda 的 langgraph 环境中设置该变量，之后 activate 就自动生效：

conda activate langgraph
conda env config vars set PYTHONUTF8=1
conda deactivate
conda activate langgraph

这样以后每次 langgraph dev 都不用再手动设置了。
```

如下所示

```python
(langgraph) PS C:\Users\Lenovo\OneDrive\文档\AI\langgraph\tutorials\hitl_demo> langgraph dev
```

启动日志如下

<font style="color:rgb(51, 51, 51);">如果配置无误，执行命令后通常会自动跳转至</font>

`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">Studio UI</font>**`<font style="color:rgb(51, 51, 51);"> 路径 </font>

[<font style="color:rgb(65, 131, 196);">https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024</font>](https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024)

<font style="color:rgb(51, 51, 51);">如果没有跳转，在浏览器输入链接即可。</font>







**<font style="color:rgb(51, 51, 51);">页面如下所示</font>**

<font style="color:rgb(51, 51, 51);">可以看到计算图的拓扑结构。</font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:rgb(51, 51, 51);"></font>

# <font style="color:rgb(51, 51, 51);">工具调用节点</font>
## <font style="color:rgb(51, 51, 51);">工具节点的实现</font>
工具是 `Agent` 扩展外部能力的重要组件之一。本节以最基础的 `ReAct` 式工具调用循环为例，分别介绍手动实现工具节点和使用预构建 `ToolNode` 两种方式。

手动实现有助于理解工具调用的基本流程：模型生成工具调用请求，工具节点根据名称查找并执行工具，再将执行结果封装成与原工具调用对应的 `ToolMessage`。不过，实际项目还需要处理参数校验、异常处理、运行时注入、并行执行以及 `Command` 传播等问题。

`ToolNode` 对这些通用逻辑进行了封装，适合在需要自定义图结构和工具调用流程的场景中使用。





### <font style="color:rgb(51, 51, 51);">手动处理工具调用</font>
**示例如下**

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langchain.messages import HumanMessage, ToolMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from loguru import logger
from dotenv import load_dotenv
load_dotenv(override=True)

@tool(parse_docstring=True)
def get_weather(city: str) -> str:
    """
    查询指定城市的当日天气

    Args:
        city: 城市名称
    """
    return f"{city} 今天天气不错"

@tool(parse_docstring=True)
def get_news(home_or_abroad: bool) -> str:
    """
    查询国内外新闻

    Args:
        home_or_abroad: 查询国内还是国外新闻，True: 国内新闻，False：国外新闻
    """
    if home_or_abroad:
        return "Kimi 新模型发布"
    return "Anthropic 暂停新模型访问"

tools_by_name = {
    "get_weather": get_weather,
    "get_news": get_news
}
tools = [get_weather, get_news]

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)
model_with_tools = model.bind_tools(tools=tools)

def llm_node(state: MessagesState) -> MessagesState:
    messages = state['messages']
    response = model_with_tools.invoke(messages)

    return {
        "messages": [response]
    }

def tool_node(state: MessagesState) -> MessagesState:
    last_msg = state['messages'][-1]
    if not last_msg.tool_calls:
        return {}

    tool_msgs = []
    for tool_call in last_msg.tool_calls:
        tool = tools_by_name[tool_call["name"]]
        logger.info("工具 {} 被调用, 对应的 tool_call: {}", tool_call["name"], tool_call)
        tool_res = tool.invoke(tool_call["args"])
        tool_msg = ToolMessage(
            name = tool_call["name"],
            content = tool_res,
            tool_call_id = tool_call["id"]
        )
        tool_msgs.append(tool_msg)

    return {
        "messages": tool_msgs
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    if state['messages'][-1].tool_calls:
        return "tool_node"
    return END

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", tool_node)
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

graph = builder.compile()

from IPython.display import display
display(graph)

res = graph.invoke({"messages": [HumanMessage("今天北京天气如何？国内有哪些新闻")]})
for msg in res['messages']:
    msg.pretty_print()
```

> 注意
>
> 上面的手动实现会按照 `tool_calls` 的顺序串行执行工具，主要用于展示底层流程。`ToolNode` 在同步调用中可通过执行器并行处理多个工具调用，在异步调用中则使用 `asyncio.gather()` 并发执行。
>



**运行结果如下**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792738973341-caceaff3-ba42-4fe5-a015-c1111f74318d.png)

```json
2026-06-22 09:59:05.258 | INFO     | __main__:tool_node:67 - 工具 get_weather 被调用, 对应的 tool_call: {'name': 'get_weather', 'args': {'city': '北京'}, 'id': 'call_00_lpemFmfsKxTtEKjweosB0896', 'type': 'tool_call'}
2026-06-22 09:59:05.259 | INFO     | __main__:tool_node:67 - 工具 get_news 被调用, 对应的 tool_call: {'name': 'get_news', 'args': {'home_or_abroad': True}, 'id': 'call_01_DDLZCKnovq6EtdrD6XXw2202', 'type': 'tool_call'}
================================ Human Message =================================

今天北京天气如何？国内有哪些新闻
================================== Ai Message ==================================

好的，我来查询今天北京的天气和国内新闻。
Tool Calls:
  get_weather (call_00_lpemFmfsKxTtEKjweosB0896)
 Call ID: call_00_lpemFmfsKxTtEKjweosB0896
  Args:
    city: 北京
  get_news (call_01_DDLZCKnovq6EtdrD6XXw2202)
 Call ID: call_01_DDLZCKnovq6EtdrD6XXw2202
  Args:
    home_or_abroad: True
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================= Tool Message =================================
Name: get_news

Kimi 新模型发布
================================== Ai Message ==================================

以下是今天的查询结果：

### 🌤️ 北京天气
今天北京的天气**不错**，适合外出活动哦！

### 📰 国内新闻
目前有一条重要新闻：**Kimi 新模型发布**

如果你想了解更多详情，可以继续问我！
```

### <font style="color:rgb(51, 51, 51);">用</font>`<font style="color:rgb(51, 51, 51);">ToolNode</font>`<font style="color:rgb(51, 51, 51);">处理工具调用</font>
上一节的工具节点完成了以下工作：读取最后一条 `AIMessage` 中的 `tool_calls`，根据工具名称查找并执行对应工具，**<font style="color:#DF2A3F;">再把执行结果封装成与原工具调用一一对应的 </font>**`**<font style="color:#DF2A3F;">ToolMessage</font>**`。

这种实现适合帮助我们理解工具调用的基本流程，但它只覆盖了最简单的场景。实际项目还需要处理无效工具名、参数校验、异常处理、运行时参数注入、`Command` 传播以及多个工具调用的并行执行等问题。

`LangGraph` 提供了预构建的 `ToolNode`。只需传入工具列表，它便可以完成上述所有操作。



**示例如下**

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt.tool_node import ToolNode
from langgraph.graph.message import MessagesState
from langchain.messages import HumanMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv
load_dotenv(override=True)

@tool(parse_docstring=True)
def get_weather(city: str) -> str:
    """
    查询指定城市的当日天气

    Args:
        city: 城市名称
    """
    return f"{city} 今天天气不错"

@tool(parse_docstring=True)
def get_news(home_or_abroad: bool) -> str:
    """
    查询国内外新闻

    Args:
        home_or_abroad: 查询国内还是国外新闻，True: 国内新闻，False：国外新闻
    """
    if home_or_abroad:
        return "Kimi 新模型发布"
    return "Anthropic 暂停新模型访问"

tools = [get_weather, get_news]

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)
model_with_tools = model.bind_tools(tools=tools)

def llm_node(state: MessagesState) -> MessagesState:
    messages = state['messages']
    response = model_with_tools.invoke(messages)

    return {
        "messages": [response]
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    if state['messages'][-1].tool_calls:
        return "tool_node"
    return END

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", ToolNode(tools=tools))
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

graph = builder.compile()

from IPython.display import display
display(graph)

res = graph.invoke({"messages": [HumanMessage("今天北京天气如何？国内有哪些新闻？")]})
for msg in res['messages']:
    msg.pretty_print()
```

**运行结果如下**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792739018321-8a9e4be3-c66b-4b88-ad0f-d33b3b926702.png)

```json
================================ Human Message =================================

今天北京天气如何？国内有哪些新闻？
================================== Ai Message ==================================

好的，我来同时查询北京天气和国内新闻。
Tool Calls:
  get_weather (call_00_BBHYjTwSiWbGolSrp7yb9812)
 Call ID: call_00_BBHYjTwSiWbGolSrp7yb9812
  Args:
    city: 北京
  get_news (call_01_3XEu887comx2s1SMtw8h3282)
 Call ID: call_01_3XEu887comx2s1SMtw8h3282
  Args:
    home_or_abroad: True
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================= Tool Message =================================
Name: get_news

Kimi 新模型发布
================================== Ai Message ==================================

以下是您查询的信息：

---

### 🌤️ 北京今日天气
北京今天**天气不错**，适合外出活动。

### 📰 国内新闻
今日国内有一条重要新闻：**Kimi 新模型发布**。

如果您想了解更多详细内容，随时可以继续问我！
```

## <font style="color:rgb(51, 51, 51);">进阶用法</font>
### `ToolRuntime`介绍
`ToolRuntime` 是专门面向工具调用的运行时对象。

按照官方规范：

+ 工具函数中存在名为 `runtime`
+ 并且类型标注为 `ToolRuntime` 的参数时

底层运行时会在调用工具前自动注入**<font style="color:#DF2A3F;"> </font>**`**<font style="color:#DF2A3F;">ToolRuntime</font>**`**<font style="color:#DF2A3F;"> 实例。</font>**根据源码实现，某些不规范的写法也能注入运行时实例，但并不规范，不推荐。

需要注意，`ToolRuntime` 与图节点中使用的 `langgraph.runtime.Runtime` 并不是同一个类型。前者额外提供了当前图状态、运行配置和工具调用 ID 等工具调用专属信息。

当前版本，`ToolRuntime` 的核心定义如下：

```python
@dataclass
class ToolRuntime(_DirectlyInjectedToolArg, Generic[ContextT, StateT]):
    state: StateT
    context: ContextT
    config: RunnableConfig
    stream_writer: StreamWriter
    tool_call_id: str | None
    store: BaseStore | None
```

+ `**<font style="color:#DF2A3F;">state</font>**`**<font style="color:#DF2A3F;">：图状态，短期记忆</font>**
+ `**<font style="color:#DF2A3F;">context</font>**`**<font style="color:#DF2A3F;">：运行时上下文</font>**
+ `**<font style="color:#DF2A3F;">config</font>**`**<font style="color:#DF2A3F;">：运行时配置，包含元数据</font>**
+ `**<font style="color:#DF2A3F;">stream_writer</font>**`**<font style="color:#DF2A3F;">：自定义流式输出写入器</font>**
+ `**<font style="color:#DF2A3F;">tool_call_id</font>**`**<font style="color:#DF2A3F;">：工具调用 </font>**`**<font style="color:#DF2A3F;">ID</font>**`
+ `**<font style="color:#DF2A3F;">store</font>**`**<font style="color:#DF2A3F;">：长期记忆</font>**

**<font style="color:#DF2A3F;"></font>**

借助 `runtime`，工具可以访问上述所有资源。





### 在工具中更新状态
+ 工具不仅可以返回普通结果，还可以返回 `Command(update=...)`，将业务结果写入图状态。
+ 当工具由模型发起调用时，消息历史中的 `AIMessage.tool_calls` 必须紧跟与之匹配的 `ToolMessage`。
+ 当工具返回普通结果时，`ToolNode` 会帮我们完成运行结果到 `ToolMessage` 的转换，并追加到消息列表。
+ 但工具返回 `Command` 时，上述操作需要开发者完成，此时还应在 `Command.update` 的消息字段中写入对应的 `ToolMessage`。
+ 如果同一轮并行执行的多个工具可能更新同一个普通状态字段，需要为该字段配置合适的归并函数；否则可能出现并发更新冲突。本例中的两个工具分别更新 `weather_res` 和 `news_res`，因此不存在该问题。





```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt.tool_node import ToolNode, ToolRuntime
from langgraph.types import Command
from langgraph.graph.message import MessagesState

from langchain.messages import HumanMessage, ToolMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv
load_dotenv(override=True)

class OverAllState(MessagesState):
    weather_res: str
    news_res: str

@tool(parse_docstring=True)
def get_weather(city: str, runtime: ToolRuntime) -> Command:
    """
    查询指定城市的当日天气

    Args:
        city: 城市名称
    """
    res = f"{city} 今天天气不错"
    tool_call_id = runtime.tool_call_id
    tool_msg = ToolMessage(tool_call_id=tool_call_id, content=res)
    return Command(
        update = {
            "weather_res": res,
            "messages": [tool_msg]
        }
    )

@tool(parse_docstring=True)
def get_news(home_or_abroad: bool, runtime: ToolRuntime) -> Command:
    """
    查询国内外新闻

    Args:
        home_or_abroad: 查询国内还是国外新闻，True: 国内新闻，False：国外新闻
    """
    if home_or_abroad:
        res = "Kimi 新模型发布"
    else:
        res = "Anthropic 暂停新模型访问"
    tool_call_id = runtime.tool_call_id
    tool_msg = ToolMessage(tool_call_id=tool_call_id, content=res)
    return Command(
        update = {
            "news_res": res,
            "messages": [tool_msg]
        }
    )

tools = [get_weather, get_news]

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)
model_with_tools = model.bind_tools(tools=tools)

def llm_node(state: OverAllState) -> OverAllState:
    messages = state['messages']
    response = model_with_tools.invoke(messages)

    return {
        "messages": [response]
    }

def router(state: OverAllState) -> Literal["tool_node", END]:
    if state['messages'][-1].tool_calls:
        return "tool_node"
    return END

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", ToolNode(tools=tools))
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

graph = builder.compile()

from IPython.display import display
display(graph)

res = graph.invoke({"messages": [HumanMessage("今天北京天气如何？国内有哪些新闻？")]})
print('=' * 30, '-> messages <-', '=' * 30)
for msg in res.pop('messages'):
    msg.pretty_print()

print('=' * 30, '-> res without messages <-', '=' * 30)
print(res)
```

运行结果如下

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792739246030-23bbe0e0-8467-4547-8268-9f516e729942.png)





<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260622104243639.png)

```json
============================== -> messages <- ==============================
================================ Human Message =================================

今天北京天气如何？国内有哪些新闻？
================================== Ai Message ==================================

好的，我来同时查询北京天气和国内新闻。
Tool Calls:
  get_weather (call_00_hbEppxygz7DLDwaxg8X19090)
 Call ID: call_00_hbEppxygz7DLDwaxg8X19090
  Args:
    city: 北京
  get_news (call_01_wkG4j5Hjq4wkyTOaxzxT5267)
 Call ID: call_01_wkG4j5Hjq4wkyTOaxzxT5267
  Args:
    home_or_abroad: True
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================= Tool Message =================================
Name: get_news

Kimi 新模型发布
================================== Ai Message ==================================

为您查询到以下信息：

### 🌤️ 北京今日天气
今天北京的天气**不错**，比较适宜出行。

### 📰 国内新闻
- **Kimi 新模型发布** — 国内AI领域又有新动态。

如果您想了解更多详情，可以继续问我！
============================== -> res without messages <- ==============================
{'weather_res': '北京 今天天气不错', 'news_res': 'Kimi 新模型发布'}
```



### 工具节点执行与容错机制
+ `ToolNode` 提供同步工具调用包装器 `**<font style="color:#DF2A3F;background-color:#FBDE28;">wrap_tool_call</font>**`**<font style="color:#DF2A3F;background-color:#FBDE28;">，用于在工具执行前后插入自定义逻辑</font>**。
+ 它与 `LangChain Agent` 中间件 `wrap_tool_call` 的核心机制相同：接收当前工具调用请求 `request` 和真正执行工具的回调 `execute`，可以选择不调用、调用一次或多次 `execute(request)`，并最终返回 `ToolMessage` 或 `Command`。
+ 因此，`wrap_tool_call` 可以用于重试、缓存、请求改写、短路返回和自定义控制流等场景。异步工具链路还可以使用对应的 `awrap_tool_call`。

此处通过重试和缓存机制的实现，介绍 `wrap_tool_call` 的用法。

#### 实现重试机制
示例如下

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt.tool_node import ToolNode
from langgraph.graph.message import MessagesState

from langchain.messages import HumanMessage, ToolMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from dataclasses import dataclass
from loguru import logger
from dotenv import load_dotenv
load_dotenv(override=True)

import random

@tool(parse_docstring=True)
def get_weather(city: str) -> str:
    """
    查询指定城市的当日天气

    Args:
        city: 城市名称
    """
    # 70% 概率因网络波动而调用失败
    rand_int = random.randint(1,10)
    if rand_int < 8:
        raise ConnectionError("网络波动失败")
    return f"{city} 今天天气不错"

tools = [get_weather]

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)
model_with_tools = model.bind_tools(tools=tools)

@dataclass
class UserContext:
    max_attempts: int

def llm_node(state: MessagesState) -> MessagesState:
    messages = state['messages']
    response = model_with_tools.invoke(messages)

    return {
        "messages": [response]
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    if state['messages'][-1].tool_calls:
        return "tool_node"
    return END

def wrap_tool_call(request, execute):
    max_attempts = request.runtime.context.max_attempts
    tool_call_id = request.runtime.tool_call_id

    tool_msg = ""
    for i in range(max_attempts):
        try:
            tool_msg = execute(request)
            break
        except ConnectionError as e:
            logger.info("工具调用失败，当前调用次数: {}, 调用次数上限: {}, 异常信息: {}", i + 1, max_attempts, e)

    if not tool_msg:
        tool_msg = ToolMessage(
            tool_call_id = tool_call_id,
            content = "调用次数到达上限，调用失败"
        )

    return tool_msg

builder = StateGraph(state_schema=MessagesState, context_schema=UserContext)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", ToolNode(tools=tools, wrap_tool_call=wrap_tool_call))
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

graph = builder.compile()

from IPython.display import display
display(graph)

res = graph.invoke(
    {"messages": [HumanMessage("今天北京天气如何？")]},
    context = UserContext(max_attempts=3))
for msg in res['messages']:
    msg.pretty_print()
```

运行结果如下

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792739318150-a70965e7-6cf2-4059-80fe-67df48884e31.png)

按照当前设计，每次工具调用有 `70%` 的概率失败，最多尝试 `3` 次。仍然可能最终失败，如下所示。

```json
2026-06-22 11:12:21.039 | INFO     | __main__:wrap_tool_call:71 - 工具调用失败，当前调用次数: 1, 调用次数上限: 3, 异常信息: 网络波动失败
2026-06-22 11:12:21.042 | INFO     | __main__:wrap_tool_call:71 - 工具调用失败，当前调用次数: 1, 调用次数上限: 3, 异常信息: 网络波动失败
2026-06-22 11:12:21.046 | INFO     | __main__:wrap_tool_call:71 - 工具调用失败，当前调用次数: 1, 调用次数上限: 3, 异常信息: 网络波动失败
================================ Human Message =================================

今天北京天气如何？
================================== Ai Message ==================================

好的，我来查询一下北京今天的天气情况。
Tool Calls:
  get_weather (call_00_iYUeyQlDB0NuAAkcU1Zy3110)
 Call ID: call_00_iYUeyQlDB0NuAAkcU1Zy3110
  Args:
    city: 北京
================================= Tool Message =================================

调用次数到达上限，调用失败
================================== Ai Message ==================================

抱歉，查询天气的服务暂时遇到了问题，无法获取到北京今天的天气信息。可能是以下原因导致的：

1. **服务暂时不可用** - 可能是天气数据接口出现了临时故障
2. **网络问题** - 查询过程中网络连接不稳定

建议您可以：
- **稍后再试**，看看服务是否恢复
- 通过手机上的天气App或浏览器搜索"北京天气"来获取最新信息

非常抱歉给您带来不便！如果还有其他问题需要帮助，请随时告诉我。
```

也有可能最终成功，如下

```json
2026-06-22 14:03:22.371 | INFO     | __main__:wrap_tool_call:70 - 工具调用失败，当前调用次数: 1, 调用次数上限: 3, 异常信息: 网络波动失败
================================ Human Message =================================

今天北京天气如何？
================================== Ai Message ==================================

好的，我来帮你查询一下今天北京的天气情况。
Tool Calls:
  get_weather (call_00_IXWZ2mg3lxI09gzm9BbZ4314)
 Call ID: call_00_IXWZ2mg3lxI09gzm9BbZ4314
  Args:
    city: 北京
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================== Ai Message ==================================

今天北京的天气**不错**，是个适合外出活动的好日子！☀️

如果你需要更详细的天气信息（比如具体温度、风向等），可以告诉我，我再帮你进一步查询。
```



#### 实现缓存机制
示例如下

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt.tool_node import ToolNode
from langgraph.graph.message import MessagesState

from langchain.messages import HumanMessage, ToolMessage
from langchain.tools import tool
from langchain_deepseek import ChatDeepSeek

from loguru import logger
import json
from dotenv import load_dotenv
load_dotenv(override=True)

@tool(parse_docstring=True)
def get_weather(city: str) -> str:
    """
    查询指定城市的当日天气

    Args:
        city: 城市名称
    """
    return f"{city} 今天天气不错"

tools = [get_weather]

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)
model_with_tools = model.bind_tools(tools=tools)

def llm_node(state: MessagesState) -> MessagesState:
    messages = state['messages']
    response = model_with_tools.invoke(messages)

    return {
        "messages": [response]
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    if state['messages'][-1].tool_calls:
        return "tool_node"
    return END

global_cache = dict()
def wrap_tool_call(request, execute):
    tool_name = request.tool_call["name"]
    tool_args = json.dumps(request.tool_call["args"])
    tool_call_id = request.runtime.tool_call_id

    cache_key = (tool_name, tool_args)
    cache = global_cache.get(cache_key)

    if cache:
        logger.info("{} 调用命中缓存", tool_name)
        tool_msg = ToolMessage(
            tool_call_id = tool_call_id,
            content = cache
        )
    else:
        tool_msg = execute(request)
        logger.info("将 {} 的调用结果写入缓存", tool_name)
        global_cache[cache_key] = tool_msg.content

    return tool_msg

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", ToolNode(tools=tools, wrap_tool_call=wrap_tool_call))
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

graph = builder.compile()

from IPython.display import display
display(graph)

print('=' * 30, '-> 第一次调用 - 北京 <-', '=' * 30)
res = graph.invoke({"messages": [HumanMessage("今天北京天气如何？")]})
for msg in res['messages']:
    msg.pretty_print()

print('=' * 30, '-> 第二次调用 - 北京 <-', '=' * 30)
res = graph.invoke({"messages": [HumanMessage("今天北京天气如何？")]})
for msg in res['messages']:
    msg.pretty_print()

print('=' * 30, '-> 第三次调用 - 杭州 <-', '=' * 30)
res = graph.invoke({"messages": [HumanMessage("今天杭州天气如何？")]})
for msg in res['messages']:
    msg.pretty_print()
```

> 注意
>
> 这里使用的 `global_cache` 只是进程内缓存，主要用于演示：它没有 `TTL`、容量限制、并发保护和持久化能力，也只缓存普通 `ToolMessage` 的内容。生产环境应根据数据时效性选择 `Redis` 等缓存系统，并把用户、权限、地域、工具版本等会影响结果的因素纳入缓存键；具有副作用的工具通常不应直接缓存。
>

  
** 运行结果如下**

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/51057965/1792739365373-1db7cd46-0656-465d-a014-5896a9f07256.png)

```python
============================== -> 第一次调用 - 北京 <- ==============================
2026-06-22 14:05:29.845 | INFO     | __main__:wrap_tool_call:67 - 将 get_weather 的调用结果写入缓存
================================ Human Message =================================

今天北京天气如何？
================================== Ai Message ==================================

好的，我来查询一下北京今天的天气情况。
Tool Calls:
  get_weather (call_00_RWpYgZGeskrexH2dOWol2713)
 Call ID: call_00_RWpYgZGeskrexH2dOWol2713
  Args:
    city: 北京
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================== Ai Message ==================================

北京今天天气不错哦！☀️ 是个适合外出活动的好天气。不过具体温度、风力等详细信息我就没法提供了，建议您出门前可以根据自己的体感适当增减衣物。希望您度过愉快的一天！😊
============================== -> 第二次调用 - 北京 <- ==============================
2026-06-22 14:05:32.277 | INFO     | __main__:wrap_tool_call:60 - get_weather 调用命中缓存
================================ Human Message =================================

今天北京天气如何？
================================== Ai Message ==================================

好的，我来查询一下北京今天的天气情况。
Tool Calls:
  get_weather (call_00_Kimj3ySGhtIFhuYjIuw67098)
 Call ID: call_00_Kimj3ySGhtIFhuYjIuw67098
  Args:
    city: 北京
================================= Tool Message =================================

北京 今天天气不错
================================== Ai Message ==================================

北京今天天气不错！☀️ 是个适合出门的好天气。不过具体的温度和风力等详细信息没有提供，如果你需要了解更多细节，可以告诉我哦~ 😊
============================== -> 第三次调用 - 杭州 <- ==============================
2026-06-22 14:05:34.514 | INFO     | __main__:wrap_tool_call:67 - 将 get_weather 的调用结果写入缓存
================================ Human Message =================================

今天杭州天气如何？
================================== Ai Message ==================================

好的，我来查询一下杭州今天的天气情况。
Tool Calls:
  get_weather (call_00_KvnVrq2WNq1C9I0xKb3U7776)
 Call ID: call_00_KvnVrq2WNq1C9I0xKb3U7776
  Args:
    city: 杭州
================================= Tool Message =================================
Name: get_weather

杭州 今天天气不错
================================== Ai Message ==================================

杭州今天天气不错！☀️ 看起来是一个适合出行或者户外活动的好日子。如果你有出门的计划，可以放心安排哦～不过如果需要更具体的温度、风力等信息，建议可以再查一下实时的天气预报。😊
```

  
 

 

# 流式执行
##  概述
### 什么是流式执行
**流式执行（**`Streaming Execution`**）**，是指程序在任务尚未全部完成时，就将执行过程中已经产生的中间结果、状态变化或事件持续输出给调用方，而不是等待整个任务结束后一次性返回最终结果。

简而言之：流式执行就是

> 边执行、边产生事件、边向调用方返回结果。
>

### 什么是`LangGraph`的流式执行
`LangGraph` 的流式执行是指，状态图计算过程中，将节点输出、状态更新、消息增量、自定义事件或调试信息等写入流式队列，并在特定的时机将流式队列中的信息返还给调用者。

在同步执行中，这些数据最终进入内部的**同步流式队列**；在异步执行中，则进入对应的**异步队列**。调用方通过**迭代器**逐条消费这些数据，因此不必等到整张图执行完毕后再获得反馈。

### `LangGraph`的流式执行`API`
`LangGraph` 提供了两套流式执行 `API`

#### `stream/astream`
获取 `LangGraph` 执行过程中的业务数据或运行时数据

+ `stream`：同步 `API`
+ `astream`：异步 `API`

调用方通过 `stream_mode` 选择需要消费的内容，例如状态、节点更新、模型消息、自定义数据、检查点或任务事件。这是本章的重点。

`stream`**与**`astream`**的区别**

| 维度 | `stream()` | `astream()` |
| :--- | :--- | :--- |
| 调用方式 | 同步迭代 `for chunk in graph.stream(...)` | 异步迭代 `async for chunk in graph.astream(...)` |
| 运行环境 | 普通 Python 脚本 | `asyncio` 事件循环（Jupyter、FastAPI 等） |
| 参数与输出 | 相同 `stream_mode`，语义一致 | 相同 `stream_mode`，语义一致 |
| 适用场景 | 同步批处理、CLI 工具 | `async` 服务端、结合其他异步 IO |


#### `astream_events`
获取图运行过程中产生的 `Runnable` 标准事件（组件生命周期、父子调用关系、输入输出等）。详见 [11.3. ](#113-astream_events)`astream_events`。

##  `stream/astream`
`astream` 是 `stream` 的异步版本。二者使用相同的流模式，且在参数一致时，输出数据的语义和封装格式一致；区别主要在于调用方分别使用同步迭代和异步迭代进行消费。

前面的章节我们都是通过 `invoke` 调用状态图。实际上，`invoke` 会在内部消费 `stream`，并把流式执行结果汇总为最终返回值；`ainvoke` 与 `astream` 的关系同理。

### 输出格式版本说明
从 `LangGraph 1.1` 开始，`stream/astream` 支持两种输出格式版本：

+ `v1`：当前默认格式。输出形态会随参数变化：
    - 单个流模式通常直接返回该模式的数据；
    - 多个流模式返回 `(mode, data)`；
    - 启用子图流式输出后，还会增加命名空间信息。
+ `v2`：在 `v1` 基础上做了封装，统一返回 `StreamPart` 字典，固定包含 `type`、`ns` 和 `data` 三个字段，增强了输出的可读性，简化了解析难度、并补充了校验机制。

本节基于 `v1` 展开。示例刻意把 `stream_mode` 写成列表，即使每次只指定一种模式，也会得到 `(mode, data)` 二元组，便于观测模式名称，并且可以无缝扩展到指定多种模式的情况。

### `stream_mode`
流式输出支持不同的输出模式 `stream_mode`，这规定了流式输出的内容（在 `LangGraph` 中称为 `chunk`）。`stream_mode` 可传入单个字符串或模式列表。

| 模式 | 输出内容 | 适用场景 |
| :--- | :--- | :--- |
| `values` | 每个超步后的完整状态 | 需要完整状态快照 |
| `updates` | 节点产生的状态更新（增量） | 只关心节点输出变化 |
| `messages` | `messages` 状态字段的增量更新 | LLM 对话流式输出 |
| `checkpoints` | 检查点更新事件（需检查点存储器） | 持久化监控、中断调试 |
| `tasks` | 任务开始/结果事件（含触发通道、异常） | 运行时观测、任务追踪 |
| `debug` | `checkpoints` + `tasks` 的统一封装，附加超步编号、时间戳 | 调试排错 |
| `custom` | 节点/工具通过 `stream_writer` 主动写出的自定义数据 | 进度通知、阶段说明 |


为便于观察每种模式的数据结构，本节每次只启用一种模式，但仍使用列表形式传参（得到 `(mode, data)` 二元组）。

#### `values`
**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class OverAllState(TypedDict):
    initial_state: str
    node_a_output: str
    node_b_output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "node_a_output": "节点A的输出"
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
for chunk in graph.stream(
    {"initial_state": "初始状态"},
    stream_mode=["values"],
):
    print(chunk)
```

**运行结果如下**

```json
('values', {'initial_state': '初始状态'})
('values', {'initial_state': '初始状态', 'node_a_output': '节点A的输出'})
('values', {'initial_state': '初始状态', 'node_a_output': '节点A的输出', 'node_b_output': '节点B的输出'})
```

#### `updates`
**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class OverAllState(TypedDict):
    initial_state: str
    node_a_output: str
    node_b_output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "node_a_output": "节点A的输出"
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
for chunk in graph.stream(
    {"initial_state": "初始状态"},
    stream_mode=["updates"],
):
    print(chunk)
```

**运行结果如下**

```json
('updates', {'node_a': {'node_a_output': '节点A的输出'}})
('updates', {'node_b': {'node_b_output': '节点B的输出'}})
```

#### `messages`
**示例如下**

```python
from langgraph.graph import StateGraph, START, END, MessagesState

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

def llm_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    response = model.invoke(messages)

    return {
        "messages": [response],
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", END)

graph = builder.compile()

for chunk in graph.stream(
        {
            "messages":[HumanMessage(content="你好!")]
        },
    stream_mode=["values","messages"],
):
    print(chunk)

```

**运行结果如下**

```json
('values', {'messages': [HumanMessage(content='你好!', additional_kwargs={}, response_metadata={}, id='0f0ff522-bc84-4ead-8228-66221903e9b4')]})
('messages', (AIMessageChunk(content='', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='你好', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='！', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='很高兴', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='见到', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='你', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='，', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='有什么', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='可以', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='帮', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='你的', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='吗', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='？', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='无论是', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='聊天', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='、', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='解答', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='问题', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='，', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='还是', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='需要', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='帮助', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='，', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='都可以', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='告诉我', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='哦', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='！', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='😊', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='', additional_kwargs={}, response_metadata={'finish_reason': 'stop', 'model_name': 'deepseek-v4-flash', 'system_fingerprint': 'fp_8b330d02d0_prod0820_fp8_kvcache_20260402', 'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], usage_metadata={'input_tokens': 6, 'output_tokens': 28, 'total_tokens': 34, 'input_token_details': {'cache_read': 0}, 'output_token_details': {}}, tool_call_chunks=[]), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('messages', (AIMessageChunk(content='', additional_kwargs={}, response_metadata={}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[], chunk_position='last'), {'langgraph_step': 1, 'langgraph_node': 'llm_node', 'langgraph_triggers': ('branch:to:llm_node',), 'langgraph_path': ('__pregel_pull', 'llm_node'), 'langgraph_checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'checkpoint_ns': 'llm_node:e0f03332-6641-0bb0-cf08-e99ffeb6c7a6', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
('values', {'messages': [HumanMessage(content='你好!', additional_kwargs={}, response_metadata={}, id='0f0ff522-bc84-4ead-8228-66221903e9b4'), AIMessage(content='你好！很高兴见到你，有什么可以帮你的吗？无论是聊天、解答问题，还是需要帮助，都可以告诉我哦！😊', additional_kwargs={}, response_metadata={'finish_reason': 'stop', 'model_name': 'deepseek-v4-flash', 'system_fingerprint': 'fp_8b330d02d0_prod0820_fp8_kvcache_20260402', 'model_provider': 'deepseek'}, id='lc_run--019f5a6a-64cf-7573-874a-e669e00000ba', tool_calls=[], invalid_tool_calls=[], usage_metadata={'input_tokens': 6, 'output_tokens': 28, 'total_tokens': 34, 'input_token_details': {'cache_read': 0}, 'output_token_details': {}})]})
```

#### `checkpoints`
`checkpoints` 模式依赖检查点存储器。它输出的是运行时构造的检查点事件，其结构接近 `get_state()` 返回的状态快照.

**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

from langgraph.types import interrupt, Command

class OverAllState(TypedDict):
    initial_state: str
    parallel_node_a_1: str
    parallel_node_a_2: str
    node_b_output: str

def parallel_node_a_1(state: OverAllState) -> OverAllState:
    return {
        "parallel_node_a_1": "并行节点A-1的输出"
    }

def parallel_node_a_2(state: OverAllState) -> OverAllState:
    return {
        "parallel_node_a_2": "并行节点A-2的输出"
    }

def node_b(state: OverAllState) -> OverAllState:
    interrupt("hello")
    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("parallel_node_a_1", parallel_node_a_1)
builder.add_node("parallel_node_a_2", parallel_node_a_2)
builder.add_node("node_b", node_b)
builder.add_edge(START, "parallel_node_a_1")
builder.add_edge(START, "parallel_node_a_2")
builder.add_edge(["parallel_node_a_1", "parallel_node_a_2"], "node_b")
builder.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "123"}}
for chunk in graph.stream(
    {"initial_state": "初始状态"},
    stream_mode=["checkpoints"],
    config=config
):
    print(chunk)

print('=' * 30, '-> 中断前后分界线 <-', '=' * 30)

for chunk in graph.stream(
    Command(resume=""),
    stream_mode=["checkpoints"],
    config=config
):
    print(chunk)
```

**运行结果如下**

```python
# === 第 1 个检查点：input 阶段（step=-1），仅有 __start__ 任务 ===
(
    'checkpoints',
    {
        'config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a431-6009-bfff-dd033caf8a09'
            }
        },
        'parent_config': None,
        'values': {},
        'metadata': {
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        'next': ['__start__'],
        'tasks': [
            {
                'id': '6f4add75-8430-fd5b-46d5-93f40b5f142a',
                'name': '__start__',
                'interrupts': (),
                'state': None
            }
        ]
    }
)

# === 第 2 个检查点：loop 阶段（step=0），两个并行节点待执行 ===
(
    'checkpoints',
    {
        'config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a436-66f0-8000-61f51460d18e'
            }
        },
        'parent_config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a431-6009-bfff-dd033caf8a09'
            }
        },
        'values': {
            'initial_state': '初始状态'
        },
        'metadata': {
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        'next': ['parallel_node_a_1', 'parallel_node_a_2'],
        'tasks': [
            {
                'id': '0aa4a4d7-1e7f-f02f-25ef-46c088c7e1e0',
                'name': 'parallel_node_a_1',
                'interrupts': (),
                'state': None
            },
            {
                'id': 'd3670671-0bdb-6574-be1e-915be93b239e',
                'name': 'parallel_node_a_2',
                'interrupts': (),
                'state': None
            }
        ]
    }
)

# === 第 3 个检查点：loop 阶段（step=1），并行节点执行完毕，node_b 待执行 ===
(
    'checkpoints',
    {
        'config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a439-67c6-8001-f441fbddc2f9'
            }
        },
        'parent_config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a436-66f0-8000-61f51460d18e'
            }
        },
        'values': {
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出'
        },
        'metadata': {
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        'next': ['node_b'],
        'tasks': [
            {
                'id': 'cc97ac77-de6f-470b-d3c5-d0a94ea5534e',
                'name': 'node_b',
                'interrupts': (),
                'state': None
            }
        ]
    }
)

============================== -> 中断前后分界线 <- ==============================

# === 第 4 个检查点：中断发生时（step=1），node_b 的 interrupts 字段包含中断信息 ===
(
    'checkpoints',
    {
        'config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a439-67c6-8001-f441fbddc2f9'
            }
        },
        'parent_config': {
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f17e909-a436-66f0-8000-61f51460d18e'
            }
        },
        'values': {
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出'
        },
        'metadata': {
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        'next': ['node_b'],
        'tasks': [
            {
                'id': 'cc97ac77-de6f-470b-d3c5-d0a94ea5534e',
                'name': 'node_b',
                'interrupts': (
                    {
                        'value': 'hello',
                        'id': '486f817f7367de26620067969c8780eb'
                    },
                ),
                'state': None
            }
        ]
    }
)

# === 第 5 个检查点：resume 后（step=2），node_b 执行完毕，next 为空 ===
(
    'checkpoints',
    {
        'config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e926-5137-6203-8002-4e1f7ffb6aa1'
            }
        },
        'parent_config': {
            'configurable': {
                'checkpoint_ns': '',
                'thread_id': '123',
                'checkpoint_id': '1f17e909-a439-67c6-8001-f441fbddc2f9'
            }
        },
        'values': {
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出',
            'node_b_output': '节点B的输出'
        },
        'metadata': {
            'source': 'loop',
            'step': 2,
            'parents': {}
        },
        'next': [],
        'tasks': []
    }
)
```

**查看历史检查点列表**

```python
list(graph.get_state_history(config=config))
```

**运行结果如下**

```json
[
    StateSnapshot(
        values={
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出',
            'node_b_output': '节点B的输出'
        },
        next=(),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f230-641b-8002-53e494a0e53c'
            }
        },
        metadata={
            'source': 'loop',
            'step': 2,
            'parents': {}
        },
        created_at='2026-06-23T04:00:50.062223+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f229-6a9b-8001-4049f4629aea'
            }
        },
        tasks=(),
        interrupts=()
    ),

    StateSnapshot(
        values={
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出'
        },
        next=(
            'node_b',
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f229-6a9b-8001-4049f4629aea'
            }
        },
        metadata={
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        created_at='2026-06-23T04:00:50.059522+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f225-646d-8000-b7e7c665f75a'
            }
        },
        tasks=(
            PregelTask(
                id='f2153246-63f8-7422-31f6-2712d781be78',
                name='node_b',
                path=(
                    '__pregel_pull',
                    'node_b'
                ),
                error=None,
                interrupts=(
                    Interrupt(
                        value='hello',
                        id='6e3aa823181df313cfac730c024f4ed7'
                    ),
                ),
                state=None,
                result={
                    'node_b_output': '节点B的输出'
                }
            ),
        ),
        interrupts=(
            Interrupt(
                value='hello',
                id='6e3aa823181df313cfac730c024f4ed7'
            ),
        )
    ),

    StateSnapshot(
        values={
            'initial_state': '初始状态'
        },
        next=(
            'parallel_node_a_1',
            'parallel_node_a_2'
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f225-646d-8000-b7e7c665f75a'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-23T04:00:50.057725+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f221-6a80-bfff-d157e04c0492'
            }
        },
        tasks=(
            PregelTask(
                id='419afd20-961a-9a90-49dd-d19f1448ebe0',
                name='parallel_node_a_1',
                path=(
                    '__pregel_pull',
                    'parallel_node_a_1'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'parallel_node_a_1': '并行节点A-1的输出'
                }
            ),
            PregelTask(
                id='339d1e80-39f2-24d5-c677-3fb3f8314fd6',
                name='parallel_node_a_2',
                path=(
                    '__pregel_pull',
                    'parallel_node_a_2'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'parallel_node_a_2': '并行节点A-2的输出'
                }
            )
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={},
        next=(
            '__start__',
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16eb81-f221-6a80-bfff-d157e04c0492'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-23T04:00:50.056245+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='20116610-8f9e-c556-b647-778737961e5a',
                name='__start__',
                path=(
                    '__pregel_pull',
                    '__start__'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'initial_state': '初始状态'
                }
            ),
        ),
        interrupts=()
    )
]
```

| 对比维度 | `get_state_history()` | `checkpoints` 流 |
| :--- | :--- | :--- |
| 排序方式 | 时间倒序（最新在前） | 超步编号升序（正向执行顺序） |
| 顶层 `interrupts` | 有 | **无**（中断信息附着在 `tasks[].interrupts` 中） |


**中断时序说明**：检查点 `chunk` 在超步**第一阶段**输出，中断在**第二阶段**触发。因此中断发生时最新检查点仍为上一步（`step=1`），任务中断信息尚为空；恢复执行后第一阶段重新输出 `step=1` 检查点时，中断信息已被记录。

#### `tasks`
`tasks` 模式以任务为单位输出开始事件和结果事件。它比 `updates` 更偏向运行时观测：除了任务结果，还会暴露任务输入、触发通道、异常和中断信息。

**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class OverAllState(TypedDict):
    initial_state: str
    parallel_node_a_1: str
    parallel_node_a_2: str
    node_b_output: str

def parallel_node_a_1(state: OverAllState) -> OverAllState:
    return {
        "parallel_node_a_1": "并行节点A-1的输出"
    }

def parallel_node_a_2(state: OverAllState) -> OverAllState:
    return {
        "parallel_node_a_2": "并行节点A-2的输出"
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("parallel_node_a_1", parallel_node_a_1)
builder.add_node("parallel_node_a_2", parallel_node_a_2)
builder.add_node("node_b", node_b)
builder.add_edge(START, "parallel_node_a_1")
builder.add_edge(START, "parallel_node_a_2")
builder.add_edge(["parallel_node_a_1", "parallel_node_a_2"], "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()

for chunk in graph.stream(
    {"initial_state": "初始状态"},
    stream_mode=["tasks"]
):
    print(chunk)
```

**运行结果如下**

```json
(
    'tasks',
    {
        'id': '0b0c1e51-456f-d37c-346c-2109d78f8c2a',
        'name': 'parallel_node_a_1',
        'input': {
            'initial_state': '初始状态'
        },
        'triggers': (
            'branch:to:parallel_node_a_1',
        )
    }
)

(
    'tasks',
    {
        'id': '018ab5c2-0a5a-9094-8876-05668d3f27e0',
        'name': 'parallel_node_a_2',
        'input': {
            'initial_state': '初始状态'
        },
        'triggers': (
            'branch:to:parallel_node_a_2',
        )
    }
)

(
    'tasks',
    {
        'id': '0b0c1e51-456f-d37c-346c-2109d78f8c2a',
        'name': 'parallel_node_a_1',
        'error': None,
        'result': {
            'parallel_node_a_1': '并行节点A-1的输出'
        },
        'interrupts': []
    }
)

(
    'tasks',
    {
        'id': '018ab5c2-0a5a-9094-8876-05668d3f27e0',
        'name': 'parallel_node_a_2',
        'error': None,
        'result': {
            'parallel_node_a_2': '并行节点A-2的输出'
        },
        'interrupts': []
    }
)

(
    'tasks',
    {
        'id': '6ebeabdb-1735-2cab-ca3e-ea2fa67be0ff',
        'name': 'node_b',
        'input': {
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出'
        },
        'triggers': (
            'branch:to:node_b',
            'join:parallel_node_a_1+parallel_node_a_2:node_b'
        )
    }
)

(
    'tasks',
    {
        'id': '6ebeabdb-1735-2cab-ca3e-ea2fa67be0ff',
        'name': 'node_b',
        'error': None,
        'result': {
            'node_b_output': '节点B的输出'
        },
        'interrupts': []
    }
)
```

**任务 **`chunk`** 分为两类**：

| 事件类型 | 产生时机 | 关键字段 |
| :--- | :--- | :--- |
| **任务开始事件** | 任务准备完毕，尚未执行 | `input`（任务输入）、`triggers`（触发通道） |
| **任务结果事件** | 任务执行完成/失败/中断后 | `error`（异常，`None`=正常）、`result`、`interrupts` |


并行任务时：所有**开始事件**（第一阶段）全部输出后，才出现**结果事件**（第二阶段）。

#### `debug`
**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

from langgraph.types import interrupt, Command

class OverAllState(TypedDict):
    initial_state: str
    parallel_node_a_1: str
    parallel_node_a_2: str
    node_b_output: str

def parallel_node_a_1(state: OverAllState) -> OverAllState:
    return {
        "parallel_node_a_1": "并行节点A-1的输出"
    }

def parallel_node_a_2(state: OverAllState) -> OverAllState:
    return {
        "parallel_node_a_2": "并行节点A-2的输出"
    }

def node_b(state: OverAllState) -> OverAllState:
    interrupt("hello")
    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("parallel_node_a_1", parallel_node_a_1)
builder.add_node("parallel_node_a_2", parallel_node_a_2)
builder.add_node("node_b", node_b)
builder.add_edge(START, "parallel_node_a_1")
builder.add_edge(START, "parallel_node_a_2")
builder.add_edge(["parallel_node_a_1", "parallel_node_a_2"], "node_b")
builder.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "123"}}
for chunk in graph.stream(
    {"initial_state": "初始状态"},
    stream_mode=["debug"],
    config=config
):
    print(chunk)

print('=' * 30, '-> 中断前后分界线 <-', '=' * 30)

for chunk in graph.stream(
    Command(resume=""),
    stream_mode=["debug"],
    config=config
):
    print(chunk)
```

**运行结果如下**

```json
(
    'debug',
    {
        'step': -1,
        'timestamp': '2026-06-23T06:02:59.369405+00:00',
        'type': 'checkpoint',
        'payload': {
            'config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbe6-68b4-bfff-30257767db2f'
                }
            },
            'parent_config': None,
            'values': {},
            'metadata': {
                'source': 'input',
                'step': -1,
                'parents': {}
            },
            'next': [
                '__start__'
            ],
            'tasks': [
                {
                    'id': '3a25e32c-3a9a-a6fa-3c77-e958d2cf0021',
                    'name': '__start__',
                    'interrupts': (),
                    'state': None
                }
            ]
        }
    }
)

(
    'debug',
    {
        'step': 0,
        'timestamp': '2026-06-23T06:02:59.371035+00:00',
        'type': 'checkpoint',
        'payload': {
            'config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbe9-63f9-8000-bc529e935a70'
                }
            },
            'parent_config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbe6-68b4-bfff-30257767db2f'
                }
            },
            'values': {
                'initial_state': '初始状态'
            },
            'metadata': {
                'source': 'loop',
                'step': 0,
                'parents': {}
            },
            'next': [
                'parallel_node_a_1',
                'parallel_node_a_2'
            ],
            'tasks': [
                {
                    'id': '8f4954b8-e096-ed44-fd3b-b5741b66ce14',
                    'name': 'parallel_node_a_1',
                    'interrupts': (),
                    'state': None
                },
                {
                    'id': '33b02562-5b8b-c072-3dd4-d939f02f3835',
                    'name': 'parallel_node_a_2',
                    'interrupts': (),
                    'state': None
                }
            ]
        }
    }
)

(
    'debug',
    {
        'step': 1,
        'timestamp': '2026-06-23T06:02:59.371050+00:00',
        'type': 'task',
        'payload': {
            'id': '8f4954b8-e096-ed44-fd3b-b5741b66ce14',
            'name': 'parallel_node_a_1',
            'input': {
                'initial_state': '初始状态'
            },
            'triggers': (
                'branch:to:parallel_node_a_1',
            )
        }
    }
)

(
    'debug',
    {
        'step': 1,
        'timestamp': '2026-06-23T06:02:59.371054+00:00',
        'type': 'task',
        'payload': {
            'id': '33b02562-5b8b-c072-3dd4-d939f02f3835',
            'name': 'parallel_node_a_2',
            'input': {
                'initial_state': '初始状态'
            },
            'triggers': (
                'branch:to:parallel_node_a_2',
            )
        }
    }
)

(
    'debug',
    {
        'step': 1,
        'timestamp': '2026-06-23T06:02:59.371811+00:00',
        'type': 'task_result',
        'payload': {
            'id': '8f4954b8-e096-ed44-fd3b-b5741b66ce14',
            'name': 'parallel_node_a_1',
            'error': None,
            'result': {
                'parallel_node_a_1': '并行节点A-1的输出'
            },
            'interrupts': []
        }
    }
)

(
    'debug',
    {
        'step': 1,
        'timestamp': '2026-06-23T06:02:59.371861+00:00',
        'type': 'task_result',
        'payload': {
            'id': '33b02562-5b8b-c072-3dd4-d939f02f3835',
            'name': 'parallel_node_a_2',
            'error': None,
            'result': {
                'parallel_node_a_2': '并行节点A-2的输出'
            },
            'interrupts': []
        }
    }
)

(
    'debug',
    {
        'step': 1,
        'timestamp': '2026-06-23T06:02:59.372140+00:00',
        'type': 'checkpoint',
        'payload': {
            'config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbee-6f65-8001-4e724c2acb07'
                }
            },
            'parent_config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbe9-63f9-8000-bc529e935a70'
                }
            },
            'values': {
                'initial_state': '初始状态',
                'parallel_node_a_1': '并行节点A-1的输出',
                'parallel_node_a_2': '并行节点A-2的输出'
            },
            'metadata': {
                'source': 'loop',
                'step': 1,
                'parents': {}
            },
            'next': [
                'node_b'
            ],
            'tasks': [
                {
                    'id': 'e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
                    'name': 'node_b',
                    'interrupts': (),
                    'state': None
                }
            ]
        }
    }
)

(
    'debug',
    {
        'step': 2,
        'timestamp': '2026-06-23T06:02:59.372150+00:00',
        'type': 'task',
        'payload': {
            'id': 'e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
            'name': 'node_b',
            'input': {
                'initial_state': '初始状态',
                'parallel_node_a_1': '并行节点A-1的输出',
                'parallel_node_a_2': '并行节点A-2的输出'
            },
            'triggers': (
                'branch:to:node_b',
                'join:parallel_node_a_1+parallel_node_a_2:node_b'
            )
        }
    }
)

(
    'debug',
    {
        'step': 2,
        'timestamp': '2026-06-23T06:02:59.372332+00:00',
        'type': 'task_result',
        'payload': {
            'id': 'e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
            'name': 'node_b',
            'error': None,
            'result': {},
            'interrupts': [
                {
                    'value': 'hello',
                    'id': 'b195d3f05c81aac2aab438899fe67823'
                }
            ]
        }
    }
)

============================== -> 中断前后分界线 <- ==============================

(
    'debug',
    {
        'step': 1,
        'timestamp': '2026-06-23T06:02:59.373847+00:00',
        'type': 'checkpoint',
        'payload': {
            'config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbee-6f65-8001-4e724c2acb07'
                }
            },
            'parent_config': {
                'configurable': {
                    'thread_id': '123',
                    'checkpoint_ns': '',
                    'checkpoint_id': '1f16ec92-fbe9-63f9-8000-bc529e935a70'
                }
            },
            'values': {
                'initial_state': '初始状态',
                'parallel_node_a_1': '并行节点A-1的输出',
                'parallel_node_a_2': '并行节点A-2的输出'
            },
            'metadata': {
                'source': 'loop',
                'step': 1,
                'parents': {}
            },
            'next': [
                'node_b'
            ],
            'tasks': [
                {
                    'id': 'e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
                    'name': 'node_b',
                    'interrupts': (
                        {
                            'value': 'hello',
                            'id': 'b195d3f05c81aac2aab438899fe67823'
                        },
                    ),
                    'state': None
                }
            ]
        }
    }
)

(
    'debug',
    {
        'step': 2,
        'timestamp': '2026-06-23T06:02:59.373864+00:00',
        'type': 'task',
        'payload': {
            'id': 'e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
            'name': 'node_b',
            'input': {
                'initial_state': '初始状态',
                'parallel_node_a_1': '并行节点A-1的输出',
                'parallel_node_a_2': '并行节点A-2的输出'
            },
            'triggers': (
                'branch:to:node_b',
                'join:parallel_node_a_1+parallel_node_a_2:node_b'
            )
        }
    }
)

(
    'debug',
    {
        'step': 2,
        'timestamp': '2026-06-23T06:02:59.374037+00:00',
        'type': 'task_result',
        'payload': {
            'id': 'e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
            'name': 'node_b',
            'error': None,
            'result': {
                'node_b_output': '节点B的输出'
            },
            'interrupts': []
        }
    }
)

(
    'debug',
    {
        'step': 2,
        'timestamp': '2026-06-23T06:02:59.374356+00:00',
        'type': 'checkpoint',
        'payload': {
            'config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbf3-6ffe-8002-32361c07b81c'
                }
            },
            'parent_config': {
                'configurable': {
                    'checkpoint_ns': '',
                    'thread_id': '123',
                    'checkpoint_id': '1f16ec92-fbee-6f65-8001-4e724c2acb07'
                }
            },
            'values': {
                'initial_state': '初始状态',
                'parallel_node_a_1': '并行节点A-1的输出',
                'parallel_node_a_2': '并行节点A-2的输出',
                'node_b_output': '节点B的输出'
            },
            'metadata': {
                'source': 'loop',
                'step': 2,
                'parents': {}
            },
            'next': [],
            'tasks': []
        }
    }
)
```

**查看历史检查点**

```python
list(graph.get_state_history(config=config))
```

**结果如下**

```json
[
    StateSnapshot(
        values={
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出',
            'node_b_output': '节点B的输出'
        },
        next=(),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbf3-6ffe-8002-32361c07b81c'
            }
        },
        metadata={
            'source': 'loop',
            'step': 2,
            'parents': {}
        },
        created_at='2026-06-23T06:02:59.374073+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbee-6f65-8001-4e724c2acb07'
            }
        },
        tasks=(),
        interrupts=()
    ),

    StateSnapshot(
        values={
            'initial_state': '初始状态',
            'parallel_node_a_1': '并行节点A-1的输出',
            'parallel_node_a_2': '并行节点A-2的输出'
        },
        next=(
            'node_b',
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbee-6f65-8001-4e724c2acb07'
            }
        },
        metadata={
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        created_at='2026-06-23T06:02:59.372008+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbe9-63f9-8000-bc529e935a70'
            }
        },
        tasks=(
            PregelTask(
                id='e7d4c64e-8ab7-99ec-4031-170fd0122ed7',
                name='node_b',
                path=(
                    '__pregel_pull',
                    'node_b'
                ),
                error=None,
                interrupts=(
                    Interrupt(
                        value='hello',
                        id='b195d3f05c81aac2aab438899fe67823'
                    ),
                ),
                state=None,
                result={
                    'node_b_output': '节点B的输出'
                }
            ),
        ),
        interrupts=(
            Interrupt(
                value='hello',
                id='b195d3f05c81aac2aab438899fe67823'
            ),
        )
    ),

    StateSnapshot(
        values={
            'initial_state': '初始状态'
        },
        next=(
            'parallel_node_a_1',
            'parallel_node_a_2'
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbe9-63f9-8000-bc529e935a70'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-23T06:02:59.369665+00:00',
        parent_config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbe6-68b4-bfff-30257767db2f'
            }
        },
        tasks=(
            PregelTask(
                id='8f4954b8-e096-ed44-fd3b-b5741b66ce14',
                name='parallel_node_a_1',
                path=(
                    '__pregel_pull',
                    'parallel_node_a_1'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'parallel_node_a_1': '并行节点A-1的输出'
                }
            ),
            PregelTask(
                id='33b02562-5b8b-c072-3dd4-d939f02f3835',
                name='parallel_node_a_2',
                path=(
                    '__pregel_pull',
                    'parallel_node_a_2'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'parallel_node_a_2': '并行节点A-2的输出'
                }
            )
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={},
        next=(
            '__start__',
        ),
        config={
            'configurable': {
                'thread_id': '123',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16ec92-fbe6-68b4-bfff-30257767db2f'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-23T06:02:59.368563+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='3a25e32c-3a9a-a6fa-3c77-e958d2cf0021',
                name='__start__',
                path=(
                    '__pregel_pull',
                    '__start__'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'initial_state': '初始状态'
                }
            ),
        ),
        interrupts=()
    )
]
```

`debug`** 模式 = **`checkpoints`** + **`tasks`** 的统一封装**，外层增加调试字段：

| 字段 | 含义 |
| :--- | :--- |
| `step` | 超步编号 |
| `timestamp` | chunk 生成时间 |
| `type` | `checkpoint` / `task` / `task_result` |
| `payload` | 对应模式下 chunk 的原始内容 |


**chunk 输出顺序**：每个超步第一阶段按 `准备任务 → 检查点日志 → 任务日志` 的顺序生成，因此任务 chunk 位于前后两个超步的检查点 chunk 之间。中断恢复时对应超步会重新执行，任务 chunk 会再次生成。

   具体来说，`step=2` 的超步发生了中断，恢复运行时，`step=2` 会重新执行，其第一阶段会再次准备任务，并生成这一阶段的所有任务 `chunk`。

#### `custom`
`custom` 模式允许**节点或工具**主动写出自定义数据，调用方通过 `stream/astream` 消费这些数据。

它适合输出进度、阶段说明、外部 API 的增量结果等非状态数据。

##### 11.2.2.7.1. 节点中写出内容
节点中写出内容通过 `Runtime` 实例的 `stream_writer`（也可通过 `get_stream_writer()` 获取，二者等效）。当 `stream_mode` 包含 `custom` 时，底层会定义 `stream_writer` 并注入 `Runtime`，相关源码：

**定义 **`stream_writer`

```python
if "custom" in stream_modes:

    def stream_writer(c: Any) -> None:
        stream.put(
            (
                tuple(
                    get_config()[CONF][CONFIG_KEY_CHECKPOINT_NS].split(
                        NS_SEP
                    )[:-1]
                ),
                "custom",
                c,
            )
        )
```

创建 `Runtime` 实例

```python
runtime = Runtime(
    context=_coerce_context(self.context_schema, context),
    store=store,
    stream_writer=stream_writer,
    previous=None,
)
```

**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime

class OverAllState(TypedDict):
    initial_state: str
    node_a_output: str
    node_b_output: str

def node_a(state: OverAllState, runtime: Runtime) -> OverAllState:
    stream_writer = runtime.stream_writer
    stream_writer("节点 A 正在执行...")

    return {
        "node_a_output": "节点A的输出"
    }

def node_b(state: OverAllState, runtime: Runtime) -> OverAllState:
    stream_writer = runtime.stream_writer
    stream_writer("节点 B 正在执行...")

    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
for chunk in graph.stream(
    {"initial_state": "初始状态"},
    stream_mode=["custom"],
):
    print(chunk)
```

**运行结果如下**

```json
('custom', '节点 A 正在执行...')
('custom', '节点 B 正在执行...')
```

##### 11.2.2.7.2. 工具中写出内容
工具中写出内容通过 `ToolRuntime` 实例的 `stream_writer`，用法与节点中一致。

**示例如下**

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt.tool_node import ToolNode, ToolRuntime
from langgraph.runtime import Runtime
from langchain.tools import tool

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

@tool(parse_docstring=True)
def get_weather(city: str, runtime: ToolRuntime) -> str:
    """
    根据城市查询当日天气

    Args:
        city: 城市名称
    """
    stream_writer = runtime.stream_writer
    stream_writer(f"正在查询 {city} 今天的天气...")
    return f"{city} 今天天气不错"

tools = [get_weather]
model_with_tools = model.bind_tools(tools=tools)

def llm_node(state: MessagesState, runtime: Runtime) -> MessagesState:
    messages = state["messages"]
    response = model_with_tools.invoke(messages)

    stream_writer = runtime.stream_writer
    stream_writer("正在执行 llm_node...")

    return {
        "messages": [response]
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    last_msg = state["messages"][-1]
    if last_msg.tool_calls:
        return "tool_node"
    return END

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_node("tool_node", ToolNode(tools=tools))
builder.add_edge(START, "llm_node")
builder.add_conditional_edges("llm_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "llm_node")

graph = builder.compile()

for chunk in graph.stream(
    {"messages": [HumanMessage("今天北京天气如何？")]},
    stream_mode=["custom"]
):
    print(chunk)
```

**运行结果如下**

```json
('custom', '正在执行 llm_node...')
('custom', '正在查询 北京 今天的天气...')
('custom', '正在执行 llm_node...')
```

### 11.2.3. 底层机制
以 `LangGraph 1.1.2` 的 `stream()` 为例，流式处理分为三个阶段：

| 阶段 | 说明 |
| :--- | :--- |
| **创建队列** | `stream()` 调用 `SyncQueue()` 创建 FIFO 队列，接收运行时写入的 chunk |
| **生产** | 根据 `stream_mode`，在运行时的不同阶段将原始数据写入队列 |
| **消费** | 在两处调用 `_output()` 函数，按 FIFO 从队列取出数据 `yield` 给调用者：① `runner.tick()` 交还控制权时；② 超步循环结束后清空收尾 |


对 `messages` 和 `custom` 等需及时输出的模式，框架启用**等待器**，队列有新数据时立即交还控制权。

## 11.3. `astream_events`
`astream_events` 用于异步获取图运行过程中产生的 `Runnable`** 标准事件**。它关注的是可执行组件的生命周期、父子调用关系、输入输出和元数据，而不是单纯返回图状态。

`Runnable`**接口**

`Runnable` 是 `LangChain` 与 `LangGraph` 生态的统一可执行对象协议（`langchain_core.runnables`），统一了模型、工具、Agent、节点和编译图等组件的调用方式。

| 方法 | 说明 |
| :--- | :--- |
| `invoke()` / `ainvoke()` | 同步/异步调用 |
| `stream()` / `astream()` | 同步/异步流式执行 |
| `astream_events()` | 异步获取标准事件 |
| `batch()` | 批量执行 |


常见 `Runnable` 组件：`CompiledStateGraph`、`PromptTemplate`、`ChatModel`、`Tool`、`Retriever`、`RunnableSequence`。

`Runnable`**事件**

`Runnable` 执行过程会被框架转换为标准化事件，遵循 `on_<type>_<phase>` 命名规则：

| 阶段 | 事件后缀 | 含义 |
| :--- | :--- | :--- |
| 开始 | `_start` | 组件开始执行 |
| 中间 | `_stream` | 产生中间结果 |
| 结束 | `_end` | 组件执行结束 |


类型（`<type>`）可为 `chain`、`chat_model`、`llm`、`tool`、`retriever`、`prompt` 等。并非所有类型都有三种事件——例如工具通常只有 `on_tool_start` 和 `on_tool_end`。

实际运行拿到的 `Runnable`** 事件** 示例如下

```json
{
    'event': 'on_chain_start',
    'data': {
        'input': {
            'initial_state': '初始状态'
        }
    },
    'name': 'LangGraph',
    'tags': [],
    'run_id': '019eeedd-9dfd-7400-b465-6b95f7292333',
    'metadata': {},
    'parent_ids': []
}
```

### 11.3.1. `astream_events`的版本
从 `LangGraph 1.2.0` 开始，`astream_events` 支持三个版本：

| 版本 | 基础 | 特点 | 入口 |
| :--- | :--- | :--- | :--- |
| `v1` | `Runnable` 标准事件 | `parent_ids` 始终为空列表 | `astream_events(version="v1")` |
| `v2`（默认） | `Runnable` 标准事件 | `parent_ids` 可表达完整父级运行链，改进父子调用关系 | `astream_events(version="v2")` |
| `v3` | `Pregel` 原始流（= `stream/astream` 底层） | 全新协议，通过类型化投影消费（见下表），独立于 v1/v2 | `astream_events(version="v3")` / `stream_events(version="v3")` |


`v3` 从底层 `stream/astream` 获取 `Pregel` 原始流数据，经事件路由器与 `StreamTransformer` 生成类型化投影。投影之间可独立/并发消费：

| 投影 | 用法 |
| :--- | :--- |
| `stream` | 遍历每个协议事件 |
| `stream.messages` | 流式传输聊天模型的消息。 |
| `stream.values` | 流式传输状态快照 |
| `stream.output` | 等待计算图的最终输出 |
| `stream.subgraphs` | 观测子图的运行 |
| `stream.interrupts` | 观测 `HITL` 的中断信息 |
| `stream.interrupted` | 检查运行是否因人工输入而中断 |
| `stream.extensions` | 消费自定义流的投影 |


不同投影之间可以独立或并发消费，读取 `stream.messages` 不会消耗 `stream.values` 、 `stream.subgraphs` 或 `stream.output` 所需的事件。

`Pregel` 流即 `stream/astream` 的底层支持，上文 7 种 `stream_mode` 分别对应其中的 7 类流式数据。v3 并非 v1/v2 的封装——二者数据协议和内部管线独立，但共享同一组公开 API 入口和底层图运行时。

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260623180949275.png)

当前版本（`LangGraph 1.1.2`）仅支持 `astream_events` 的 `v1/v2`，默认版本为 `v2`。

### 11.3.2. `astream_events`的用法
事件流不是我们研究的重点，此处仅提供最简示例。

**示例如下**

底层也会调用 `astream` 来真正触发计算图的运行。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class OverAllState(TypedDict):
    initial_state: str
    node_a_output: str
    node_b_output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "node_a_output": "节点A的输出"
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "node_b_output": "节点B的输出"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
async for chunk in graph.astream_events(
    {"initial_state": "初始状态"},
    version="v2"
):
    print(chunk)
```

**运行结果如下**

```python
{
    'event': 'on_chain_start',
    'data': {
        'input': {
            'initial_state': '初始状态'
        }
    },
    'name': 'LangGraph',
    'tags': [],
    'run_id': '019ef40f-2522-7212-9ad6-cb83c499652d',
    'metadata': {},
    'parent_ids': []
}

{
    'event': 'on_chain_start',
    'data': {
        'input': {
            'initial_state': '初始状态'
        }
    },
    'name': 'node_a',
    'tags': [
        'graph:step:1'
    ],
    'run_id': '019ef40f-2524-7f83-9501-9248f93fd0a4',
    'metadata': {
        'langgraph_step': 1,
        'langgraph_node': 'node_a',
        'langgraph_triggers': (
            'branch:to:node_a',
        ),
        'langgraph_path': (
            '__pregel_pull',
            'node_a'
        ),
        'langgraph_checkpoint_ns': 'node_a:9a47e3e1-ccab-85cc-f837-f3c986bafd1f'
    },
    'parent_ids': [
        '019ef40f-2522-7212-9ad6-cb83c499652d'
    ]
}

{
    'event': 'on_chain_stream',
    'run_id': '019ef40f-2524-7f83-9501-9248f93fd0a4',
    'name': 'node_a',
    'tags': [
        'graph:step:1'
    ],
    'metadata': {
        'langgraph_step': 1,
        'langgraph_node': 'node_a',
        'langgraph_triggers': (
            'branch:to:node_a',
        ),
        'langgraph_path': (
            '__pregel_pull',
            'node_a'
        ),
        'langgraph_checkpoint_ns': 'node_a:9a47e3e1-ccab-85cc-f837-f3c986bafd1f'
    },
    'data': {
        'chunk': {
            'node_a_output': '节点A的输出'
        }
    },
    'parent_ids': [
        '019ef40f-2522-7212-9ad6-cb83c499652d'
    ]
}

{
    'event': 'on_chain_end',
    'data': {
        'output': {
            'node_a_output': '节点A的输出'
        },
        'input': {
            'initial_state': '初始状态'
        }
    },
    'run_id': '019ef40f-2524-7f83-9501-9248f93fd0a4',
    'name': 'node_a',
    'tags': [
        'graph:step:1'
    ],
    'metadata': {
        'langgraph_step': 1,
        'langgraph_node': 'node_a',
        'langgraph_triggers': (
            'branch:to:node_a',
        ),
        'langgraph_path': (
            '__pregel_pull',
            'node_a'
        ),
        'langgraph_checkpoint_ns': 'node_a:9a47e3e1-ccab-85cc-f837-f3c986bafd1f'
    },
    'parent_ids': [
        '019ef40f-2522-7212-9ad6-cb83c499652d'
    ]
}

{
    'event': 'on_chain_stream',
    'run_id': '019ef40f-2522-7212-9ad6-cb83c499652d',
    'name': 'LangGraph',
    'tags': [],
    'metadata': {},
    'data': {
        'chunk': {
            'node_a': {
                'node_a_output': '节点A的输出'
            }
        }
    },
    'parent_ids': []
}

{
    'event': 'on_chain_start',
    'data': {
        'input': {
            'initial_state': '初始状态',
            'node_a_output': '节点A的输出'
        }
    },
    'name': 'node_b',
    'tags': [
        'graph:step:2'
    ],
    'run_id': '019ef40f-2526-70a3-8ad1-04ce18598b65',
    'metadata': {
        'langgraph_step': 2,
        'langgraph_node': 'node_b',
        'langgraph_triggers': (
            'branch:to:node_b',
        ),
        'langgraph_path': (
            '__pregel_pull',
            'node_b'
        ),
        'langgraph_checkpoint_ns': 'node_b:1fc56ad3-e5a9-e2a2-4c3e-b8cffec34d9c'
    },
    'parent_ids': [
        '019ef40f-2522-7212-9ad6-cb83c499652d'
    ]
}

{
    'event': 'on_chain_stream',
    'run_id': '019ef40f-2526-70a3-8ad1-04ce18598b65',
    'name': 'node_b',
    'tags': [
        'graph:step:2'
    ],
    'metadata': {
        'langgraph_step': 2,
        'langgraph_node': 'node_b',
        'langgraph_triggers': (
            'branch:to:node_b',
        ),
        'langgraph_path': (
            '__pregel_pull',
            'node_b'
        ),
        'langgraph_checkpoint_ns': 'node_b:1fc56ad3-e5a9-e2a2-4c3e-b8cffec34d9c'
    },
    'data': {
        'chunk': {
            'node_b_output': '节点B的输出'
        }
    },
    'parent_ids': [
        '019ef40f-2522-7212-9ad6-cb83c499652d'
    ]
}

{
    'event': 'on_chain_end',
    'data': {
        'output': {
            'node_b_output': '节点B的输出'
        },
        'input': {
            'initial_state': '初始状态',
            'node_a_output': '节点A的输出'
        }
    },
    'run_id': '019ef40f-2526-70a3-8ad1-04ce18598b65',
    'name': 'node_b',
    'tags': [
        'graph:step:2'
    ],
    'metadata': {
        'langgraph_step': 2,
        'langgraph_node': 'node_b',
        'langgraph_triggers': (
            'branch:to:node_b',
        ),
        'langgraph_path': (
            '__pregel_pull',
            'node_b'
        ),
        'langgraph_checkpoint_ns': 'node_b:1fc56ad3-e5a9-e2a2-4c3e-b8cffec34d9c'
    },
    'parent_ids': [
        '019ef40f-2522-7212-9ad6-cb83c499652d'
    ]
}

{
    'event': 'on_chain_stream',
    'run_id': '019ef40f-2522-7212-9ad6-cb83c499652d',
    'name': 'LangGraph',
    'tags': [],
    'metadata': {},
    'data': {
        'chunk': {
            'node_b': {
                'node_b_output': '节点B的输出'
            }
        }
    },
    'parent_ids': []
}

{
    'event': 'on_chain_end',
    'data': {
        'output': {
            'initial_state': '初始状态',
            'node_a_output': '节点A的输出',
            'node_b_output': '节点B的输出'
        }
    },
    'run_id': '019ef40f-2522-7212-9ad6-cb83c499652d',
    'name': 'LangGraph',
    'tags': [],
    'metadata': {},
    'parent_ids': []
}
```

**事件分类**：纵向按生命周期分为 `on_xxx_start`（开始）、`on_xxx_stream`（中间结果）、`on_xxx_end`（结束）；横向按 `Runnable` 类型分为以下事件：

| 组件类型 | 事件前缀 | name 示例 |
| :--- | :--- | :--- |
| 模型 | `on_chat_model_*` / `on_llm_*` | `'[model name]'` |
| 工具 | `on_tool_*`（仅 start/end） | `'some_tool'` |
| 检索器 | `on_retriever_*` | `'[retriever name]'` |
| 提示词模板 | `on_prompt_*` | `'[template_name]'` |
| 通用（含编译图、普通节点） | `on_chain_*` | `'format_docs'` 等 |


**关键字段**：

| 字段 | 含义 |
| :--- | :--- |
| `event` | 事件类型 |
| `run_id` | 当前 Runnable 运行实例的唯一 ID |
| `name` | 运行实例名称 |
| `metadata` | 运行相关元数据 |
| `data` | 事件输入/输出/中间结果 |
| `parent_ids` | 从根运行实例到父运行实例的 ID 链（最外层 LangGraph 为空，子节点包含父 run_id） |


# 12. 子图
状态图节点中调用另外的状态图或直接将另外的状态图作为其节点则后者为子图。

**注意：**

为节点命名时特别注意避开 `Mermaid` 关键字，否则拓扑结构渲染可能失败，常见的 `Mermaid` 关键字如下

```python
subgraph
end
graph
flowchart
classDef
```

## 12.1. 两种子图嵌入方式
| 方式 | 用法 | 适用场景 | 通信方式 |
| :--- | :--- | :--- | :--- |
| **节点函数中调用子图** | 在节点函数内 `subgraph.invoke()` | 父子图状态完全隔离 | 手动做输入输出映射 |
| **子图直接作为父图节点** | `add_node("name", compiled_subgraph)` | 父子图共享状态字段 | 通过共享字段自动通信 |


### 12.1.1. 在节点函数中调用子图
如果父图和子图状态完全隔离，可以用这种方式。

#### 12.1.1.1. 用法
**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# 构建子图
class SubgraphState(TypedDict):
    raw_text: str # 未清洗文本
    stripped_text: str # 去除收尾空格的文本
    punctuated_text: str # 句尾添加句号的文本

def subgraph_strip_node(state: SubgraphState) -> SubgraphState:
    raw_text = state["raw_text"]
    stripped_text = raw_text.strip()

    return {
        "stripped_text": stripped_text
    }

def subgraph_punctuate_node(state: SubgraphState) -> SubgraphState:
    stripped_text = state["stripped_text"]
    punctuated_text = stripped_text + "。"

    return {
        "punctuated_text": punctuated_text
    }

builder = StateGraph(state_schema=SubgraphState)
builder.add_node("subgraph_strip_node", subgraph_strip_node)
builder.add_node("subgraph_punctuate_node", subgraph_punctuate_node)

builder.add_edge(START, "subgraph_strip_node")
builder.add_edge("subgraph_strip_node", "subgraph_punctuate_node")
builder.add_edge("subgraph_punctuate_node", END)

subgraph = builder.compile()

# 构建父图
class ParentState(TypedDict):
    input_text: str # 输入的未清洗的文本
    cleaned_text: str # 清洗后的文本

def call_subgraph(state: ParentState) -> ParentState:
    input_text = state["input_text"]

    res = subgraph.invoke({"raw_text": input_text})
    cleaned_text = res["punctuated_text"]
    return {
        "cleaned_text": cleaned_text
    }

builder = StateGraph(state_schema=ParentState)
builder.add_node("call_subgraph", call_subgraph)

builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

parent_graph = builder.compile()

input_text = "   LangGraph 真有意思        "
res = parent_graph.invoke({"input_text": input_text})
cleaned_text = res["cleaned_text"]
print("=" * 30, "->   原始文本   <-", "=" * 30)
print(input_text)
print("=" * 30, "-> 清洗后的文本 <-", "=" * 30)
print(cleaned_text)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

调用 `get_graph()` 时传递的 `xray=True` 表示自动解析并递归获取子图的拓扑结构。

**运行结果如下**

```json
============================== ->   原始文本   <- ==============================
   LangGraph 真有意思        
============================== -> 清洗后的文本 <- ==============================
LangGraph 真有意思。
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625084145735.png)

**注意：**

当前方式的子图信息依赖于 `LangGraph` 对于节点函数代码的解析，解析的关键代码如下

```python
res = subgraph.invoke({"raw_text": input_text})
```

这是推荐的写法，调用子图时最好不要链式调用，否则解析失败，`LangGraph` 可能无法感知子图，这不仅会导致 `display` 渲染时不包含子图的拓扑结构，还会导致启用检查点存储器时，无法查看子图检查点快照。

**错误写法示例如下**

当前案例看不到效果，下文的 `Multi-turn` 案例能看到差别

```python
assistant_response = subgraph.invoke(
    {"messages": messages}
)["messages"][-1].content
```

此时，`LangGraph` 无法感知这里的子图 `subgraph`

**正确写法如下**

```python
subgraph_response = subgraph.invoke(
    {"messages": messages}
)

assistant_response = subgraph_response["messages"][-1].content
```

#### 12.1.1.2. 查看子图
在上述代码的基础上执行以下代码

```python
list(parent_graph.get_subgraphs())
```

**结果如下**

```python
[('call_subgraph',
  <langgraph.graph.state.CompiledStateGraph at 0x19bd9e84ec0>)]
```

### 12.1.2. 子图直接作为父图的节点
#### 12.1.2.1. 用法
**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# 定义全局共享状态
class OverAllState(TypedDict):
    raw_text: str # 未清洗文本
    cleaned_text: str # 清洗后的文本

# 构建子图
def subgraph_strip_node(state: OverAllState) -> OverAllState:
    raw_text = state["raw_text"]
    stripped_text = raw_text.strip()

    return {
        "cleaned_text": stripped_text
    }

def subgraph_punctuate_node(state: OverAllState) -> OverAllState:
    cleaned_text = state["cleaned_text"]
    punctuated_text = cleaned_text + "。"

    return {
        "cleaned_text": punctuated_text
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("subgraph_strip_node", subgraph_strip_node)
builder.add_node("subgraph_punctuate_node", subgraph_punctuate_node)

builder.add_edge(START, "subgraph_strip_node")
builder.add_edge("subgraph_strip_node", "subgraph_punctuate_node")
builder.add_edge("subgraph_punctuate_node", END)

subgraph = builder.compile()

# 构建父图
builder = StateGraph(state_schema=OverAllState)
builder.add_node("subgraph_node", subgraph)

builder.add_edge(START, "subgraph_node")
builder.add_edge("subgraph_node", END)

parent_graph = builder.compile()

raw_text = "   LangGraph 真有意思        "
res = parent_graph.invoke({"raw_text": raw_text})
cleaned_text = res["cleaned_text"]
print("=" * 30, "->   原始文本   <-", "=" * 30)
print(raw_text)
print("=" * 30, "-> 清洗后的文本 <-", "=" * 30)
print(cleaned_text)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**运行结果如下**

```json
============================== ->   原始文本   <- ==============================
   LangGraph 真有意思        
============================== -> 清洗后的文本 <- ==============================
LangGraph 真有意思。
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625085408914.png)

#### 12.1.2.2. 查看子图
**示例如下**

```python
list(parent_graph.get_subgraphs())
```

**运行结果如下**

```python
[('subgraph_node',
  <langgraph.graph.state.CompiledStateGraph at 0x297c50a2520>)]
```

## 12.2. 子图持久化
### 12.2.1. 查看子图检查点快照
本节测试都依赖于以下示例

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

# 构建子图
class SubgraphState(TypedDict):
    raw_text: str # 未清洗文本
    stripped_text: str # 去除收尾空格的文本
    punctuated_text: str # 句尾添加句号的文本

def subgraph_strip_node(state: SubgraphState) -> SubgraphState:
    raw_text = state["raw_text"]
    stripped_text = raw_text.strip()

    return {
        "stripped_text": stripped_text
    }

def subgraph_punctuate_node(state: SubgraphState) -> SubgraphState:
    stripped_text = state["stripped_text"]
    punctuated_text = stripped_text + "。"

    return {
        "punctuated_text": punctuated_text
    }

builder = StateGraph(state_schema=SubgraphState)
builder.add_node("subgraph_strip_node", subgraph_strip_node)
builder.add_node("subgraph_punctuate_node", subgraph_punctuate_node)

builder.add_edge(START, "subgraph_strip_node")
builder.add_edge("subgraph_strip_node", "subgraph_punctuate_node")
builder.add_edge("subgraph_punctuate_node", END)

subgraph = builder.compile()

# 构建父图
class ParentState(TypedDict):
    input_text: str # 输入的未清洗的文本
    cleaned_text: str # 清洗后的文本

def call_subgraph(state: ParentState) -> ParentState:
    input_text = state["input_text"]

    res = subgraph.invoke({"raw_text": input_text})
    cleaned_text = res["punctuated_text"]
    return {
        "cleaned_text": cleaned_text
    }

builder = StateGraph(state_schema=ParentState)
builder.add_node("call_subgraph", call_subgraph)

builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

checkpointer=InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

input_text = "   LangGraph 真有意思        "
config={"configurable": {"thread_id": "observe_checkpoint_1"}}
res = parent_graph.invoke(
    {"input_text": input_text},
    config=config
)
cleaned_text = res["cleaned_text"]
print("=" * 30, "->   原始文本   <-", "=" * 30)
print(input_text)
print("=" * 30, "-> 清洗后的文本 <-", "=" * 30)
print(cleaned_text)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**运行结果如下**

```python
============================== ->   原始文本   <- ==============================
   LangGraph 真有意思        
============================== -> 清洗后的文本 <- ==============================
LangGraph 真有意思。
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625085620495.png)

#### 12.2.1.1. 在父图检查点快照中查看子图信息
**获取父图历史检查点列表**

```python
histories = list(parent_graph.get_state_history(config=config))
histories
```

**结果如下**

```python
[
    StateSnapshot(
        values={
            'input_text': '   LangGraph 真有意思        ',
            'cleaned_text': 'LangGraph 真有意思。'
        },
        next=(),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16fd86-3406-6072-8001-1e8b9691a9cf'
            }
        },
        metadata={
            'source': 'loop',
            'step': 1,
            'parents': {}
        },
        created_at='2026-06-24T14:24:19.397838+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16fd86-33fb-6847-8000-8914ff1be716'
            }
        },
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'input_text': '   LangGraph 真有意思        '
        },
        next=(
            'call_subgraph',
        ),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16fd86-33fb-6847-8000-8914ff1be716'
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {}
        },
        created_at='2026-06-24T14:24:19.393533+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16fd86-33fa-61a4-bfff-538c9fb00687'
            }
        },
        tasks=(
            PregelTask(
                id='9923d4be-9845-576d-785f-5b1d6da9a94f',
                name='call_subgraph',
                path=(
                    '__pregel_pull',
                    'call_subgraph'
                ),
                error=None,
                interrupts=(),
                state={
                    'configurable': {
                        'thread_id': 'observe_checkpoint_1',
                        'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f'
                    }
                },
                result={
                    'cleaned_text': 'LangGraph 真有意思。'
                }
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={},
        next=(
            '__start__',
        ),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': '',
                'checkpoint_id': '1f16fd86-33fa-61a4-bfff-538c9fb00687'
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {}
        },
        created_at='2026-06-24T14:24:19.392958+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='1b333b41-1e2d-1770-8dab-a09a47002aef',
                name='__start__',
                path=(
                    '__pregel_pull',
                    '__start__'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'input_text': '   LangGraph 真有意思        '
                }
            ),
        ),
        interrupts=()
    )
]
```

可以看到，超步为 `0` 的快照下的 `tasks` 字段下只有一个任务实例，该任务实例在编号为 `1` 的超步执行，其中的 `state` 字段记录了子图检查点的 `thread_id` 和 `checkpoint_ns`，如下所示：

```python
state={
    'configurable': {
        'thread_id': 'observe_checkpoint_1',
        'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f'
    }
}
```

+ `thread_id`：整个线程的唯一 `ID`，和父图一致
+ `checkpoint_ns`：子图命名空间，格式为

```python
<子图所属父图节点名称>:<子图所属父图任务ID>
```

我们在介绍 `LangGraph` 中提到，父图的 `checkpoint_ns` 通常为空字符串，在子图嵌套系统中该字段排上了用场：

当嵌套层级不止一层时，命名空间将会通过 `|` 拼接，具体来说，假设拓扑结构如下：

```python
父图
└── call_subgraph          # 调用子图 A
    └── call_inner_graph   # 子图 A 调用子图 B
```

那么命名空间为

```python
父图：
""

子图 A：
call_subgraph:<task_id_1>

子图 B：
call_subgraph:<task_id_1>|call_inner_graph:<task_id_2>
```

依次类推。

    - 父图的 `checkpoint_ns` 仍未空字符串
    - 子图的 `checkpoint_ns` 遵循以上格式

#### 12.2.1.2. 在父图检查点快照中展开子图最新快照
我们可以在父图的 `state` 字段中展开子图快照，也就是将 `state` 字段的值替换为子图的最新检查点快照

##### 12.2.1.2.1. 获取父图检查点配置
超步为 `0` 的检查点记录了子图的配置，所以我们要获取这个快照的父图配置，观察检查点快照列表可知：快照列表按照超步编号降序排列，且最小超步为 `-1`，取列表的倒数第二个元素即可拿到超步编号为 `0` 的检查点快照。

**代码如下**

```python
parent_config = histories[-2].config
parent_config
```

**结果如下**

```python
{
  'configurable': {
    'thread_id': 'observe_checkpoint_1',
    'checkpoint_ns': '',
    'checkpoint_id': '1f16fd86-33fb-6847-8000-8914ff1be716'
  }
}
```

##### 12.2.1.2.2. 展开子图快照
在调用 `get_state()` 时传递带有 `checkpoint_id` 的父图配置，并传递 `subgraphs=True` 即可展开子图快照。

**示例如下**

```python
parent_graph.get_state(config=parent_config, subgraphs=True)
```

**运行结果如下**

```python
StateSnapshot(
  values={
    'input_text': '   LangGraph 真有意思        '
  },
  next=(
    'call_subgraph',
  ),
  config={
    'configurable': {
      'thread_id': 'observe_checkpoint_1',
      'checkpoint_ns': '',
      'checkpoint_id': '1f16fd86-33fb-6847-8000-8914ff1be716'
    }
  },
  metadata={
    'source': 'loop',
    'step': 0,
    'parents': {}
  },
  created_at='2026-06-24T14:24:19.393533+00:00',
  parent_config={
    'configurable': {
      'thread_id': 'observe_checkpoint_1',
      'checkpoint_ns': '',
      'checkpoint_id': '1f16fd86-33fa-61a4-bfff-538c9fb00687'
    }
  },
  tasks=(
    PregelTask(
      id='9923d4be-9845-576d-785f-5b1d6da9a94f',
      name='call_subgraph',
      path=(
        '__pregel_pull',
        'call_subgraph'
      ),
      error=None,
      interrupts=(),
      state=StateSnapshot(
        values={
          'raw_text': '   LangGraph 真有意思        ',
          'stripped_text': 'LangGraph 真有意思',
          'punctuated_text': 'LangGraph 真有意思。'
        },
        next=(),
        config={
          'configurable': {
            'thread_id': 'observe_checkpoint_1',
            'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
            'checkpoint_id': '1f16fd86-3404-640e-8002-84e2d98a88dc',
            'checkpoint_map': {
              '': '1f16fd86-33fb-6847-8000-8914ff1be716',
              'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-3404-640e-8002-84e2d98a88dc'
            }
          }
        },
        metadata={
          'source': 'loop',
          'step': 2,
          'parents': {
            '': '1f16fd86-33fb-6847-8000-8914ff1be716'
          }
        },
        created_at='2026-06-24T14:24:19.397114+00:00',
        parent_config={
          'configurable': {
            'thread_id': 'observe_checkpoint_1',
            'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
            'checkpoint_id': '1f16fd86-3401-694f-8001-2c6a2666b501',
            'checkpoint_map': {
              '': '1f16fd86-33fb-6847-8000-8914ff1be716',
              'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-3401-694f-8001-2c6a2666b501'
            }
          }
        },
        tasks=(),
        interrupts=()
      ),
      result={
        'cleaned_text': 'LangGraph 真有意思。'
      }
    ),
  ),
  interrupts=()
)
```

可以看到，`state` 字段已展开为子图的最新检查点快照。

#### 12.2.1.3. 获取子图配置，查看完整子图检查点列表
某些场景下，我们希望查看完整的子图检查点快照列表，此时只需要将包含 `thread_id` 和子图命名空间 `checkpoint_ns` 的配置信息传递给 `get_state_history()` 即可实现。

##### 12.2.1.3.1. 获取子图检查点配置
**示例如下**

```python
subgraph_config = histories[-2].tasks[0].state
subgraph_config
```

**结果如下**

```python
{
  'configurable': {
    'thread_id': 'observe_checkpoint_1',
    'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f'
  }
}
```

##### 12.2.1.3.2. 查看完整子图检查点快照列表
**示例如下**

```python
list(parent_graph.get_state_history(config=subgraph_config))
```

**结果如下**

```python
[
    StateSnapshot(
        values={
            'raw_text': '   LangGraph 真有意思        ',
            'stripped_text': 'LangGraph 真有意思',
            'punctuated_text': 'LangGraph 真有意思。'
        },
        next=(),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-3404-640e-8002-84e2d98a88dc',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-3404-640e-8002-84e2d98a88dc'
                }
            }
        },
        metadata={
            'source': 'loop',
            'step': 2,
            'parents': {
                '': '1f16fd86-33fb-6847-8000-8914ff1be716'
            }
        },
        created_at='2026-06-24T14:24:19.397114+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-3401-694f-8001-2c6a2666b501',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-3401-694f-8001-2c6a2666b501'
                }
            }
        },
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'raw_text': '   LangGraph 真有意思        ',
            'stripped_text': 'LangGraph 真有意思'
        },
        next=(
            'subgraph_punctuate_node',
        ),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-3401-694f-8001-2c6a2666b501',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-3401-694f-8001-2c6a2666b501'
                }
            }
        },
        metadata={
            'source': 'loop',
            'step': 1,
            'parents': {
                '': '1f16fd86-33fb-6847-8000-8914ff1be716'
            }
        },
        created_at='2026-06-24T14:24:19.396020+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-33ff-6ad9-8000-0fb11208046d',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-33ff-6ad9-8000-0fb11208046d'
                }
            }
        },
        tasks=(
            PregelTask(
                id='d37272d8-d167-4fb9-6e31-466a58bf91cc',
                name='subgraph_punctuate_node',
                path=(
                    '__pregel_pull',
                    'subgraph_punctuate_node'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'punctuated_text': 'LangGraph 真有意思。'
                }
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'raw_text': '   LangGraph 真有意思        '
        },
        next=(
            'subgraph_strip_node',
        ),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-33ff-6ad9-8000-0fb11208046d',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-33ff-6ad9-8000-0fb11208046d'
                }
            }
        },
        metadata={
            'source': 'loop',
            'step': 0,
            'parents': {
                '': '1f16fd86-33fb-6847-8000-8914ff1be716'
            }
        },
        created_at='2026-06-24T14:24:19.395239+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-33fe-685b-bfff-d0a73977e807',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-33fe-685b-bfff-d0a73977e807'
                }
            }
        },
        tasks=(
            PregelTask(
                id='9d9a1009-3212-b789-f40a-3d8f213ce4fb',
                name='subgraph_strip_node',
                path=(
                    '__pregel_pull',
                    'subgraph_strip_node'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'stripped_text': 'LangGraph 真有意思'
                }
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={},
        next=(
            '__start__',
        ),
        config={
            'configurable': {
                'thread_id': 'observe_checkpoint_1',
                'checkpoint_ns': 'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f',
                'checkpoint_id': '1f16fd86-33fe-685b-bfff-d0a73977e807',
                'checkpoint_map': {
                    '': '1f16fd86-33fb-6847-8000-8914ff1be716',
                    'call_subgraph:9923d4be-9845-576d-785f-5b1d6da9a94f': '1f16fd86-33fe-685b-bfff-d0a73977e807'
                }
            }
        },
        metadata={
            'source': 'input',
            'step': -1,
            'parents': {
                '': '1f16fd86-33fb-6847-8000-8914ff1be716'
            }
        },
        created_at='2026-06-24T14:24:19.394769+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='5516615f-62ca-2dc4-d41e-6a8f20a981bb',
                name='__start__',
                path=(
                    '__pregel_pull',
                    '__start__'
                ),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'raw_text': '   LangGraph 真有意思        '
                }
            ),
        ),
        interrupts=()
    )
]
```

+ 上述快照的命名空间都不是空字符串，且保持一致，全部属于同一个子图
+ `metadata['parents']` 记录了父检查点的信息，格式为`<父检查点命名空间>:<父检查点ID>`
+ `config['configurable']['checkpoint_map']` 记录了父图和当前子图检查点命名空间和 `ID` 的映射关系，包含两个条目，格式如下

```python
<父图检查点命名空间>:<父图检查点ID>,
<当前子图检查点命名空间>:<当前子图检查点ID>
```

### 12.2.2. 持久化策略
本节讨论的持久化策略以**父图启用检查点存储器**为前提。子图支持三种策略：

| 策略 | 编译参数 | 检查点保存 | 中断恢复 | 多轮记忆 |
| :--- | :--- | :--- | :--- | :--- |
| `Per-invocation`（默认） | `checkpointer=None` 或省略 | ✓ | ✓ | ✗（同 thread_id 再次调用不加载历史） |
| `Per-thread` | `checkpointer=True` | ✓ | ✓ | ✓（同 thread_id 调用加载历史） |
| `Stateless` | `checkpointer=False` | ✗ | ✗ | ✗ |


> **三种策略的唯一区别在于子图编译时的 **`checkpointer`** 参数**，其余代码完全相同。下面按场景组织，每个场景展示一份完整代码，策略切换只需修改一行。
>

#### 12.2.2.1. 场景一：中断（interrupt）
子图节点中使用 `interrupt()` 中断执行，等待用户输入后恢复。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

from loguru import logger

# 定义全局共享状态
class OverAllState(TypedDict):
    raw_text: str # 未清洗文本
    cleaned_text: str # 清洗后的文本

# 构建子图
def subgraph_strip_node(state: OverAllState) -> OverAllState:
    logger.info("子图 subgraph_strip_node 节点执行了")
    raw_text = state["raw_text"]
    stripped_text = raw_text.strip()

    return {
        "cleaned_text": stripped_text
    }

def subgraph_punctuate_node(state: OverAllState) -> OverAllState:
    logger.info("子图 subgraph_punctuate_node 节点执行了")
    cleaned_text = state["cleaned_text"]
    punctuation = interrupt("您希望在句尾添加的标点符号是？[。/，/！/？/；]")
    punctuation1 = interrupt("您希望在句尾添加的标点符号是？[。/，/！/？/；]")
    punctuated_text = cleaned_text + punctuation + punctuation1

    return {
        "cleaned_text": punctuated_text
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("subgraph_strip_node", subgraph_strip_node)
builder.add_node("subgraph_punctuate_node", subgraph_punctuate_node)

builder.add_edge(START, "subgraph_strip_node")
builder.add_edge("subgraph_strip_node", "subgraph_punctuate_node")
builder.add_edge("subgraph_punctuate_node", END)

# ==================================================
# 【策略切换点】只需修改这一行
# subgraph = builder.compile()  # Per-invocation（默认）
# subgraph = builder.compile(checkpointer=True)   # Per-thread
subgraph = builder.compile(checkpointer=False)  # Stateless（中断不可用）
# ==================================================

# 构建父图
builder = StateGraph(state_schema=OverAllState)
builder.add_node("subgraph_node", subgraph)

builder.add_edge(START, "subgraph_node")
builder.add_edge("subgraph_node", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

raw_text = "   LangGraph 真有意思        "
config = {"configurable": {"thread_id": "123"}}
# 首次调用
interrupted_res = parent_graph.invoke(
    {"raw_text": raw_text},
    config=config
)
print("=" * 30, "->   中断信息   <-", "=" * 30)
print(interrupted_res)


from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)

# 恢复调用
parent_graph.invoke(
    Command(resume="！"),
    config=config
)
res = parent_graph.invoke(
    Command(resume="！"),
    config=config
)

cleaned_text = res["cleaned_text"]
print("=" * 30, "->   原始文本   <-", "=" * 30)
print(raw_text)
print("=" * 30, "-> 清洗后的文本 <-", "=" * 30)
print(cleaned_text)
```

**运行结果（Per-invocation / Per-thread 均可）**

```python
2026-06-25 09:41:40.629 | INFO     | __main__:subgraph_strip_node:15 - 子图 subgraph_strip_node 节点执行了
2026-06-25 09:41:40.630 | INFO     | __main__:subgraph_punctuate_node:24 - 子图 subgraph_punctuate_node 节点执行了
============================== ->   中断信息   <- ==============================
{'raw_text': '   LangGraph 真有意思        ', '__interrupt__': [Interrupt(value='您希望在句尾添加的标点符号是？[。/，/！/？/；]', id='a4c6479870430be1a857af43cc7eff99')]}
2026-06-25 09:41:40.633 | INFO     | __main__:subgraph_punctuate_node:24 - 子图 subgraph_punctuate_node 节点执行了
============================== ->   原始文本   <- ==============================
   LangGraph 真有意思        
============================== -> 清洗后的文本 <- ==============================
LangGraph 真有意思！
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625094209973.png)

> **策略差异**：三种策略下中断机制行为一致，`Per-invocation` 和 `Per-thread` 均支持中断恢复，`Stateless` 不支持中断。
>

| 策略 | 编译参数 | 中断 |
| :--- | :--- | :--- |
| Per-invocation（默认） | `subgraph = builder.compile()` | ✅ |
| Per-thread | `subgraph = builder.compile(checkpointer=True)` | ✅ |
| Stateless | `subgraph = builder.compile(checkpointer=False)` | ❌ |


#### 12.2.2.2. 场景二：多轮对话
父图多次调用子图进行 LLM 对话，验证子图是否保留历史消息。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver

from langchain.messages import HumanMessage, SystemMessage, AIMessage
from langchain_deepseek import ChatDeepSeek

import sys
from loguru import logger
# 删除 Loguru 默认的 stderr sink
logger.remove()
# 重新添加支持颜色的 sink
logger.add(sys.stdout, colorize=True)

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

# 构建子图
def llm_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    logger.info("=" * 30)
    logger.info("调用子图的 llm_node 节点，当前的 messages: ")
    for index, message in enumerate(messages, start=1):
        logger.opt(colors=True).info(
            "\n<cyan><bold>[消息 {}]</bold></cyan>\n"
            "<yellow>类型：</yellow><magenta>{}</magenta>\n"
            "<yellow>内容：</yellow><green>{}</green>",
            index,
            message.type,
            message.content
        )
    logger.info("=" * 30)

    response = model.invoke(input=messages)
    ai_msg = AIMessage(content=response.content)

    return {
        "messages": [ai_msg]
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", END)

# ==================================================
# 【策略切换点】只需修改这一行
subgraph = builder.compile()  # Per-invocation（默认）：无记忆
# subgraph = builder.compile(checkpointer=True)   # Per-thread：有记忆
# ==================================================

# 构建父图
class OverAllState(TypedDict):
    user_input: str # 用户提问
    assistant_response: str # 助手回答

def call_subgraph(state: OverAllState) -> OverAllState:
    user_input = state["user_input"]

    messages = [
        SystemMessage("尽可能用简短的语言回答"),
        HumanMessage(user_input)
    ]
    subgraph_response = subgraph.invoke(
        {"messages": messages}
    )

    assistant_response = subgraph_response["messages"][-1].content

    return {
        "assistant_response": assistant_response
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("call_subgraph", call_subgraph)
builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "multi-turn-demo"}}

first_invoke = parent_graph.invoke(
    {"user_input": "我是老王，从现在开始，你是小王"},
    config = config
)
print("=" * 30, "-> 第一次调用 <-", "=" * 30)
print(first_invoke)

second_invoke = parent_graph.invoke(
    {"user_input": "我是谁？你是谁？"},
    config = config
)
print("=" * 30, "-> 第二次调用 <-", "=" * 30)
print(second_invoke)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**Per-invocation 运行结果（无记忆）**

```plain
2026-06-25 09:43:14.012 | INFO     | __main__:llm_node:31 - ==============================
2026-06-25 09:43:14.013 | INFO     | __main__:llm_node:32 - 调用子图的 llm_node 节点，当前的 messages: 
2026-06-25 09:43:14.014 | INFO     | __main__:llm_node:34 - 
[消息 1]
类型：system
内容：尽可能用简短的语言回答
2026-06-25 09:43:14.014 | INFO     | __main__:llm_node:34 - 
[消息 2]
类型：human
内容：我是老王，从现在开始，你是小王
2026-06-25 09:43:14.015 | INFO     | __main__:llm_node:42 - ==============================
============================== -> 第一次调用 <- ==============================
{'user_input': '我是老王，从现在开始，你是小王', 'assistant_response': '好的，老王。我是小王，您有什么吩咐？'}
2026-06-25 09:43:14.814 | INFO     | __main__:llm_node:31 - ==============================
2026-06-25 09:43:14.815 | INFO     | __main__:llm_node:32 - 调用子图的 llm_node 节点，当前的 messages: 
2026-06-25 09:43:14.815 | INFO     | __main__:llm_node:34 - 
[消息 1]
类型：system
内容：尽可能用简短的语言回答
2026-06-25 09:43:14.816 | INFO     | __main__:llm_node:34 - 
[消息 2]
类型：human
内容：我是谁？你是谁？
2026-06-25 09:43:14.816 | INFO     | __main__:llm_node:42 - ==============================
============================== -> 第二次调用 <- ==============================
{'user_input': '我是谁？你是谁？', 'assistant_response': '你是用户，我是DeepSeek，一个由深度求索公司开发的AI助手。'}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625094339255.png)

第二次调用时子图 `messages` 被重置，无法获知第一次调用时传递的信息。**Per-invocation 模式无法在子图中进行多轮对话。**

**Per-thread 运行结果（有记忆）**——仅子图编译改为 `builder.compile(checkpointer=True)`：

```plain
2026-06-25 10:42:37.121 | INFO     | __main__:llm_node:31 - ==============================
2026-06-25 10:42:37.122 | INFO     | __main__:llm_node:32 - 调用子图的 llm_node 节点，当前的 messages: 
2026-06-25 10:42:37.123 | INFO     | __main__:llm_node:34 - 
[消息 1]
类型：system
内容：尽可能用简短的语言回答
2026-06-25 10:42:37.124 | INFO     | __main__:llm_node:34 - 
[消息 2]
类型：human
内容：我是老王，从现在开始，你是小王
2026-06-25 10:42:37.124 | INFO     | __main__:llm_node:42 - ==============================
============================== -> 第一次调用 <- ==============================
{'user_input': '我是老王，从现在开始，你是小王', 'assistant_response': '（严肃点头）好，老王，有啥指示？'}
2026-06-25 10:42:37.940 | INFO     | __main__:llm_node:31 - ==============================
2026-06-25 10:42:37.941 | INFO     | __main__:llm_node:32 - 调用子图的 llm_node 节点，当前的 messages: 
2026-06-25 10:42:37.942 | INFO     | __main__:llm_node:34 - 
[消息 1]
类型：system
内容：尽可能用简短的语言回答
2026-06-25 10:42:37.943 | INFO     | __main__:llm_node:34 - 
[消息 2]
类型：human
内容：我是老王，从现在开始，你是小王
2026-06-25 10:42:37.944 | INFO     | __main__:llm_node:34 - 
[消息 3]
类型：ai
内容：（严肃点头）好，老王，有啥指示？
2026-06-25 10:42:37.945 | INFO     | __main__:llm_node:34 - 
[消息 4]
类型：system
内容：尽可能用简短的语言回答
2026-06-25 10:42:37.945 | INFO     | __main__:llm_node:34 - 
[消息 5]
类型：human
内容：我是谁？你是谁？
2026-06-25 10:42:37.946 | INFO     | __main__:llm_node:42 - ==============================
============================== -> 第二次调用 <- ==============================
{'user_input': '我是谁？你是谁？', 'assistant_response': '老王。小王。'}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625104256756.png)

历史消息被保留，上下文连续。**Per-thread 模式适用于子图多轮对话场景。**

> **策略差异**
>

| 策略 | 编译参数 | 多轮记忆 |
| :--- | :--- | :--- |
| Per-invocation（默认） | `subgraph = builder.compile()` | ❌ 每次调用状态重置 |
| Per-thread | `subgraph = builder.compile(checkpointer=True)` | ✅ 跨调用状态连续 |


#### 12.2.2.3. 场景三：多次调用同一子图
父图的一个节点内循环多次调用同一子图。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver

from langchain.messages import HumanMessage, SystemMessage, AIMessage
from langchain_deepseek import ChatDeepSeek

import sys
from loguru import logger
logger.remove()
logger.add(sys.stdout, colorize=True)

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

# 构建子图
def llm_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    logger.info("=" * 30)
    logger.info("调用子图的 llm_node 节点，当前的 messages: ")
    for index, message in enumerate(messages, start=1):
        logger.opt(colors=True).info(
            "\n<cyan><bold>[消息 {}]</bold></cyan>\n"
            "<yellow>类型：</yellow><magenta>{}</magenta>\n"
            "<yellow>内容：</yellow><green>{}</green>",
            index,
            message.type,
            message.content
        )
    logger.info("=" * 30)
    response = model.invoke(input=messages)
    ai_msg = AIMessage(content=response.content)

    return {
        "messages": [ai_msg]
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", END)

# ==================================================
# 【策略切换点】只需修改这一行
subgraph = builder.compile()  # Per-invocation（默认）：每次调用状态隔离
# subgraph = builder.compile(checkpointer=True)   # Per-thread：历史相互干扰
# ==================================================

# 构建父图
class OverAllState(TypedDict):
    user_inputs: list[str] # 用户提问
    assistant_responses: str # 助手回答

def call_subgraph(state: OverAllState) -> OverAllState:
    user_inputs = state["user_inputs"]

    assistant_responses = []
    for user_input in user_inputs:
        subgraph_response = subgraph.invoke(
            {
                "messages": [
                    SystemMessage("用最简短的话回答用户的提问"),
                    HumanMessage(user_input)
                ]
            }
        )
        assistant_response = subgraph_response["messages"][-1].content
        assistant_responses.append(assistant_response)

    return {
        "assistant_responses": assistant_responses
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("call_subgraph", call_subgraph)
builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "multi-call-same-demo"}}

first_response = parent_graph.invoke(
    {
        "user_inputs": [
            "5*5等于几",
            "10*10等于几"
        ]
    },
    config = config
)
print("=" * 30, "-> 第一次运行结果 <-", "=" * 30)
print(first_response)

second_response = parent_graph.invoke(
    {
        "user_inputs": [
            "再+1呢?",
            "再+10000呢?"
        ]
    },
    config = config
)
print("=" * 30, "-> 第二次运行结果 <-", "=" * 30)
print(second_response)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**Per-thread 运行结果**

```plain
2026-07-14 11:53:58.056 | INFO     | __main__:llm_node:29 - ==============================
2026-07-14 11:53:58.056 | INFO     | __main__:llm_node:30 - 调用子图的 llm_node 节点，当前的 messages: 
2026-07-14 11:53:58.057 | INFO     | __main__:llm_node:32 - 
[消息 1]
类型：system
内容：用最简短的话回答用户的提问
2026-07-14 11:53:58.057 | INFO     | __main__:llm_node:32 - 
[消息 2]
类型：human
内容：5*5等于几
2026-07-14 11:53:58.058 | INFO     | __main__:llm_node:40 - ==============================
2026-07-14 11:53:58.652 | INFO     | __main__:llm_node:29 - ==============================
2026-07-14 11:53:58.652 | INFO     | __main__:llm_node:30 - 调用子图的 llm_node 节点，当前的 messages: 
2026-07-14 11:53:58.653 | INFO     | __main__:llm_node:32 - 
[消息 1]
类型：system
内容：用最简短的话回答用户的提问
2026-07-14 11:53:58.653 | INFO     | __main__:llm_node:32 - 
[消息 2]
类型：human
内容：10*10等于几
2026-07-14 11:53:58.654 | INFO     | __main__:llm_node:40 - ==============================
============================== -> 第一次运行结果 <- ==============================
{'user_inputs': ['5*5等于几', '10*10等于几'], 'assistant_responses': ['5*5等于25。', '10*10等于100。']}
2026-07-14 11:53:59.230 | INFO     | __main__:llm_node:29 - ==============================
2026-07-14 11:53:59.231 | INFO     | __main__:llm_node:30 - 调用子图的 llm_node 节点，当前的 messages: 
2026-07-14 11:53:59.231 | INFO     | __main__:llm_node:32 - 
[消息 1]
类型：system
内容：用最简短的话回答用户的提问
2026-07-14 11:53:59.232 | INFO     | __main__:llm_node:32 - 
[消息 2]
类型：human
内容：5*5等于几
2026-07-14 11:53:59.233 | INFO     | __main__:llm_node:32 - 
[消息 3]
类型：ai
内容：5*5等于25。
2026-07-14 11:53:59.234 | INFO     | __main__:llm_node:32 - 
[消息 4]
类型：system
内容：用最简短的话回答用户的提问
2026-07-14 11:53:59.234 | INFO     | __main__:llm_node:32 - 
[消息 5]
类型：human
内容：再+1呢?
2026-07-14 11:53:59.235 | INFO     | __main__:llm_node:40 - ==============================
2026-07-14 11:53:59.766 | INFO     | __main__:llm_node:29 - ==============================
2026-07-14 11:53:59.767 | INFO     | __main__:llm_node:30 - 调用子图的 llm_node 节点，当前的 messages: 
2026-07-14 11:53:59.767 | INFO     | __main__:llm_node:32 - 
[消息 1]
类型：system
内容：用最简短的话回答用户的提问
2026-07-14 11:53:59.768 | INFO     | __main__:llm_node:32 - 
[消息 2]
类型：human
内容：10*10等于几
2026-07-14 11:53:59.768 | INFO     | __main__:llm_node:32 - 
[消息 3]
类型：ai
内容：10*10等于100。
2026-07-14 11:53:59.769 | INFO     | __main__:llm_node:32 - 
[消息 4]
类型：system
内容：用最简短的话回答用户的提问
2026-07-14 11:53:59.769 | INFO     | __main__:llm_node:32 - 
[消息 5]
类型：human
内容：再+10000呢?
2026-07-14 11:53:59.769 | INFO     | __main__:llm_node:40 - ==============================
============================== -> 第二次运行结果 <- ==============================
{'user_inputs': ['再+1呢?', '再+10000呢?'], 'assistant_responses': ['25+1=26。', '10100']}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625094551009.png)

每次调用子图时 `messages` 状态都是独立的。**Per-invocation 适用于父图中多次调用同一子图的场景，调用间互不干扰。**

**Per-thread 行为**（改为 `builder.compile(checkpointer=True)`）：同一节点内的多次子图调用，历史按调用位置独立——即第一次子图调用的历史是连续的，第二次子图调用的历史也是连续的，但两次之间独立。**如果调用顺序发生变化，检查点历史会相互干扰。**

> **策略差异**
>

| 策略 | 编译参数 | 多次调用同一子图 |
| :--- | :--- | :--- |
| Per-invocation（默认） | `subgraph = builder.compile()` | ✅ 每次调用完全隔离 |
| Per-thread | `subgraph = builder.compile(checkpointer=True)` | ❌ 历史按位置关联，可能干扰 |


#### 12.2.2.4. 场景四：多次调用不同子图
父图的一个节点内调用两个不同的子图。

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver

from langchain.messages import HumanMessage, SystemMessage, AIMessage
from langchain_deepseek import ChatDeepSeek

import sys
from loguru import logger
logger.remove()
logger.add(sys.stdout, colorize=True)

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

# ==================================================
# 【策略切换点】两个子图的编译参数
# ==================================================

# 构建水果子图
def fruit_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    logger.info("=" * 30)
    logger.info("调用水果子图的 fruit_node 节点，当前的 messages: ")
    for index, message in enumerate(messages, start=1):
        logger.opt(colors=True).info(
            "\n<cyan><bold>[消息 {}]</bold></cyan>\n"
            "<yellow>类型：</yellow><magenta>{}</magenta>\n"
            "<yellow>内容：</yellow><green>{}</green>",
            index,
            message.type,
            message.content
        )
    logger.info("=" * 30)
    response = model.invoke(input=messages)
    ai_msg = AIMessage(content=response.content)

    return {
        "messages": [ai_msg]
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("fruit_node", fruit_node)
builder.add_edge(START, "fruit_node")
builder.add_edge("fruit_node", END)

fruit_subgraph = builder.compile(checkpointer=True)

# 构建蔬菜子图
def vegetable_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    logger.info("=" * 30)
    logger.info("调用蔬菜子图的 vegetable_node 节点，当前的 messages: ")
    for index, message in enumerate(messages, start=1):
        logger.opt(colors=True).info(
            "\n<cyan><bold>[消息 {}]</bold></cyan>\n"
            "<yellow>类型：</yellow><magenta>{}</magenta>\n"
            "<yellow>内容：</yellow><green>{}</green>",
            index,
            message.type,
            message.content
        )
    logger.info("=" * 30)
    response = model.invoke(input=messages)
    ai_msg = AIMessage(content=response.content)

    return {
        "messages": [ai_msg]
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("vegetable_node", vegetable_node)
builder.add_edge(START, "vegetable_node")
builder.add_edge("vegetable_node", END)

vegetable_subgraph = builder.compile(checkpointer=True)

# 构建父图
class OverAllState(TypedDict):
    fruit: str # 水果
    vegetable: str # 蔬菜

    fruit_introduction: str # 水果介绍
    vegetable_introduction: str # 蔬菜介绍

def call_subgraph(state: OverAllState) -> OverAllState:
    fruit = state["fruit"]
    vegetable = state["vegetable"]

    fruit_messages = [
        SystemMessage("用最简短的话介绍用户输入的水果"),
        HumanMessage(fruit)
    ]

    vegetable_messages = [
        SystemMessage("用最简短的话介绍用户输入的蔬菜"),
        HumanMessage(vegetable)
    ]

    fruit_response = fruit_subgraph.invoke({"messages": fruit_messages})
    vegetable_response = vegetable_subgraph.invoke({"messages": vegetable_messages})

    fruit_introduction = fruit_response["messages"][-1].content
    vegetable_introduction = vegetable_response["messages"][-1].content

    return {
        "fruit_introduction": fruit_introduction,
        "vegetable_introduction": vegetable_introduction
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("call_subgraph", call_subgraph)
builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "multi-call-diff-demo"}}

first_response = parent_graph.invoke(
    {
        "fruit": "桑葚",
        "vegetable": "西兰苔"
    },
    config = config
)
print("=" * 30, "-> 第一次运行结果 <-", "=" * 30)
print(first_response)

second_response = parent_graph.invoke(
    {
        "fruit": "香蕉",
        "vegetable": "西兰花"
    },
    config = config
)
print("=" * 30, "-> 第二次运行结果 <-", "=" * 30)
print(second_response)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**运行结果**

```plain
2026-06-25 11:37:49.176 | INFO     | __main__:fruit_node:32 - ==============================
2026-06-25 11:37:49.176 | INFO     | __main__:fruit_node:33 - 调用水果子图的 fruit_node 节点，当前的 messages: 
2026-06-25 11:37:49.177 | INFO     | __main__:fruit_node:35 - 
[消息 1]
类型：system
内容：用最简短的话介绍用户输入的水果
2026-06-25 11:37:49.178 | INFO     | __main__:fruit_node:35 - 
[消息 2]
类型：human
内容：桑葚
2026-06-25 11:37:49.178 | INFO     | __main__:fruit_node:43 - ==============================
2026-06-25 11:37:49.872 | INFO     | __main__:vegetable_node:61 - ==============================
2026-06-25 11:37:49.873 | INFO     | __main__:vegetable_node:62 - 调用蔬菜子图的 vegetable_node 节点，当前的 messages: 
2026-06-25 11:37:49.873 | INFO     | __main__:vegetable_node:64 - 
[消息 1]
类型：system
内容：用最简短的话介绍用户输入的蔬菜
2026-06-25 11:37:49.874 | INFO     | __main__:vegetable_node:64 - 
[消息 2]
类型：human
内容：西兰苔
2026-06-25 11:37:49.874 | INFO     | __main__:vegetable_node:72 - ==============================
============================== -> 第一次运行结果 <- ==============================
{'fruit': '桑葚', 'vegetable': '西兰苔', 'fruit_introduction': '紫黑多汁的浆果，味酸甜。', 'vegetable_introduction': '西兰苔是西兰花与芥蓝杂交的蔬菜，以鲜嫩花茎和球花为食。'}
...（第二次调用时水果、蔬菜子图各自保留历史消息，按调用位置独立）
{'fruit': '香蕉', 'vegetable': '西兰花', 'fruit_introduction': '黄色弯月形水果，软糯香甜。', 'vegetable_introduction': '西兰花，绿色花球状蔬菜，营养丰富。'}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625113908367.png)

本例在同一个节点中多次调用不同子图，通过专有状态保证了**水果一定发给 **`fruit_subgraph`、**蔬菜一定发给 **`vegetable_subgraph`，一定程度上可以确保不同子图的检查点历史不相互干扰。

但这种靠调用顺序区分的方式不够稳健——如果调用顺序变化，检查点历史也会错乱。官方推荐的做法见下一节。

#### 12.2.2.5. 场景五：独立节点调用不同子图（推荐）
为每个子图准备独立的父图节点，这样它们的命名空间是各自的节点名，完全独立。

```python
from typing import TypedDict, Literal
from collections.abc import Sequence
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver

from langchain.messages import HumanMessage, SystemMessage, AIMessage
from langchain_deepseek import ChatDeepSeek

import sys
from loguru import logger
logger.remove()
logger.add(sys.stdout, colorize=True)

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

# 构建水果子图
def fruit_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    logger.info("=" * 30)
    logger.info("调用水果子图的 fruit_node 节点，当前的 messages: ")
    for index, message in enumerate(messages, start=1):
        logger.opt(colors=True).info(
            "\n<cyan><bold>[消息 {}]</bold></cyan>\n"
            "<yellow>类型：</yellow><magenta>{}</magenta>\n"
            "<yellow>内容：</yellow><green>{}</green>",
            index,
            message.type,
            message.content
        )
    logger.info("=" * 30)
    response = model.invoke(input=messages)
    ai_msg = AIMessage(content=response.content)

    return {
        "messages": [ai_msg]
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("fruit_node", fruit_node)
builder.add_edge(START, "fruit_node")
builder.add_edge("fruit_node", END)

fruit_subgraph = builder.compile(checkpointer=True)

# 构建蔬菜子图
def vegetable_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    logger.info("=" * 30)
    logger.info("调用蔬菜子图的 vegetable_node 节点，当前的 messages: ")
    for index, message in enumerate(messages, start=1):
        logger.opt(colors=True).info(
            "\n<cyan><bold>[消息 {}]</bold></cyan>\n"
            "<yellow>类型：</yellow><magenta>{}</magenta>\n"
            "<yellow>内容：</yellow><green>{}</green>",
            index,
            message.type,
            message.content
        )
    logger.info("=" * 30)
    response = model.invoke(input=messages)
    ai_msg = AIMessage(content=response.content)

    return {
        "messages": [ai_msg]
    }

builder = StateGraph(state_schema=MessagesState)
builder.add_node("vegetable_node", vegetable_node)
builder.add_edge(START, "vegetable_node")
builder.add_edge("vegetable_node", END)

vegetable_subgraph = builder.compile(checkpointer=True)

# 构建父图
class OverAllState(TypedDict):
    fruit: str # 水果
    vegetable: str # 蔬菜

    fruit_introduction: str # 水果介绍
    vegetable_introduction: str # 蔬菜介绍

def call_fruit_subgraph(state: OverAllState) -> OverAllState:
    fruit = state["fruit"]

    fruit_messages = [
        SystemMessage("用最简短的话介绍用户输入的水果"),
        HumanMessage(fruit)
    ]

    fruit_response = fruit_subgraph.invoke({"messages": fruit_messages})
    fruit_introduction = fruit_response["messages"][-1].content

    return {
        "fruit_introduction": fruit_introduction
    }

def call_vegetable_subgraph(state: OverAllState) -> OverAllState:
    vegetable = state["vegetable"]

    vegetable_messages = [
        SystemMessage("用最简短的话介绍用户输入的蔬菜"),
        HumanMessage(vegetable)
    ]

    vegetable_response = vegetable_subgraph.invoke({"messages": vegetable_messages})
    vegetable_introduction = vegetable_response["messages"][-1].content

    return {
        "vegetable_introduction": vegetable_introduction
    }

def router(state: OverAllState) -> Sequence[Literal["call_fruit_subgraph", "call_vegetable_subgraph", END]]:
    next_nodes = []
    if state.get("fruit"):
        next_nodes.append("call_fruit_subgraph")
    if state.get("vegetable"):
        next_nodes.append("call_vegetable_subgraph")
    if not next_nodes:
        next_nodes.append(END)
    return next_nodes

builder = StateGraph(state_schema=OverAllState)
builder.add_node("call_fruit_subgraph", call_fruit_subgraph)
builder.add_node("call_vegetable_subgraph", call_vegetable_subgraph)

builder.add_conditional_edges(START, router, path_map=["call_fruit_subgraph", "call_vegetable_subgraph", END])
builder.add_edge("call_fruit_subgraph", END)
builder.add_edge("call_vegetable_subgraph", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "independent-nodes-demo"}}

first_response = parent_graph.invoke(
    {
        "fruit": "桑葚",
        "vegetable": "西兰苔"
    },
    config = config
)
print("=" * 30, "-> 第一次运行结果 <-", "=" * 30)
print(first_response)

second_response = parent_graph.invoke(
    {
        "fruit": "香蕉",
        "vegetable": "西兰花"
    },
    config = config
)
print("=" * 30, "-> 第二次运行结果 <-", "=" * 30)
print(second_response)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**运行结果**

```plain
2026-06-25 11:59:10.331 | INFO     | __main__:fruit_node:33 - ==============================
2026-06-25 11:59:10.334 | INFO     | __main__:fruit_node:34 - 调用水果子图的 fruit_node 节点，当前的 messages: 
2026-06-25 11:59:10.335 | INFO     | __main__:fruit_node:36 - 
[消息 1]
类型：system
内容：用最简短的话介绍用户输入的水果
2026-06-25 11:59:10.334 | INFO     | __main__:vegetable_node:62 - ==============================
2026-06-25 11:59:10.335 | INFO     | __main__:fruit_node:36 - 
[消息 2]
类型：human
内容：桑葚
2026-06-25 11:59:10.337 | INFO     | __main__:fruit_node:44 - ==============================
2026-06-25 11:59:10.337 | INFO     | __main__:vegetable_node:63 - 调用蔬菜子图的 vegetable_node 节点，当前的 messages: 
2026-06-25 11:59:10.341 | INFO     | __main__:vegetable_node:65 - 
[消息 1]
类型：system
内容：用最简短的话介绍用户输入的蔬菜
2026-06-25 11:59:10.341 | INFO     | __main__:vegetable_node:65 - 
[消息 2]
类型：human
内容：西兰苔
2026-06-25 11:59:10.342 | INFO     | __main__:vegetable_node:73 - ==============================
============================== -> 第一次运行结果 <- ==============================
{'fruit': '桑葚', 'vegetable': '西兰苔', 'fruit_introduction': '桑葚是桑树的果实，味甜可食，常被用于制酱或入药。', 'vegetable_introduction': '西兰苔是西兰花和芥蓝的杂交品种。'}
...（第二次调用时，水果子图和蔬菜子图各自保留独立的历史）
{'fruit': '香蕉', 'vegetable': '西兰花', 'fruit_introduction': '香蕉是软甜水果，富含钾，能缓解便秘。', 'vegetable_introduction': '一种绿色花状蔬菜，营养丰富。'}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625115935316.png)

每个子图有独立的父图节点，命名空间为各自的节点名，完全隔离。**这是 Per-thread 模式下多次调用不同子图的推荐做法。**

#### 12.2.2.6. 场景六：Stateless 模式
子图完全无状态，不保存检查点，中断和多轮对话均不适用。适用于纯计算、无需追踪历史的场景。

```python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

# 构建子图
class SubgraphState(TypedDict):
    raw_text: str
    clean_texts: Annotated[list[str], add]

def strip_node(state: SubgraphState) -> SubgraphState:
    raw_text = state["raw_text"]
    clean_text = raw_text.strip()

    return {
        "clean_texts": [clean_text]
    }

builder = StateGraph(state_schema=SubgraphState)
builder.add_node("strip_node", strip_node)
builder.add_edge(START, "strip_node")
builder.add_edge("strip_node", END)

# ==================================================
# 【策略切换点】只需修改这一行
subgraph = builder.compile(checkpointer=False)  # Stateless：无检查点
# subgraph = builder.compile(checkpointer=True)   # Per-thread：有状态（对比用）
# ==================================================

# 构建父图
class ParentState(TypedDict):
    input_texts: list[str]
    output_texts: Annotated[list[str], add]

def call_subgraph(state: ParentState) -> ParentState:
    input_texts = state["input_texts"]

    output_texts = []
    for input_text in input_texts:
        res = subgraph.invoke({"raw_text": input_text})
        output_texts += res["clean_texts"]

    return {
        "output_texts": output_texts
    }

builder = StateGraph(state_schema=ParentState)
builder.add_node("call_subgraph", call_subgraph)
builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "stateless-demo"}}

first_response = parent_graph.invoke(
    {
        "input_texts": ["  LangGraph 真有意思 ", "  我喜欢 LangChain "]
    },
    config = config
)
print("=" * 30, "-> 第一次运行结果 <-", "=" * 30)
print(first_response)

second_response = parent_graph.invoke(
    {
        "input_texts": ["  Hello, LangGraph  ", "  Hello, LangChain "]
    },
    config = config
)
print("=" * 30, "-> 第二次运行结果 <-", "=" * 30)
print(second_response)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**Stateless 运行结果**

```plain
============================== -> 第一次运行结果 <- ==============================
{'input_texts': ['  LangGraph 真有意思 ', '  我喜欢 LangChain '], 'output_texts': ['LangGraph 真有意思', '我喜欢 LangChain']}
============================== -> 第二次运行结果 <- ==============================
{'input_texts': ['  Hello, LangGraph  ', '  Hello, LangChain '], 'output_texts': ['LangGraph 真有意思', '我喜欢 LangChain', 'Hello, LangGraph', 'Hello, LangChain']}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625121810977.png)

渲染的图结构中，子图节点内部没有展开子图拓扑——无状态模式的子图内部状态不可访问。

**Per-thread 对比结果**（仅改为 `builder.compile(checkpointer=True)`）：

```plain
============================== -> 第一次运行结果 <- ==============================
{'input_texts': ['  LangGraph 真有意思 ', '  我喜欢 LangChain '], 'output_texts': ['LangGraph 真有意思', '我喜欢 LangChain']}
============================== -> 第二次运行结果 <- ==============================
{'input_texts': ['  Hello, LangGraph  ', '  Hello, LangChain '], 'output_texts': ['LangGraph 真有意思', '我喜欢 LangChain', 'LangGraph 真有意思', 'Hello, LangGraph', '我喜欢 LangChain', 'Hello, LangChain']}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625122230644.png)

Per-thread 模式下第二次调用结果出现了重复（子图的历史 `clean_texts` 被累积），拓扑结构中也展开了子图。**Stateless 不会累积历史，每次调用都是干净的。**

#### 12.2.2.7. `Per-invocation`和`Per-thread`的设计哲学
默认子图检查点命名空间为 `<节点名称>:<任务ID>`。两种模式的核心差异在于对 `<任务ID>` 的处理：

| 维度 | `Per-invocation` | `Per-thread` |
| :--- | :--- | :--- |
| 命名空间 | `<节点名称>:<任务ID>` | `<节点名称>`（去掉 `:<任务ID>`） |
| 设计意图 | `<任务ID>` 每次不同 → 每次调用状态独立 | `<节点名称>` 不变 → 跨多次调用状态连续 |
| 同节点多次调用 | `:<任务ID>|1`, `:<任务ID>|2` … | `|1`, `|2` … |
| 注意事项 | — | 调用顺序变化会导致检查点历史相互干扰；`get_state_history()` 存在 Bug，父图检查点仍记录 `:<任务ID>` 格式，需额外处理 |


#### 12.2.2.8. 总结
| 特性 | `Per-invocation (default)` | `Per-thread` | `Stateless` |
| :--- | :--- | :--- | :--- |
| `checkpointer=` | `None` | `True` | `False` |
| 中断 | ✅ | ✅ | ❌ |
| 多轮对话 | ❌ | ✅ | ❌ |
| 多次调用不同子图 | ✅ | ⚠️最好通过节点隔离 | ✅ |
| 多次调用相同子图 | ✅ | ❌子图历史相互干扰 | ✅ |
| 观测子图检查点快照 | ⚠️可以观测，但每次调用独立 | ✅ | ❌ |


## 12.3. 子图流式运行
只需要在父图调用 `stream` 时传递 `subgraphs=True` 即可，这样父图和子图的流数据都会汇入同一个流队列，再通过命名空间区分来源。

**需要注意的是**：`subgraphs=True` 是在原有父图流数据的基础上，**额外加入子图内部产生的流数据**，并不会用子图数据替代父图数据。它会同时流式返回父图和所有子图的输出。

### 12.3.1. 分析`chunk`格式
`chunk` 格式为

```python
(namespace, stream_mode, data)
```

此处的命名空间不同于检查点命名空间，是流处理内部专门用于区分父图和子图流数据的字段。

我们通过模式为 `updates` 的简单案例查看数据格式

**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# 构建子图
class SubgraphState(TypedDict):
    raw_text: str # 未清洗文本
    stripped_text: str # 去除收尾空格的文本
    punctuated_text: str # 句尾添加句号的文本

def subgraph_strip_node(state: SubgraphState) -> SubgraphState:
    raw_text = state["raw_text"]
    stripped_text = raw_text.strip()

    return {
        "stripped_text": stripped_text
    }

def subgraph_punctuate_node(state: SubgraphState) -> SubgraphState:
    stripped_text = state["stripped_text"]
    punctuated_text = stripped_text + "。"

    return {
        "punctuated_text": punctuated_text
    }

builder = StateGraph(state_schema=SubgraphState)
builder.add_node("subgraph_strip_node", subgraph_strip_node)
builder.add_node("subgraph_punctuate_node", subgraph_punctuate_node)

builder.add_edge(START, "subgraph_strip_node")
builder.add_edge("subgraph_strip_node", "subgraph_punctuate_node")
builder.add_edge("subgraph_punctuate_node", END)

subgraph = builder.compile()

# 构建父图
class ParentState(TypedDict):
    input_text: str # 输入的未清洗的文本
    cleaned_text: str # 清洗后的文本

def call_subgraph(state: ParentState) -> ParentState:
    input_text = state["input_text"]

    res = subgraph.invoke({"raw_text": input_text})
    cleaned_text = res["punctuated_text"]
    return {
        "cleaned_text": cleaned_text
    }

builder = StateGraph(state_schema=ParentState)
builder.add_node("call_subgraph", call_subgraph)

builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

parent_graph = builder.compile()

input_text = "   LangGraph 真有意思        "
for chunk in parent_graph.stream(
    {"input_text": input_text},
    subgraphs=True,
    stream_mode=["updates"]
):
    print(chunk)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**运行结果如下**

```python
(
    (
        'call_subgraph:4daa7290-d5c9-ae96-849e-d2f1dafadd17',
    ),
    'updates',
    {
        'subgraph_strip_node': {
            'stripped_text': 'LangGraph 真有意思'
        }
    }
)
(
    (
        'call_subgraph:4daa7290-d5c9-ae96-849e-d2f1dafadd17',
    ),
    'updates',
    {
        'subgraph_punctuate_node': {
            'punctuated_text': 'LangGraph 真有意思。'
        }
    }
)
(
    (),
    'updates',
    {
        'call_subgraph': {
            'cleaned_text': 'LangGraph 真有意思。'
        }
    }
)
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625143710651.png)

**流数据命名空间规则**：

| 来源 | 命名空间 | 备注 |
| :--- | :--- | :--- |
| 父图 | `()`（空元组） |  |
| 子图（常规） | `(<父图节点名>:<父图任务ID>,)` |  |
| 子图（Per-thread） | `(<父图节点名>,)` | 与检查点命名空间一致 |


### 12.3.2. 多轮对话消息流处理
**示例如下**

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langchain.messages import HumanMessage, SystemMessage
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

# 构建子图
class SubgraphState(MessagesState):
    system_prompt: SystemMessage

def llm_mode(state: SubgraphState) -> SubgraphState:
    system_prompt = state["system_prompt"]
    messages = state["messages"]

    response = model.invoke([system_prompt] + messages)
    return {
        "messages": [response]
    }

builder = StateGraph(state_schema=SubgraphState)
builder.add_node("llm_mode", llm_mode)
builder.add_edge(START, "llm_mode")
builder.add_edge("llm_mode", END)

subgraph = builder.compile(checkpointer=True)

# 构建父图
class ParentState(TypedDict):
    user_input: str
    assistant_response: str

def call_subgraph(state: ParentState) -> ParentState:
    user_input = state["user_input"]

    response = subgraph.invoke(
        {
            "system_prompt": SystemMessage("你是个善解人意的助手"),
            "messages": [HumanMessage(user_input)]
        }
    )

    assistant_response = response["messages"][-1].content

    return {
        "assistant_response": assistant_response
    }

builder = StateGraph(state_schema=ParentState)
builder.add_node("call_subgraph", call_subgraph)
builder.add_edge(START, "call_subgraph")
builder.add_edge("call_subgraph", END)

checkpointer = InMemorySaver()
parent_graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "subgraph_stream_test"}}

print("\n", "=" * 30, "-> 第一次运行结果 <-", "=" * 30)
# chunk 形如
# (('call_subgraph',), 'messages', (AIMessageChunk(content='', additional_kwargs={}, response_metadata={'model_provider': 'deepseek'}, id='lc_run--019efd65-c54e-7012-912e-29ff4f84328e', tool_calls=[], invalid_tool_calls=[], tool_call_chunks=[]), {'thread_id': 'subgraph_stream_test', 'langgraph_step': 1, 'langgraph_node': 'llm_mode', 'langgraph_triggers': ('branch:to:llm_mode',), 'langgraph_path': ('__pregel_pull', 'llm_mode'), 'langgraph_checkpoint_ns': 'call_subgraph|llm_mode:55577c3e-22da-877a-c77b-f16bdbcd2c6c', 'checkpoint_ns': 'call_subgraph:9c469391-f094-1c1b-c5d7-6578f1603151', 'ls_provider': 'deepseek', 'ls_model_name': 'deepseek-v4-flash', 'ls_model_type': 'chat', 'ls_temperature': None}))
for chunk in parent_graph.stream(
    {
        "user_input": "花儿为什么这样红？"
    },
    config=config,
    subgraphs=True,
    stream_mode=["messages"]
):
    print(chunk[2][0].content, end="", flush=True)

print("\n", "=" * 30, "-> 第二次运行结果 <-", "=" * 30)
for chunk in parent_graph.stream(
    {
        "user_input": "刚才我们聊了什么？"
    },
    config=config,
    subgraphs=True,
    stream_mode=["messages"]
):
    print(chunk[2][0].content, end="", flush=True)

from IPython.display import display, Image
display(
    Image(
        parent_graph
        .get_graph(xray=True)
        .draw_mermaid_png()
    )
)
```

**运行结果如下**

```python
 ============================== -> 第一次运行结果 <- ==============================
“花儿为什么这样红”是一个经典的问题，可以从不同角度来理解：

1. **自然科学的视角**：花朵的颜色主要由花瓣中的色素决定，尤其是**花青素**（anthocyanins）。花青素在酸性环境下呈现红色，在碱性环境下偏向蓝色。某些花朵（如玫瑰、山茶花）细胞液偏酸性，因此吸收阳光中的蓝绿光，反射红光，让我们看到鲜艳的红色。此外，红色也能吸引蜂鸟等传粉动物，帮助植物繁殖。

2. **文化艺术的视角**：这是一首传唱已久的中国民歌（《花儿为什么这样红》），由雷振邦作曲，是电影《冰山上的来客》的插曲。歌词用“花儿”比喻纯洁、热烈的情感，红色象征着爱情、勇气或革命的赤诚。

3. **诗意隐喻**：就像“花儿为什么这样红”可以指代生命的热情、青春的绽放，或者经风雨后更加绚烂的坚韧。红色的花朵也常被赋予“美好而短暂”的哲学意味。

如果你指的是特定语境（比如学生物的朋友、教孩子、写赏析），我可以把回答调整得更具体。🌹
 ============================== -> 第二次运行结果 <- ==============================
我们刚才聊了“花儿为什么这样红”这个话题。我分别从自然科学（花青素与酸碱性）、文化艺术（经典民歌《花儿为什么这样红》）以及诗意隐喻三个角度做了解答。如果你有更多想深入的方向，随时告诉我～
```

实际测试可以看到模型返回的消息是 **逐**`token` 返回的。

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625143740875.png)

## 12.4. 子图动态路由
在子图中可以通过 `Command` 动态路由到父图节点，只要将参数 `graph` 的值设置为 `Command.PARENT` 即可，如下所示

```python
return Command(
    update={
        "fruits": fruits,
        "vegetables": vegetables
    },
    goto="router_node",
    graph=Command.PARENT
)
```

 **示例如下**

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
from loguru import logger

# ==================== 子图 ====================
class SubState(TypedDict):
    data: str

def sub_node(state: SubState) -> Command:
    """子图节点：执行后动态路由回父图的 parent_router"""
    logger.info("[子图] sub_node 执行")
    return Command(
        update={"data": state["data"] + " → 子图"},
        goto="node_b",       # 路由到父图节点
        graph=Command.PARENT,       # 指定目标为父图
    )

sub_builder = StateGraph(state_schema=SubState)
sub_builder.add_node("sub_node", sub_node)
sub_builder.add_edge(START, "sub_node")
sub_graph = sub_builder.compile()

# ==================== 父图 ====================
class ParentState(TypedDict):
    data: str

def node_a(state: ParentState) -> ParentState:
    print("[父图] node_a 执行")
    return {"visited_a": True}

def node_b(state: ParentState) -> ParentState:
    print("[父图] node_b 执行")
    return {"visited_b": True}

parent_builder = StateGraph(state_schema=ParentState)
parent_builder.add_node(
    "sub_graph", sub_graph
)

parent_builder.add_node("node_a", node_a)
parent_builder.add_node("node_b", node_b)
parent_builder.add_edge(START, "sub_graph")
parent_builder.add_edge("node_a", "node_b")
parent_builder.add_edge("node_b", END)

parent_graph = parent_builder.compile()

result = parent_graph.invoke({"data": "初始"})
print(f"\n最终结果: {result}")

from IPython.display import display
display(parent_graph)
```

**要点**：

1. 子图通过 `Command(goto="parent_router", graph=Command.PARENT)` 将执行流动态路由回父图的指定节点
2. 子图返回值类型注解不能用 `Command[Literal["parent_router"]]` 的形式，因为子图并不知道父图节点，静态类型校验会报错

```python
ValueError: Found edge ending at unknown node `parent_router`
```

3. 添加子图节点时通过 `destinations` 声明下游节点，便于渲染图结构时正确描绘拓扑关系

```python
builder.add_node(
    "sub_graph", sub_graph,
    destinations=("parent_router",)
)
```

不加 `destinations` 不会影响动态路由，只是渲染的图结构不能正确描绘拓扑。

# 13. 运行图设计模式
## 13.1. 概述
`LangGraph` 官方总结了常用的状态图设计模式，如下

| 模式 | 图结构 | 运行时动态性 | 核心 `LangGraph` 能力 |
| --- | --- | --- | --- |
| `Prompt Chaining` | 顺序链 | 低 | 静态边、条件边 |
| `Parallelization` | 固定 `Fan-out/Fan-in` | 低 | 并行超步、汇聚 |
| `Routing` | 条件分支 | 中 | 结构化输出、条件边 |
| `Orchestrator-worker` | 动态 `Fan-out/Fan-in` | 高 | `Send`、`WorkerState`、`Reducer` |
| `Evaluator-optimizer` | 反馈循环 | 中 | 条件边、循环、反馈状态 |
| `Agent` | 自主决策循环 | 最高 | `MessagesState`、工具调用、`ToolNode` |


## 13.2. 实现
### 13.2.1. `Prompt Chaining`：提示词链
#### 13.2.1.1. 核心思想
将一个复杂任务拆分成若干个顺序执行的小任务：

```latex
输入
 ↓
任务 A
 ↓
任务 B
 ↓
任务 C
 ↓
输出
```

后一个节点依赖前一个节点的结果。

适合：

+ 翻译 → 校对 → 润色
+ 生成内容 → 检查一致性 → 修订
+ 提取信息 → 分类 → 格式化
+ 需求分析 → 生成代码 → 代码解释

官方示例是：

```latex
生成笑话
   ↓
检查是否合格
  ├─ 合格 → END
  └─ 不合格 → 改进笑话 → 最终润色 → END
```

所以 `Prompt Chaining` 不一定只是简单的直线，也可以在中间加入一个 `Gate`**，质量门控节点**。

`Prompt Chaining` 的核心是：

> **将复杂任务拆解为可独立验证的阶段，并把每一阶段的结果记录下来向后传递。**
>

它的优点是执行过程稳定且易于调试；缺点是流程固定，无法处理未知数量或高度动态的任务。

#### 13.2.1.2. 示例
**示例如下**

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

# 图状态
class OverAllState(TypedDict):
    topic: str
    joke: str
    improved_joke: str
    final_joke: str

# 节点
def generate_joke(state: OverAllState) -> OverAllState:
    """第一次调用大模型，生成初始笑话"""

    msg = model.invoke(
        [HumanMessage(content=f"写一个关于“{state['topic']}”的简短笑话")]
    )
    return {"joke": msg.content}

def check_punchline(state: OverAllState) -> Literal["pass", "no_pass"]:
    """判断笑话中是否包含包袱"""

    # 简单判断：笑话中是否包含问号或感叹号
    if "?" in state["joke"] or "!" in state["joke"]:
        return "pass"
    return "no_pass"

def improve_joke(state: OverAllState) -> OverAllState:
    """第二次调用大模型，通过添加双关语改进笑话"""

    msg = model.invoke(
        [HumanMessage(content=f"通过添加双关语，让下面这个笑话变得更有趣：\n{state['joke']}")]
    )
    return {"improved_joke": msg.content}

def polish_joke(state: OverAllState) -> OverAllState:
    """第三次调用大模型，对笑话进行最终润色"""

    msg = model.invoke(
        [HumanMessage(content=f"为下面这个笑话添加一个出人意料的反转：\n{state['improved_joke']}")]
    )
    return {"final_joke": msg.content}

# 构建工作流
builder = StateGraph(state_schema=OverAllState)

# 添加节点
builder.add_node("generate_joke", generate_joke)
builder.add_node("improve_joke", improve_joke)
builder.add_node("polish_joke", polish_joke)

# 添加边，连接各个节点
builder.add_edge(START, "generate_joke")
builder.add_conditional_edges(
    "generate_joke",
    check_punchline,
    {
        "no_pass": "improve_joke",
        "pass": END,
    },
)
builder.add_edge("improve_joke", "polish_joke")
builder.add_edge("polish_joke", END)

# 编译工作流
graph = builder.compile()

# 调用工作流
response = graph.invoke({"topic": "猫"})
print(response)

# 绘制拓扑结构
from IPython.display import display
display(graph)
```

**运行结果如下**

```python
{
    'topic': '猫',
    'joke': '一只猫去面试，面试官问：“你有什么特长？”  \n猫淡定地回答：“我能用尾巴画圆。”  \n面试官惊讶：“真的吗？演示一下。”  \n猫转身，尾巴甩了几下，画出一个完美的圆。  \n面试官鼓掌：“太好了！你被录取了！”  \n第二天，猫上班了，发现它被送到了数学实验室——负责画π。',
    'improved_joke': '加了一点数学梗和双关后，版本如下：\n\n---\n\n一只猫去面试，面试官问：“你有什么特长？”  \n猫淡定地回答：“我能用尾巴画圆。”  \n面试官惊讶：“真的吗？演示一下。”  \n猫转身，尾巴甩了几下，画出一个完美的圆。  \n面试官鼓掌：“太好了！你被录取了！”  \n\n第二天，猫上班了，发现它被送到了数学实验室——**负责画π**。  \n\n猫叹了口气，说：“**原来我的‘圆’满人生，终究逃不过π的‘鼠’命。”**  \n\n（双关解析：  \n1. “π”音同“派”，既指圆周率，也暗示“被派遣”去画圆。  \n2. “鼠命”谐音“宿命”，同时猫抓老鼠，这里暗指猫被困在数学“鼠”洞（无限不循环小数）里，永远画不完。）',
    'final_joke': '---\n\n**反转版：**  \n第二天，猫上班了，发现它被送到了数学实验室——**负责画π**。  \n\n猫叹了口气，说：“原来我的‘圆’满人生，终究逃不过π的‘鼠’命。”  \n\n话音刚落，实验室主任推门进来，是个戴眼镜的仓鼠。仓鼠拍了拍猫的尾巴，微笑道：“别担心，我们用的是**有理数**——你每天只用画前100位，剩下的**无限循环**，由我们**鼠**类负责演算。”  \n\n猫愣了一下，突然跳起来：“等等！你是说——我画圆，你们算剩下的尾巴？”  \n\n仓鼠点头：“对，这叫……**π的‘分工’合作**。”  \n\n猫沉默三秒，忽然掏出手机：“那我得先给抓老鼠的同行打个电话——告诉他们，**‘有理’走遍天下，无理寸步难行**。”  \n\n（双关补充：  \n1. **“有理数”** 双关：既指数学上有理数（可终止或循环），也暗示“有道理的反转”——猫的“无限宿命”被仓鼠强行打断。  \n2. **“剩下的尾巴”** 双关：猫的尾巴画圆，但仓鼠们负责“补完”无限小数，字面与数学双关。  \n3. **“有理走遍天下”** 化用俗语，暗讽“有理数”才是猫的出路，而“无理”的π已变成职场协作梗。）'
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625180419653.png)

---

### 13.2.2. `Parallelization`：并行化
#### 13.2.2.1. 核心思想
把相互独立的任务同时执行，最后汇总结果：

```latex
                 ┌→ 任务 A ─┐
输入 → Fan-out ──┼→ 任务 B ─┼→ 聚合 → 输出
                 └→ 任务 C ─┘
```

官方将并行化分成两种用途：

1. **任务拆分**
    - 一个节点检查关键词
    - 一个节点检查格式
    - 一个节点检查事实准确性
2. **多次独立判断**
    - 多个模型或多个提示词分别评分
    - 最后投票、平均或综合判断

前者主要提升速度，后者主要提升置信度。

#### 13.2.2.2. 示例
**示例如下**

```python
from typing import TypedDict
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

# 图状态
class OverAllState(TypedDict):
    topic: str
    joke: str
    story: str
    poem: str
    combined_output: str

# 节点
def call_model_1(state: OverAllState) -> OverAllState:
    """第一次调用大模型，生成笑话"""

    msg = model.invoke(
        [HumanMessage(content=f"写一个关于“{state['topic']}”的简短的笑话")]
    )
    return {"joke": msg.content}

def call_model_2(state: OverAllState) -> OverAllState:
    """第二次调用大模型，生成故事"""

    msg = model.invoke(
        [HumanMessage(content=f"写一个关于“{state['topic']}”的简短的故事")]
    )
    return {"story": msg.content}

def call_model_3(state: OverAllState) -> OverAllState:
    """第三次调用大模型，生成诗歌"""

    msg = model.invoke(
        [HumanMessage(content=f"写一首关于“{state['topic']}”的简短的诗")]
    )
    return {"poem": msg.content}

def aggregator(state: OverAllState) -> OverAllState:
    """将笑话、故事和诗歌合并为单一输出"""

    combined = f"下面是一个关于“{state['topic']}”的故事、笑话和诗歌！\n\n"
    combined += f"故事：\n{state['story']}\n\n"
    combined += f"笑话：\n{state['joke']}\n\n"
    combined += f"诗歌：\n{state['poem']}"
    return {"combined_output": combined}

# 构建工作流
builder = StateGraph(state_schema=OverAllState)

# 添加节点
builder.add_node("call_model_1", call_model_1)
builder.add_node("call_model_2", call_model_2)
builder.add_node("call_model_3", call_model_3)
builder.add_node("aggregator", aggregator)

# 添加边，连接各个节点
builder.add_edge(START, "call_model_1")
builder.add_edge(START, "call_model_2")
builder.add_edge(START, "call_model_3")
builder.add_edge(["call_model_1", "call_model_2", "call_model_3"], "aggregator")
builder.add_edge("aggregator", END)

graph = builder.compile()

# 调用工作流
response = graph.invoke({"topic": "猫"})
print(response)

from IPython.display import display
display(graph)
```

**运行结果如下**

```python
{
    'topic': '猫',
    'joke': '“为什么猫总是赢不了电脑游戏？”\n“因为鼠标（鼠）一出现，它就光顾着抓屏（屏幕）了……”',
    'story': '# 猫的故事\n\n老张退休那年，在楼下捡到一只瘦骨嶙峋的橘猫。\n\n他本不爱猫，但看它可怜，便每天在楼道口放一碗剩饭。猫很警惕，非要等他走远了才肯吃。就这样喂了三个月，猫终于允许他靠近一臂的距离。\n\n半年后的一天夜里，老张突发心梗，倒在客厅地板上，手机摔在茶几底下，怎么也够不着。意识模糊间，他感到一团温热的东西贴上了他的胸口——是那只橘猫。它不知怎么钻进了他家门。\n\n猫用头蹭他的手，蹭他的脸，见他没反应，便冲进屋里的每一个房间，歇斯底里地叫。最后它跳上窗台，用爪子疯狂拍打玻璃，叫声惊动了楼下乘凉的邻居。\n\n120赶到时，老张已经在地上躺了将近一个小时。医生说，再晚十分钟，就悬了。\n\n出院后，橘猫正式住进了老张家。老张给它取名“富贵”，逢人就说：“我家富贵啊，有九条命，分了我一条。”\n\n富贵永远听不懂这些话。它只是每天黄昏，准时跳上老张的膝盖，把一团温暖塞进他的怀里，呼噜呼噜地睡去。',
    'poem': '## 《猫》\n\n黄昏被毛茸茸地踩碎，\n肚皮起伏如钟摆，\n眼睫间，星辰坠落。\n当琥珀瞳孔转动，\n整片夜色都来蜷卧。',
    'combined_output': '下面是一个关于“猫”的故事、笑话和诗歌！\n\n故事：\n# 猫的故事\n\n老张退休那年，在楼下捡到一只瘦骨嶙峋的橘猫。\n\n他本不爱猫，但看它可怜，便每天在楼道口放一碗剩饭。猫很警惕，非要等他走远了才肯吃。就这样喂了三个月，猫终于允许他靠近一臂的距离。\n\n半年后的一天夜里，老张突发心梗，倒在客厅地板上，手机摔在茶几底下，怎么也够不着。意识模糊间，他感到一团温热的东西贴上了他的胸口——是那只橘猫。它不知怎么钻进了他家门。\n\n猫用头蹭他的手，蹭他的脸，见他没反应，便冲进屋里的每一个房间，歇斯底里地叫。最后它跳上窗台，用爪子疯狂拍打玻璃，叫声惊动了楼下乘凉的邻居。\n\n120赶到时，老张已经在地上躺了将近一个小时。医生说，再晚十分钟，就悬了。\n\n出院后，橘猫正式住进了老张家。老张给它取名“富贵”，逢人就说：“我家富贵啊，有九条命，分了我一条。”\n\n富贵永远听不懂这些话。它只是每天黄昏，准时跳上老张的膝盖，把一团温暖塞进他的怀里，呼噜呼噜地睡去。\n\n笑话：\n“为什么猫总是赢不了电脑游戏？”\n“因为鼠标（鼠）一出现，它就光顾着抓屏（屏幕）了……”\n\n诗歌：\n## 《猫》\n\n黄昏被毛茸茸地踩碎，\n肚皮起伏如钟摆，\n眼睫间，星辰坠落。\n当琥珀瞳孔转动，\n整片夜色都来蜷卧。'
}
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625180443516.png)

---

### 13.2.3. `Routing`：路由
#### 13.2.3.1. 核心思想
先识别输入类型，再把请求送到专门的处理流程：

```latex
                    ┌→ 退款流程
输入 → 路由判断节点 ─┼→ 售价咨询流程
                    └→ 商品推荐流程
```

它适合不同类型请求需要不同处理逻辑的场景，例如：

+ 售前、售后、退款分类
+ 普通问答、代码生成、数据分析
+ 文本、图片、音频请求分类
+ 不同领域知识库的选择

#### 13.2.3.2. 设计原则
最佳实践：

+ `LLM`** 节点**负责语义判断；
+ **路由函数**负责把结构化结果映射到节点；
+ **业务节点**负责真正处理任务。

不要让路由 LLM 直接返回任意节点名，否则模型输出和图结构会过度耦合，也难以进行类型检查和异常兜底。

#### 13.2.3.3. 示例
**示例如下**

```python
from pydantic import BaseModel, Field
from typing import Literal,TypedDict
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage, SystemMessage

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

# 定义用于结构化输出的 Schema，作为路由判断依据
class Route(BaseModel):
    step: Literal["poem", "story", "joke"] = Field(
        None,
        description="路由流程中的下一执行步骤",
    )

# 为大模型添加结构化输出能力
router = model.with_structured_output(Route)

# 图状态
class OverAllState(TypedDict):
    input: str
    decision: str
    output: str

# 节点
def model_call_1(state: OverAllState) -> OverAllState:
    """生成故事"""

    result = model.invoke(
        [HumanMessage(content=state["input"])]
    )
    return {"output": result.content}


def model_call_2(state: OverAllState) -> OverAllState:
    """生成笑话"""

    result = model.invoke(
        [HumanMessage(content=state["input"])]
    )
    return {"output": result.content}


def model_call_3(state: OverAllState) -> OverAllState:
    """生成诗歌"""

    result = model.invoke(
        [HumanMessage(content=state["input"])]
    )
    return {"output": result.content}


def model_call_router(state: OverAllState) -> OverAllState:
    """将用户输入路由到合适的节点"""

    # 调用具有结构化输出能力的大模型，完成路由判断
    decision = router.invoke(
        [
            SystemMessage(
                content=(
                    "根据用户的请求，将其路由到 story、joke 或 poem。"
                    "请求编写故事时返回 story，请求编写笑话时返回 joke，"
                    "请求编写诗歌时返回 poem。"
                )
            ),
            HumanMessage(content=state["input"]),
        ]
    )

    return {"decision": decision.step}


# 条件边函数：根据路由决策选择下一个节点
def route_decision(
        state: OverAllState
) -> Literal[
    "model_call_1",
    "model_call_2",
    "model_call_3",
    END
]:
    # 返回接下来要执行的节点名称
    if state["decision"] == "story":
        return "model_call_1"
    elif state["decision"] == "joke":
        return "model_call_2"
    elif state["decision"] == "poem":
        return "model_call_3"
    return END


# 构建工作流
builder = StateGraph(OverAllState)

# 添加节点
builder.add_node("model_call_1", model_call_1)
builder.add_node("model_call_2", model_call_2)
builder.add_node("model_call_3", model_call_3)
builder.add_node("model_call_router", model_call_router)

# 添加边，连接各个节点
builder.add_edge(START, "model_call_router")
builder.add_conditional_edges(
    "model_call_router",
    route_decision,
    {
        # route_decision 返回的名称：接下来要执行的节点名称
        "model_call_1": "model_call_1",
        "model_call_2": "model_call_2",
        "model_call_3": "model_call_3",
    },
)
builder.add_edge("model_call_1", END)
builder.add_edge("model_call_2", END)
builder.add_edge("model_call_3", END)

# 编译工作流
graph = builder.compile()

# 调用工作流
state = graph.invoke({"input": "写一个关于猫的笑话"})
print(state["output"])

# 显示工作流图
from IPython.display import display
display(graph)
```

**运行结果如下**

```python
这是一个关于猫的冷笑话，希望你喜欢：

**猫为什么不喜欢洗澡？**

**因为它不想变成“落汤包”（肉包）。**

（笑点解析：谐音梗，把“落汤猫”说成“落汤包”，而猫胖胖的确实有点像包子。）
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625180521496.png)

---

### 13.2.4. `Orchestrator-worker`：编排器—工作节点
#### 13.2.4.1. 核心思想
编排器先分析任务，动态生成若干子任务，再创建对应数量的 Worker：

```latex
                         ┌→ Worker 1 ─┐
输入 → Orchestrator ─────┼→ Worker 2 ─┼→ Synthesizer → 输出
                         ├→ Worker 3 ─┤
                         └→ Worker N ─┘
```

它通常包含三个组件：

| 组件 | 职责 |
| :--- | :--- |
| `Orchestrator`（编排器） | 分析任务、制定计划、生成子任务列表 |
| `Workers`（工作节点） | 分别处理各子任务，通常并行执行 |
| `Synthesizer`（聚合节点） | 汇总所有 Worker 结果，生成最终答案 |


适合任务数量无法预先确定的场景，例如：

+ 根据主题动态生成报告章节
+ 修改未知数量的代码文件
+ 为多个数据源分别执行分析
+ 将长文档动态拆分为若干部分处理

#### 13.2.4.2. 和`Parallelization`的区别
| 模式 | 任务数量 |
| :--- | :--- |
| `Parallelization` | 编译图时已确定（静态） |
| `Orchestrator-worker` | 运行时动态计算 |


#### 13.2.4.3. 本质
`Orchestrator-worker` 本质上就是 `Map-Reduce`，我们在前面的课程中已经实现过了

```latex
Orchestrator：生成 Map 任务
Send：动态分发
Worker：执行 Map
Reducer：合并中间结果
Synthesizer：执行 Reduce
```

#### 13.2.4.4. 示例
**示例如下**

```python
from typing import Annotated, List, TypedDict
from collections.abc import Sequence

from langchain_core.messages import SystemMessage, HumanMessage
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from langchain_deepseek import ChatDeepSeek

from operator import add
from dotenv import load_dotenv
from pydantic import BaseModel, Field

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

# 结构化输出的 Schema
class Section(BaseModel):
    name: str = Field(
        description="报告的章节名称",
    )
    description: str = Field(
        description="本节主题和核心思想的概述",
    )

class Sections(BaseModel):
    sections: List[Section] = Field(
        description="报告的章节列表",
    )

# 编排器
planner = model.with_structured_output(Sections)

# 图状态
class OverAllState(TypedDict):
    topic: str  # 报告主题
    sections: list[Section]  # 报告章节列表
    completed_sections: Annotated[
        list, add
    ]  # 所有工作节点并行写入该字段
    final_report: str  # 最终报告


# 工作节点状态
class WorkerState(TypedDict):
    section: Section
    completed_sections: Annotated[list, add]

# 节点
def orchestrator(state: OverAllState) -> OverAllState:
    """编排器：生成报告编写计划"""

    # 生成报告章节规划
    report_sections = planner.invoke(
        [
            SystemMessage(content="为这份报告生成一个章节规划。"),
            HumanMessage(content=f"报告主题如下：{state['topic']}"),
        ]
    )

    return {"sections": report_sections.sections}

def model_call(state: WorkerState) -> WorkerState:
    """工作节点：编写报告中的一个章节"""

    # 生成章节内容
    section = model.invoke(
        [
            SystemMessage(
                content=(
                    "根据提供的章节名称和章节描述编写报告内容。"
                    "不要在每个章节前添加额外的开场说明。"
                    "使用 Markdown 格式。"
                )
            ),
            HumanMessage(
                content=(
                    f"章节名称：{state['section'].name}\n"
                    f"章节描述：{state['section'].description}"
                )
            ),
        ]
    )

    # 将生成的章节写入已完成章节列表
    return {"completed_sections": [section.content]}


def synthesizer(state: OverAllState) -> OverAllState:
    """将所有章节合成为完整报告"""

    # 获取所有已完成的章节
    completed_sections = state["completed_sections"]

    # 将已完成章节拼接为最终报告
    completed_report_sections = "\n\n---\n\n".join(completed_sections)

    return {"final_report": completed_report_sections}


# 条件边函数：为规划中的每个章节创建一个 model_call 工作节点
def assign_workers(state: OverAllState) -> Sequence[Send]:
    """为规划中的每个章节分配一个工作节点"""

    # 使用 Send API 并行启动各个章节的编写任务
    return [Send("model_call", {"section": s}) for s in state["sections"]]


# 构建工作流
builder = StateGraph(state_schema=OverAllState)

# 添加节点
builder.add_node("orchestrator", orchestrator)
builder.add_node("model_call", model_call)
builder.add_node("synthesizer", synthesizer)

# 添加边，连接各个节点
builder.add_edge(START, "orchestrator")
builder.add_conditional_edges(
    "orchestrator",
    assign_workers,
    ["model_call"],
)
builder.add_edge("model_call", "synthesizer")
builder.add_edge("synthesizer", END)

# 编译工作流
graph = builder.compile()

# 调用工作流
state = graph.invoke(
    {"topic": "撰写一份关于大语言模型缩放定律的报告"}
)

from IPython.display import Markdown, display

# 显示工作流图
display(graph)
# 渲染报告
Markdown(state["final_report"])
```

**运行结果如下**

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625180543541.png)

```markdown
引言
缩放定律是深度学习与人工智能领域中的核心理论基础之一。它揭示了模型性能与关键资源（如参数量、数据规模、计算算力）之间普遍存在的幂律关系，即当模型规模、训练数据量或计算预算成比例扩大时，模型的表现能力会呈现出可预测性的提升。这一发现最早源自对语言模型的系统性实证研究，随后在计算机视觉、多模态学习等众多任务中被广泛验证，深刻改变了现代AI研究的范式。

缩放定律的重要性体现在多个层面。从学术研究角度看，它不仅为“更大的模型、更多的数据”提供了科学依据，还催生了以GPT、PaLM、LLaMA为代表的大规模预训练模型浪潮。从工程实践角度看，缩放定律为资源分配提供了量化指导——研究人员可以据此预测模型性能对计算预算的依赖关系，从而制定更高效的训练策略。此外，在探索模型能力涌现性的前沿领域，缩放定律更是理解大语言模型行为边界的关键工具。因此，深入剖析缩放定律的理论内涵、实验验证与局限性，对于把握AI发展的底层逻辑具有不可替代的价值。

本报告将系统梳理缩放定律的研究脉络。后续章节首先阐述语言模型缩放的基本原理与核心公式；随后通过经典实验案例展示缩放定律的实证表现，并讨论其在不同任务与架构中的泛化能力；在此基础上，分析缩放定律的物理极限与开放挑战，包括数据效率、规模收益递减等问题；最后总结缩放定律对AI未来发展的战略启示。全文力求在学术严谨性与可读性之间取得平衡，为读者提供一幅关于缩放定律的全局图景。

缩放定律的基本概念
缩放定律（Scaling Laws）是深度学习中描述模型性能如何随模型规模（参数数量）、数据集规模（训练数据量）以及计算量（训练使用的计算资源，通常以FLOPs表示）变化的经验性规律。这些规律揭示了在大规模训练中，模型性能（例如损失值或下游任务准确率）与上述三个因素之间存在较为平滑且可预测的幂律关系。

具体而言，缩放定律通常由以下关系组成：

模型规模（Model Size）：在给定的数据集和计算预算下，增加模型参数数量（例如从几亿到数千亿）能够显著提升性能，但收益呈现递减趋势。性能提升与模型参数量的幂次方成正比，即 ( L \propto N^{-\alpha} )，其中 ( L ) 为损失， ( N ) 为参数数量， ( \alpha ) 为缩放指数。

数据集规模（Dataset Size）：在固定模型结构和计算资源的情况下，增加训练数据量同样可以降低损失。类似地，损失与数据量的关系可表示为 ( L \propto D^{-\beta} )，其中 ( D ) 为数据集规模， ( \beta ) 为对应的缩放指数。这意味着数据量越大，模型过拟合的风险越小，性能提升越稳定。

计算量（Compute）：将模型规模和数据规模同时考虑，总训练计算量（以FLOPs衡量）与性能的关系可概括为 ( L \propto C^{-\gamma} )，其中 ( C ) 为计算量。研究表明，在最优资源配置下，增加计算量能可靠地提升性能，且存在一个近似固定的计算最优前沿。

这些关系表明，模型性能并非由单一因素决定，而是三者共同作用的结果。缩放定律的核心在于，当预算（计算成本）固定时，存在一组最优的模型规模与数据规模分配，使得性能最大化。例如，若计算预算充足，则应同时扩大模型参数和训练数据；若某一资源受限，则需根据缩放指数的相对大小调整比例。

缩放定律的一个关键结论是：大模型是样本高效的——即随着模型规模增大，同样的数据量能带来更大的性能提升；反过来，大模型也是计算高效的——即同样的计算预算，训练更大的模型（配合适量数据）往往比训练小模型更优。这些规律为现代大规模语言模型的设计，如GPT系列、LLaMA等，提供了理论基础：它们正是遵循“同时扩大模型、数据和计算量”的指导原则。

核心发现：损失函数的幂律关系
Kaplan等人的研究揭示了大语言模型的损失函数与模型参数规模（(N)）、训练数据规模（(D)）以及计算量（(C)，通常以FLOPs衡量）之间存在明确的幂律（Power-Law）关系。这一发现为大规模模型的扩展提供了可预测的量化基础。

1. 损失与参数规模的幂律关系
在固定数据规模（(D)）的情况下，模型测试损失（(L)）与参数规模（(N)）呈现出双向的幂律行为：

欠参数化区域：当 (N) 较小时，模型容量不足，损失较高。
过参数化区域：当 (N) 持续增大，损失开始随参数增长以平滑的幂律形式下降，最终收敛于数据规模所限制的最优损失（即“不可约损失”）。
该关系可近似描述为： [ L(N) \approx L_{\infty} + \alpha_N \cdot N^{-\gamma_N} ] 其中 (L_{\infty}) 表示数据质量决定的不可约损失，(\gamma_N) 为幂律指数（通常在0.07-0.10之间），(\alpha_N) 为缩放常数。

2. 损失与数据规模的幂律关系
相似地，在固定参数规模（(N)）时，损失函数与训练数据规模（(D)）也呈现幂律趋势。当数据量增加，模型通过摄取更多示例改进泛化能力，损失按如下形式降低： [ L(D) \approx L_{\infty} + \alpha_D \cdot D^{-\gamma_D} ] 实验表明，(\gamma_D) 的典型值在0.07-0.10范围内，与 (\gamma_N) 接近，暗示参数和数据对损失的贡献在结构上是对称的。

3. 损失与计算量的幂律关系
计算量（(C)）被视为参数规模与数据规模的复合函数：(C \approx 6N \cdot D)（基于Transformer的前向与后向传播成本）。通过等比例扩展参数和数据，损失与计算量之间也遵循幂律规律： [ L(C) \approx L_{\infty} + \alpha_C \cdot C^{-\gamma_C} ] 其中 (\gamma_C) 通常在0.05-0.08左右，略低于前两个指数，反映了优化效率随规模增大而递减的边际效应。

4. 幂律关系的核心含义
可预测的扩展：只要维持参数和数据规模的平衡增长，损失减少的速率可被幂律精确预测，从而指导资源分配。
收益递减法则：幂律指数越小（绝对值），意味着每增加一单位计算量所获得的损失改善越慢，最终接近不可约损失。
Compute-Optimal极限：结合各项幂律曲线，存在一个最优的参数-数据配比，使得在给定总计算预算下损失最小化，这也是后续Chinchilla法则的理论基础。
综上所述，损失函数的幂律关系构成了大模型性能超越的关键先验知识，它强调了在扩展过程中参数、数据和计算量必须协同增长，否则因资源浪费（如只增加参数而不增加数据）导致幂律提升停滞。

数据与计算效率
在深度学习中，模型性能的提升往往依赖于增加参数数量与数据规模，但现实中的计算预算（如GPU时长、电力或成本）通常是有限的。如何在给定的计算预算下，最优地分配参数规模与数据规模，以实现最佳的模型性能，是数据与计算效率研究的核心问题。这涉及到缩放定律的深入理解以及训练策略的优化。

1. 缩放定律回顾与计算预算视角
根据经典的神经语言模型缩放定律（如Kaplan等人于2020年提出的研究，以及DeepMind的Chinchilla研究），模型性能（通常用验证损失衡量）受三个主要因素影响：模型参数数量（N）、训练数据Token数量（D）和计算预算（C）。缩放定律通常可以近似为幂律关系：损失 L(N,D) ≈ A/N^α + B/D^β + E，其中α和β是缩放指数，E是数据不可压缩的熵。计算预算与N和D并非独立，在大规模训练中，计算量主要来自前向和反向传播，通常近似为 C ≈ 6 * N * D（对于Transformer结构，忽略通信和嵌入层开销）。

给定固定的计算预算C，如果分配过少参数而使用过多数据（N过小，D过大），参数容量不足以捕捉数据中的模式，导致欠拟合；反之，如果参数过多而数据不足（N过大，D过小），模型会过拟合，无法通过足够的样本来泛化。最优分配的目标是找到N和D，使得在C固定的条件下，验证损失L最小化。

2. 计算最优训练：参数与数据的平衡点
Chinchilla缩放定律的研究指出，对于许多计算预算，最优的参数与数据规模满足近似关系：N_opt ∝ C^a 且 D_opt ∝ C^b，其中 a 和 b 之和接近1，且 a 通常小于 b。具体地，Chinchilla研究发现，对于使用Adam优化器的Transformer语言模型，当预算增加时，最优的参数和数据分配比例大致保持恒定，即 对于每个参数，约应分配20个训练Token（这一比例因模型架构和任务有所不同，但提供了一个直观的指导）。

例如，在计算预算为1e20 FLOPs的情况下，最优配置可能是约70B参数配合1.4T训练Token；而预算增加到2e20 FLOPs时，最优可能变为100B参数配合2T Token。这一发现反驳了早期“仅扩大参数而忽视数据”的做法，强调在固定预算下，同时按比例放大数据和参数比单独放大其中一个更有效。

3. 计算资源有限时的策略
当计算预算远小于现代主流规模（如单个GPU或小型集群）时，最优分配的策略需要更细致的考量：

预算极小的情况（如单GPU、几天训练）：参数规模和数据规模可能都受到硬件限制。此时，优先考虑模型容量，但需避免参数量超出能有效训练的最小数据量。通常，使用预训练模型微调比从头训练更高效，因为预训练模型已经在大规模数据上获得了通用表征，少量下游数据即可适配，从而将计算预算集中用于微调步骤。

中等预算（如多GPU、数周训练）：可参考缩放定律的比例关系。假设预算为C，根据N_opt ∝ C^0.5和D_opt ∝ C^0.5（此比例来自某些缩放研究，具体指数需要实验校准），推算合适的参数和Token数。实践中，可从公开的缩放曲线（如LLaMA、Chinchilla等）中提取经验公式，例如对于Transformer-LM，每增加一倍计算预算，同时将参数和Token增加约1.4倍（即C^0.5增长）。

预算极度受限且追求低成本：在边缘设备或快速原型开发中，计算效率意味着最小的参数规模实现足够好的性能。此时，推荐使用知识蒸馏、模型压缩或稀疏训练等方法。但这些方法本身需消耗计算资源，需权衡是否优于直接缩放

4. 数据效率的多重维度
除了参数与数据总量的分配，数据本身的特性也影响计算效率：

数据质量 vs. 数量：在有限计算预算下，使用高质量、代表性强的数据（例如经过筛选、去重、平衡类别的数据）往往比盲目增加数据量更有效。数据质量提升可降低所需的数据量，从而减少训练所需的计算量。

数据复用与课程学习：对数据进行多次训练（epoch）会引入过拟合风险，但在数据量不足时，适当的重复训练（如使用数据增强）可以增加有效样本数。课程学习（先易后难的数据顺序）可加速收敛，减少达到特定损失所需的计算步数。

主动学习与数据选择：在训练期间动态选择最具有信息性的数据子集进行训练，避免在容易或冗余样本上浪费计算资源。例如，基于损失值或不确定性采样，仅使用部分数据即可达到接近全量数据的性能。

5. 实际分配建议与工具使用
在工程实践中，最优分配可通过以下步骤实现：

建立小规模缩放曲线：使用计算预算的1%或更少进行小规模实验，训练不同（N,D）组合的模型，拟合幂律关系，推断出当前预算下的最优比例。
利用搜索或公式：参考已发表的缩放定律常数（如Chinchilla的20 Token/参数法则），但需注意这基于特定架构和数据集，应在目标数据上进行验证。
考虑数据生成成本：若数据获取比计算更昂贵（如需要人工标注），则应优先增大参数并使用预训练模型微调；反之，若计算昂贵而数据丰富（如互联网文本），则倾向于更多数据。
分阶段训练：若预算允许，可先训练一个大模型，再用其蒸馏小模型，是一种“先扩大参数，再缩小”的策略，但需要额外计算用于蒸馏。
总之，在有限计算预算下，最优地分配参数与数据规模的核心在于打破“越大越好”的直觉，理解缩放定律的隐含平衡，并通过质量控制和数据效率提升来最大化计算投入的回报。通过系统性实验和理论指导，可在固定成本下获得显著的性能提升。

超越损失：缩放定律对下游任务的影响
缩放定律最初源于对预训练损失随模型规模、数据量和计算量变化的观察，但其意义远不止于优化训练曲线。随着模型规模的增长，许多下游任务展现出超越简单损失改善的质性变化，呈现出涌现能力、上下文学习能力增强等非连续现象。本章将深入分析缩放定律如何预测并影响这些下游任务的表现，揭示规模提升带来的能力跃迁及其内在机制。

涌现能力：缩放定律的阶跃性预测
缩放定律通常描绘损失随规模平滑下降，但在下游任务上，模型能力并非线性提升，而是在达到特定规模后突然显现。例如，在算术推理、多步逻辑链、代码生成等任务中，小模型几乎无法完成，而大模型在参数跨过某个阈值时，准确率从接近零跃升至显著水平。这种涌现现象与实际任务性能的S形增长曲线相对应，而缩放定律通过预测模型不同区域的表示质量，揭示了涌现的触发条件：当模型参数量、数据量和训练步数满足特定组合时，隐层表征的容量与任务复杂度首次匹配。具体而言，缩放定律提供的幂律关系可以粗略估计模型在给定计算预算下的最优规模，当最优规模超过任务所需的最小表示容量时，涌现现象开始出现。因此，缩放定律并非保证所有任务同时涌现，而是定义了不同任务的能力门槛，预期在更大规模下能解锁更多高阶能力。

上下文学习能力的缩放效应
上下文学习——即模型无需梯度更新，仅通过在提示中提供少量示例即可执行新任务——是大规模语言模型最突出的特性之一。研究发现，上下文学习的性能与模型规模存在强正相关：小模型几乎不具备有效的上下文学习能力，随着参数规模扩大，模型能从示例中提取的模式更加准确，泛化偏差降低。缩放定律在此表现为：模型在上下文学习任务上的准确率通常符合对数或幂律增长，与预训练损失下降趋势并行。其背后的原因在于，随着规模增大，模型的前馈层和注意力头数增加，使得它能在前向传播中并行模拟更多的“推理步骤”，实现了内隐的梯度下降模拟。缩放定律预测，在固定计算预算下，规模更大的模型在上下文学习上的收益将超过同等效率的微调方法，从而引导研究者在提示工程和大规模预训练之间做出资源分配决策。

任务泛化与分布外表现
缩放定律对下游任务的影响不仅限于在分布内示例上的表现，更关键的是对分布外泛化能力的提升。实验表明，随着模型规模增长，模型在不同任务类别、不同领域、甚至不同语言之间的迁移性能稳定提升。这源于缩放定律中“数据多样性”的维度——更大的模型能更充分地利用多源数据中的结构共性，从而在未明确训练的领域形成鲁棒的假设。例如，在数学推理任务上，缩放定律使得模型能将从自然语言中学到的逻辑模式迁移至符号推理，表现出更强的类比和抽象能力。然而，这种迁移并非均匀：某些依赖低频知识或特殊模式的任务，可能仍需要接近边界规模才能突破泛化瓶颈。缩放定律因此为任务选择提供了依据——优先将计算资源分配给那些对规模敏感、且具有明确泛化回报的任务。

任务特定缩放曲线与优化经济性
不同下游任务对模型规模、数据量和计算量的响应曲线各异。有些任务（如简单分类或QA）在中小规模模型上即饱和，进一步扩展规模带来的收益递减很快；而另一些任务（如复杂推理、多语言理解）则持续受益于规模扩大。通过构建任务特定的缩放曲线（通常为幂律或指数饱和形式），研究者可以预测在给定计算预算下，投入资源以提升模型规模、增加训练数据或优化数据质量三者中的最优选择。例如，对于认知密集型任务，在模型规模上的投入回报率往往高于数据增量；而对于知识密集型任务，数据多样性可能更重要。这种缩放定律的延伸应用，使得模型部署决策更加数据驱动，避免了盲目扩大规模导致的经济浪费。

涌现能力背后的机制：表示压缩与结构对齐
进一步分析，缩放定律对涌现能力的积极影响可以理解为：大规模模型通过高维表示实现了对任务本质结构的压缩与对齐。在小规模模型中，隐层表示空间难以同时编码多个复杂模式，导致任务相关信号被噪声淹没。而当模型规模超过临界点，表示空间的各个维度开始分离出可解释的语义轴（如因果关系、时态关系、空间关系等）。缩放定律预测这些表示维度的“分化临界点”，在该点之后，模型能够以更少的冲突编码多个任务的隐含规则。此外，注意力模式的稳定化——即多头注意力在大型模型中形成更聚焦的依赖关系——进一步支撑了长程逻辑和上下文结构的捕捉，从而直接提升了下游任务的逻辑一致性和准确性。

缩放极限与下游任务瓶颈
尽管缩放定律整体上乐观地预测了规模带来的收益，但对下游任务的影响并非无限。随着模型参数进一步增大，部分任务（特别是需要外部知识更新或实时推理的任务）会遇到内存带宽瓶颈、指令遵循退化或幻觉加剧等问题。缩放定律在此提示，当模型规模超过数据质量所能支撑的有效容量时，损失的下行趋势放缓，而下游任务的性能可能进入“平台期”甚至出现波动。例如，极端大的模型可能对上下文中的干扰信息更敏感，导致在简单任务上的表现反常。因此，未来在追求更优下游任务表现时，不仅需要继续扩展规模，还必须结合数据治理、对齐训练和检索增强等手段，以跨越缩放定律在应用层面的新瓶颈。

3.3 Chinchilla缩放定律与最优计算分配
在深度学习的规模探索中，DeepMind提出的Chinchilla研究引入了关于模型与数据规模分配的新视角。以往的研究，如Kaplan等人提出的缩放定律，倾向于认为在增加模型参数时，数据规模可以相对缓慢地增长，即模型越大，性能随计算量提升的边际收益越显著。然而，Chinchilla研究对此提出了挑战。

核心观点：对于给定的计算预算（即总计算量，通常以FLOPs衡量），模型参数数量与训练数据规模（Token数量）应当等比例缩放。换言之，当计算预算增加时，为了达到最优性能，必须同时增加模型的参数和训练数据量，且两者间存在一个最优的比例关系。

研究方法。Chinchilla团队通过系统地变化模型大小（参数数量）和训练数据量（Token数量），在固定计算预算下评估模型性能。他们发现，在以往的研究中，许多大型模型（如GPT-3）实际上是在“计算欠调度”状态下训练的——即模型过大而数据不足。这意味着，在同样的计算预算下，如果一个模型较小但训练在更多的数据上，它可能会取得更好的性能。

关键结论。Chinchilla揭示了最优计算分配的法则：

最优模型参数与训练Token数应遵循大致相等的缩放率。
具体而言，对于主流Transformer模型，最优的模型参数数量（N）与训练Token数量（D）之间满足：N ∝ D。这意味着，当计算预算翻倍时，应同时将模型大小和数据量分别增加约1.4倍（即N ∝ C^0.5, D ∝ C^0.5，其中C为计算量）。
实际影响。基于这一发现，DeepMind训练了Chinchilla模型：一个拥有70亿参数的模型，但使用1.4万亿个Token进行训练。与当时更大规模的模型（如Gopher，2800亿参数，仅3000亿Token训练）相比，在相同的计算预算下，Chinchilla在多种自然语言处理任务中表现更优或相当。这证明了“小而精”的缩放策略的潜力：与其盲目增大模型，不如在模型和数据间取得更平衡的分配。

理论意义。Chinchilla缩放定律重新定义了“缩放”的内涵——它不再是单纯的参数扩展，而是计算资源在模型复杂度（参数空间）和数据复杂度（经验分布覆盖度）之间的优化分配。这一观点为后续的模型训练（如LLaMA系列）提供了重要指导，强调了数据质量与规模的重要性不亚于模型架构的复杂性。

缩放定律的局限性与挑战
缩放定律的边界

缩放定律（Scaling Laws）揭示了模型性能与模型参数、数据量、计算量之间的幂律关系，但这一规律并非无限制成立。随着模型规模的持续增长，性能提升呈现边际递减趋势，最终趋近于一个理论或实际的上限。该边界的出现源于多重因素：首先，现有训练数据的质量与多样性存在天花板，即使增加数据量，新数据提供的信息增益也会下降；其次，模型容量受限于硬件架构（如内存带宽、计算单元效率），极端规模的模型可能触发通信瓶颈或能耗约束；再者，缩放定律本身假设了理想化的训练条件，未考虑实际训练中的噪声、分布偏移等扰动。因此，简单地堆叠参数和计算资源已难以突破性能瓶颈，研究者需重新审视缩放定律的适用范围。

数据稀缺的困境

高质量训练数据的稀缺性是当前缩放定律面临的核心挑战之一。自然语言、图像等领域中，人工标注或精选数据的增长速率远落后于模型参数规模的扩展。无监督数据虽可大规模获取，但其噪声、冗余和不平衡分布可能导致模型学习到虚假相关性，而非真正有意义的模式。例如，在罕见语言任务或专业领域（如医学、法律）中，数据稀疏性直接限制了模型泛化能力。此外，基于互联网爬取的粗筛数据包含偏见、脏标签和版权问题，进一步加剧了数据质量与规模之间的张力。解决这一困境需要探索数据增强、半监督学习或合成数据生成等替代方案，但现有方法仍难以完全弥补真实数据的缺失。

训练不稳定性问题

大规模模型训练固有的不稳定性是缩放定律实际应用的又一障碍。当模型参数增至数百亿甚至万亿级别时，优化过程中的梯度爆炸、损失振荡、模式坍塌等问题显著增加。这主要由以下原因引起：深度网络中的梯度传播路径过长导致信号衰减；批量规模（batch size）与学习率之间的协变关系难以维护；分布式训练中的异步通信引入延迟和参数不一致。训练不稳定性不仅延长了收敛时间，还可能使模型陷入局部最优或产生灾难性遗忘。现有缓解手段（如梯度裁剪、混合精度训练、优化器调参）虽能部分缓解问题，但缺乏理论保证，尤其是当模型规模跨越特定临界点后，传统训练策略可能完全失效。

超越简单缩放的模型架构创新

面对缩放定律的局限性，研究者正转向非简单缩放的架构创新，以绕过参数扩张的性能瓶颈。例如，混合专家模型（MoE）通过稀疏激活机制，在增加总参数量的同时保持计算成本接近线性增长；维度扩展策略（如线性注意力、长序列建模）尝试替代传统transformer中的二次复杂度注意力机制；此外，结构化权重矩阵（如低秩分解、张量分解）能在不显著增加模型尺寸的前提下提升表达能力。这些创新揭示了更高效的缩放方向：而非盲目提升参数量，优化模型内部的信息流动效率、参数复用率以及计算资源分布。未来，结合神经架构搜索（NAS）或自演化设计的方法，可能进一步打破缩放定律的线性束缚，开辟新的性能增长路径。

结论与展望
本文围绕缩放定律（Scaling Laws）展开了系统性分析，揭示了大规模模型性能与计算资源、数据规模及参数数量之间存在的幂律关系。核心结论包括：模型性能在相当广泛的范围内随计算预算、数据集大小和参数数量的增加而稳定提升，这一规律为现代深度学习模型的规模化设计提供了理论依据与工程指导。然而，缩放并非无代价，边际收益递减、训练成本激增以及数据质量瓶颈成为进一步扩展的主要制约因素。

在未来研究方向上，稀疏模型（Sparse Models）展现出突破现有缩放瓶颈的潜力。通过引入混合专家系统（MoE）、参数高效微调及动态结构剪枝等机制，稀疏模型可在不显著增加计算开销的前提下，实现参数规模的有效放大。该方向有望打破密集模型中的线性缩放约束，推动模型容量与效率的协同增长。

多模态缩放（Multimodal Scaling）是另一个值得关注的前沿领域。当前缩放定律多聚焦于单一模态（如文本或图像），而跨模态联合训练下性能与资源间的映射关系尚不明确。未来亟需建立针对多模态输入的统一缩放框架，探索视觉、语言、音频等异构数据在共同训练过程中的交互效应与计算效率优化策略。

此外，训练数据质量与来源的多样性、模型泛化边界、以及缩放定律在非自监督学习范式（如强化学习、在线学习）中的适用性，构成后续研究需要深入探讨的关键议题。随着实际部署对能耗、推理速度和环境可持续性提出更高要求，缩放定律的应用亦需与资源约束感知模型设计及低碳训练方法相结合。总之，未来研究需在理解缩放基本规律的基础上，拓展其边界，提升其可控性，并为其在更广泛人工智能系统中的部署提供理论支持与实践路径。
```

---

### 13.2.5. `Evaluator-optimizer`：评估器—优化器
#### 13.2.5.1. 核心思想
一个节点生成结果，另一个节点评估结果；不合格就携带反馈重新生成：

```latex
          ┌──────────────────────┐
          ↓                      │
输入 → Generator → Evaluator ────┤
                      ├─ 不合格 ─┘
                      └─ 合格 → END
```

适合：

+ 翻译质量迭代
+ 代码生成与代码审查
+ 文案生成与合规检查
+ `SQL` 生成与语法验证
+ 报告生成与事实检查
+ 人工审批后修改

这种模式适用于存在明确质量标准，但通常需要多轮修改才能满足标准的任务。评估者既可以是 `LLM`，也可以是规则、测试程序或人类。

#### 13.2.5.2. 必要的控制
为了避免无限循环，必须引入递归限制，可以是

+ 优雅退出的主动方法
+ 异常中断的被动方法

我们在介绍循环结构时已经系统讲解过了。

#### 13.2.5.3. 示例
**示例如下**

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek

from dotenv import load_dotenv
from pydantic import BaseModel, Field

load_dotenv(override=True)

model = ChatDeepSeek(
    model="deepseek-v4-flash",
    extra_body={
        "thinking": {
            "type": "disabled"
        }
    }
)

# 图状态
class OverAllState(TypedDict):
    joke: str
    topic: str
    feedback: str
    funny_or_not: str

# 定义用于结构化输出的 Schema，作为笑话评估依据
class Feedback(BaseModel):
    grade: Literal["好笑", "不好笑"] = Field(
        description="判断这个笑话是否好笑。",
    )
    feedback: str = Field(
        description="如果笑话不好笑，请给出具体的改进建议。",
    )

# 为大模型添加结构化输出能力，定义评估器
evaluator = model.with_structured_output(Feedback)

# 节点
def model_call_generator(state: OverAllState) -> OverAllState:
    """大模型生成笑话"""

    if state.get("feedback"):
        msg = model.invoke(
            f"""
            写一个关于“{state['topic']}”的笑话。

            请参考下面的改进建议：
            {state['feedback']}
            """
        )
    else:
        msg = model.invoke(f"写一个关于“{state['topic']}”的笑话")

    return {"joke": msg.content}


def model_call_evaluator(state: OverAllState) -> OverAllState:
    """大模型评估笑话"""

    grade = evaluator.invoke(
        f"""
        请评估下面这个笑话是否好笑：

        {state['joke']}
        """
    )

    return {
        "funny_or_not": grade.grade,
        "feedback": grade.feedback,
    }


# 条件边函数：根据评估结果决定结束流程，或者返回笑话生成节点
def route_joke(state: OverAllState) -> Literal["accept", "reject_and_feedback", END]:
    """根据评估结果决定接受笑话或根据反馈重新生成"""

    if state["funny_or_not"] == "好笑":
        return "accept"
    elif state["funny_or_not"] == "不好笑":
        return "reject_and_feedback"
    return END

# 构建工作流
builder = StateGraph(state_schema=OverAllState)

# 添加节点
builder.add_node("model_call_generator", model_call_generator)
builder.add_node("model_call_evaluator", model_call_evaluator)

# 添加边，连接各个节点
builder.add_edge(START, "model_call_generator")
builder.add_edge("model_call_generator", "model_call_evaluator")
builder.add_conditional_edges(
    "model_call_evaluator",
    route_joke,
    {
        # route_joke 返回的名称：接下来要执行的节点
        "accept": END,
        "reject_and_feedback": "model_call_generator",
    },
)

# 编译工作流
graph = builder.compile()

# 调用工作流
state = graph.invoke({"topic": "猫"})
print(state["joke"])

# 显示工作流图
from IPython.display import display
display(graph)
```

**运行结果如下**

```python
# 关于猫的笑话

一只猫去应聘当保安。

面试官问：“你有什么特长？”

猫说：“我精通‘猫步’，走路悄无声息，可以抓到任何小偷。”

面试官点头：“不错。那面对危险情况，你会怎么处理？”

猫淡定地说：“我会用‘九条命’精神，即使失败一次，还有八次机会总结经验。”

面试官很满意：“那我们决定录用你。不过，我们有个夜班岗位，你介意夜间工作吗？”

猫“喵”了一声：“正合我意，白天我要晒着太阳睡觉的。”

然后面试官又问：“最后一个问题——你如果发现仓库里有老鼠，会怎么处置？”

猫瞪大了眼睛：“老鼠？那是我请来的线人！没有老鼠，我怎么假装抓到战绩，让你们给我加罐头？”

面试官：“……你被开除了。”

猫耸耸肩：“无所谓，反正我还有八条命去找下一份工作。”
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/image-20260625180812793.png)

---

### 13.2.6. `Agent`：智能体循环
#### 13.2.6.1. 核心思想
前面的运行图设计模式都属于 `Workflow`，`Agent` 与 `Workflow` 最大的不同是：

> 开发者不再预先确定每一步具体执行什么，而是让 `LLM` 根据当前消息和工具结果决定下一步行为。
>

基本拓扑是：

```latex
          ┌─────────────────┐
          ↓                 │
输入 → LLM 决策 → 是否调用工具
                ├─ 是 → Tool ┘
                └─ 否 → END
```

即最基础的 `ReAct` 架构，我们已经不止一次介绍并实现过 `ReAct` 架构的 `Agent`

`Agent` 适合问题求解步骤和工具调用顺序无法预先确定的场景。开发者仍然负责提供工具集合、系统提示词、权限和边界，但具体执行过程由模型动态决定。

#### 13.2.6.2. 示例
**示例如下**

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt.tool_node import ToolNode

from langchain.messages import HumanMessage
from langchain_deepseek import ChatDeepSeek
from langchain.tools import tool

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
def get_weather(city: str) -> str:
    """
    根据城市查询当日天气

    Args:
        city: 城市名称
    """
    return f"{city} 今天天气不错"

tools = [get_weather]
model_with_tools = model.bind_tools(tools=tools)

def model_node(state: MessagesState) -> MessagesState:
    messages = state["messages"]
    response = model_with_tools.invoke(input=messages)

    return {
        "messages": [response]
    }

def router(state: MessagesState) -> Literal["tool_node", END]:
    last_msg = state["messages"][-1]

    if last_msg.tool_calls:
        return "tool_node"
    return END

builder = StateGraph(state_schema=MessagesState)
builder.add_node("model_node", model_node)
builder.add_node("tool_node", ToolNode(tools=tools))

builder.add_edge(START, "model_node")
builder.add_conditional_edges("model_node", router, path_map=["tool_node", END])
builder.add_edge("tool_node", "model_node")

graph = builder.compile()

response = graph.invoke({"messages": [HumanMessage(content="今天北京天气怎么样？")]})
for msg in response["messages"]:
    msg.pretty_print()

from IPython.display import display
display(graph)
```

**运行结果如下**

```python
================================ Human Message =================================

今天北京天气怎么样？
================================== Ai Message ==================================

好的，我来帮你查询一下今天北京的天气情况。
Tool Calls:
  get_weather (call_00_3kFtzQHjGl7VJkfBi5280445)
 Call ID: call_00_3kFtzQHjGl7VJkfBi5280445
  Args:
    city: 北京
================================= Tool Message =================================
Name: get_weather

北京 今天天气不错
================================== Ai Message ==================================

今天北京的**天气不错**哦！☀️

看起来是个适合出门活动的好天气，不用太担心下雨或恶劣天气的问题。如果你有具体的户外活动计划，应该会比较顺利。

请问还有其他需要帮忙的吗？😊
```








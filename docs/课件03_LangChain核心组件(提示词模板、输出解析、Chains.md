# Model I/O - 提示词模板与输出解析

> 尚硅谷大模型技术之LangChain V1.x

---

## 一、提示词模板（Prompts）

### 1.1 什么是提示词模板

在大模型应用开发中，我们通常需要根据不同的用户输入来构造不同的Prompt。如果每次都手动拼接字符串，不仅代码冗余，还容易出错。提示词模板（Prompt Template）的核心思想是**将Prompt中的固定部分与可变部分分离**，通过定义模板 + 传入变量的方式，动态生成最终的Prompt。

**核心价值**：

| 优势 | 说明 |
|------|------|
| **复用性** | 一次定义模板，多次使用不同参数 |
| **参数化** | 支持变量插值和动态内容生成 |
| **标准化** | 统一管理所有提示词，便于维护和版本控制 |
| **优化友好** | 独立的模板便于A/B测试和提示词工程 |

### 1.2 提示词模板类型概览

LangChain围绕不同的使用场景，提供了多种提示模板类型。在学习具体类型之前，先从全局视角了解它们的定位：

<img src="images/1.提示词模板类型概览.png" style="zoom:67%;" />

下面我们按照**从简单到复杂**的顺序，逐一讲解每种模板。

### 1.3 提示词模板工作原理

在深入各类模板之前，先理解它们的统一工作原理。所有提示词模板都遵循相同的处理流程：

```
变量值 → PromptTemplate → PromptValue → 聊天模型 / LLM
```

**工作流程**：

1. **输入**：传入变量值（通常以字典形式，如 `{"topic": "AI"}`；若变量已通过`partial`预先填充，可不传）
2. **格式化**：模板将变量值插入到预定义的占位符位置
3. **输出**：生成 `PromptValue` 对象（一种中间格式）
4. **调用**：`PromptValue` 可以自动适配聊天模型（转为消息列表）或LLM（转为字符串）

这种统一的输入输出机制，使得所有模板都可以在LCEL链中无缝使用（LCEL将在第三节详细介绍）。

### 1.4 PromptTemplate

这是最基础的提示词模板类型，适用于简单的文本生成场景——即**字符串提示模板**。它接收一个包含占位符的字符串模板，通过传入变量值来生成最终的Prompt。

#### 1.4.1 基本用法

```python
from langchain_core.prompts import PromptTemplate

# 方式一：使用from_template
prompt = PromptTemplate.from_template(
    "讲一个关于{topic}的{adjective}故事"
)

# 方式二：使用构造方法
prompt = PromptTemplate(
    template="讲一个关于{topic}的{adjective}故事",
    input_variables=["topic", "adjective"]
)

# 调用模板
formatted_prompt = prompt.invoke({"topic": "人工智能", "adjective": "有趣的"})
print(formatted_prompt)
# 输出: text='讲一个关于人工智能的有趣的故事'
```

#### 1.4.2 部分变量

当模板中有些变量的值是固定的或可以提前确定时，可以使用 `partial` 机制预先填充部分变量，减少后续调用时需要传入的参数数量。

```python
# 方式一：调用partial方法固定部分变量
prompt = PromptTemplate.from_template(
    "讲一个关于{topic}的{adjective}故事"
)
fixed_prompt = prompt.partial(adjective="有趣的")
print(fixed_prompt.invoke({"topic": "编程"}))
# 输出: text='讲一个关于编程的有趣的故事'

# 方式二：创建时直接指定partial_variables
prompt = PromptTemplate(
    template="请解释{concept}，使用{style}风格",
    input_variables=["concept"],
    partial_variables={"style": "简单易懂"}
)
print(prompt.invoke({"concept": "递归"}))
# 输出: text='请解释递归，使用简单易懂风格'
```

**如何选择？**

- **优先用 `partial()`：当“固定值”是在运行时才确定，或需要分阶段逐步补全时**
  - 例：来自配置文件/环境变量/用户选择后再固定；或先固定 `style`，后续再固定 `adjective`。
  - 优点：更灵活，适合链式构建与复用同一个基础模板。
- **优先用 `partial_variables`：当“固定值”在定义模板时就确定，且希望模板对象一创建就自带默认常量时**
  - 例：文档里明确规定“统一用简单易懂风格”；团队约定的固定前缀/固定语气。
  - 优点：模板更“自描述”，一眼能看出哪些变量是固定的，适合沉淀为通用模板。
- **简单决策表**
  - 固定值 **是否依赖运行时上下文**：是 → `partial()`；否 → `partial_variables`
  - 是否需要 **多次派生不同版本**（同一模板固定不同值）：是 → `partial()` 更方便
  - 是否想把“默认值/常量”**写死在模板定义处**：是 → `partial_variables`

### 1.5 ChatPromptTemplate

`PromptTemplate` 生成的是一个纯字符串Prompt，而在多轮对话场景中，大模型需要接收的是**消息列表**（包含角色信息的多条消息）。`ChatPromptTemplate`——即**聊天提示模板**——正是为此设计的，它可以构建包含 `system`、`human`、`ai` 等不同角色的消息序列。

#### 1.5.1 基本用法

```python
from langchain_core.prompts import ChatPromptTemplate

# 使用from_messages构造（推荐）
chat_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个专业的{role}。"),
    ("human", "请回答关于{topic}的问题。"),
    ("ai", "好的，我会尽力回答。"),
    ("human", "{question}")
])

# 调用模板，生成消息列表
messages = chat_prompt.invoke({
    "role": "Python编程助手",
    "topic": "Python装饰器",
    "question": "什么是装饰器？"
})
print(messages)
```

其中，元组的第一个元素是**角色标识**，第二个元素是**消息内容模板**。LangChain支持的角色标识包括：

| 角色标识 | 对应消息类型 | 用途 |
|---------|-------------|------|
| `"system"` | `SystemMessage` | 设定AI的行为规范和角色 |
| `"human"` | `HumanMessage` | 用户输入的消息 |
| `"ai"` | `AIMessage` | AI的回复消息 |

#### 1.5.2 使用Message对象

除了元组形式，也可以直接使用Message对象来构造模板。两种方式可以混合使用：

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

chat_prompt = ChatPromptTemplate.from_messages([
    SystemMessage(content="你是一个有帮助的AI助手"),    # 固定内容用Message对象
    HumanMessage(content="你好！"),                     # 固定内容
    AIMessage(content="你好！有什么可以帮助你的？"),      # 固定内容
    ("human", "请介绍{topic}")                          # 含变量的用元组形式
])

messages = chat_prompt.invoke({"topic": "LangChain"})
print(messages)
```

> **选择建议**：如果消息内容是固定的（不包含变量），用Message对象更直观；如果内容包含需要填充的变量，用元组形式更简洁。

### 1.6 MessagesPlaceholder

在实际的对话应用中，我们往往需要将**历史对话记录**动态插入到Prompt中。`MessagesPlaceholder` 允许你在模板中预留一个"插槽"，运行时将一个消息列表整体插入到该位置。

这在构建带记忆的对话系统时尤为常用。

MessagesPlaceholder---->BaseMessagePromptTemplate

PromptTemplate-----> StringPromptTemplate---->BasePromptTemplate

ChatPromptTemplate---->BaseChatPromptTemplate---->BasePromptTemplate





```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import AIMessage, HumanMessage

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是AI助手"),
    MessagesPlaceholder(variable_name="history"),   # 对话历史插槽
    ("human", "{input}")
])

# 调用时传入历史消息列表
messages = prompt.invoke({
    "history": [
        HumanMessage(content="什么是Python？"),
        AIMessage(content="Python是一种通用编程语言。"),
    ],
    "input": "它有什么特点？"
})
print(messages)
```

> **注意**：history 变量在格式化的时候一定是一个列表类型

也可以使用更简洁的元组语法实现同样的效果：

```python
# 等价写法，使用("placeholder", ...)语法
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是AI助手"),
    ("placeholder", "{history}"),    # 等同于MessagesPlaceholder
    ("human", "{input}")
])
```

### 1.7 FewShotPromptTemplate

前面介绍的模板解决了"如何动态构建Prompt"的问题。但在实际应用中，我们经常需要通过**提供示例**来引导模型理解任务模式——这就是**少样本提示模板**。`FewShotPromptTemplate` 将示例数据和格式化模板组合在一起，自动生成包含示例的完整Prompt。

#### 1.7.1 基本用法

```python
from langchain_core.prompts import FewShotPromptTemplate, PromptTemplate

# 第一步：准备示例数据
examples = [
    {"input": "高兴", "output": "开心"},
    {"input": "难过", "output": "悲伤"},
    {"input": "生气", "output": "愤怒"}
]

# 第二步：定义单条示例的格式化模板
example_formatter = PromptTemplate(
    template="输入: {input}\n输出: {output}",
    input_variables=["input", "output"]
)

# 第三步：创建少样本提示模板
few_shot_prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_formatter,
    prefix="以下是一些同义词转换的例子：",    # 示例前的说明文字
    suffix="\n输入: {input}\n输出:",         # 示例后的实际问题
    input_variables=["input"]
)

# 调用
print(few_shot_prompt.invoke({"input": "兴奋"}))
```

生成的Prompt结构为：`prefix + 示例1 + 示例2 + ... + suffix`。



#### 1.7.2 对接LLM

```python
from langchain_core.prompts import FewShotPromptTemplate, PromptTemplate
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

load_dotenv()
# 第一步：准备示例数据
examples = [
    {"input": "高兴", "output": "开心"},
    {"input": "难过", "output": "悲伤"},
    {"input": "生气", "output": "愤怒"}
]

# 第二步：定义单条示例的格式化模板
example_formatter = PromptTemplate(
    template="输入: {input}\n输出: {output}",
    input_variables=["input", "output"]
)

# 第三步：创建少样本提示模板
few_shot_prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_formatter,
    prefix="以下是一些同义词转换的例子：",  # 示例前的说明文字
    suffix="\n输入: {input}\n输出:",  # 示例后的实际问题
    input_variables=["input"]
)

# 第四步：格式化
prompt_value = few_shot_prompt.invoke({"input": "兴奋"})

# 第五步：定义LLM
llm = ChatOpenAI(model="gpt-4o-mini")

# 第六步：调用
res = llm.invoke(prompt_value)

print(res.content)

# 输出：激动
```



### 1.8 提示词模板最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **清晰的指令** | 明确告诉模型要做什么 | "请翻译..." → "将以下文本从中文翻译成英文..." |
| **提供示例** | 使用少样本学习引导模型 | 使用 `FewShotPromptTemplate` |
| **角色设定** | 使用系统消息设定角色 | "你是一个有10年经验的Python工程师" |
| **约束条件** | 明确限制和约束 | "不超过100字"、"只回答与Python相关的问题" |

> **小结**：第一节介绍了LangChain提供的各类提示词模板。从简单的 `PromptTemplate`（字符串模板）到 `ChatPromptTemplate`（聊天模板），再到 `FewShotPromptTemplate`（少样本模板）它们解决的核心问题是**如何灵活、可维护地构建Prompt**。有了结构化的Prompt之后，下一个问题自然是：模型返回的结果如何解析为我们需要的结构化数据？这就是第二节要讨论的输出解析器。

---

## 二、输出解析器（Output Parsers）

### 2.1 为什么需要输出解析器

在对话场景中，大模型直接返回自然语言文本即可。但在实际生产环境中，我们通常需要将大模型用于**非对话场景**（如数据提取、内容生成流水线等），此时需要模型以结构化的格式（如JSON、列表）输出结果，以便程序进一步处理。

LangChain在 `langchain_core.output_parsers` 包中提供了一系列输出解析器，专门解决"**如何将模型的自然语言输出转换为结构化数据**"这一问题。

要让大模型输出结构化数据，有两种基本策略：

| 策略 | 方式 | 可靠性 | 适用场景 |
|------|------|--------|----------|
| **Prompt约束** | 在提示词中要求模型输出指定格式 | 依赖模型能力，可能出现格式错误 | 通用，任何模型都支持 |
| **厂商原生能力** | 使用API提供的结构化输出参数 | 由API层面保证格式正确 | 需要厂商支持（OpenAI、Google等） |

下面分别介绍这两种策略的具体实现。

### 2.2 策略一：通过Prompt约束（JsonOutputParser）

`JsonOutputParser` 的工作原理是：将你定义的JSON Schema转化为一段格式说明文字，插入到Prompt中，引导模型按照指定结构输出JSON。

**使用步骤**：
1. 通过Pydantic定义目标JSON结构
2. 构造 `JsonOutputParser` 实例
3. 调用 `get_format_instructions()` 获取格式说明，插入到Prompt中
4. 调用模型并用解析器解析结果

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field

llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.0,
    base_url=os.getenv("OPENAI_BASE_URL"),
    api_key=os.getenv("OPENAI_API_KEY")
)

# 1. 定义目标JSON结构
class Prime(BaseModel):
    prime: list[int] = Field(description="素数")
    count: list[int] = Field(description="小于该素数的素数个数")

# 2. 构造解析器
json_parser = JsonOutputParser(pydantic_object=Prime)

# 3. 将格式说明放入SystemMessage
res = llm.invoke([
    ("system", json_parser.get_format_instructions()),
    ("user", "任意生成5个1000-100000之间的素数，并标出小于该素数的素数个数")
])
print(res.content)

# 4. 解析为Python字典
parsed_res = json_parser.invoke(res)
print(type(parsed_res))  # <class 'dict'>
```

> **注意**：
>
> 1、Prompt约束依赖模型的理解能力。参数量较小的模型可能输出不规范的JSON，导致解析失败。对于生产环境，建议优先使用策略二。
>
> 2、`partial` 用来控制“是否允许解析**不完整/中途**的 JSON 输出”。
>
> - `partial=True`：用于**流式输出/增量生成**场景。此时模型可能只吐出了半段 JSON（还没闭合括号、还缺字段）。解析时：
>   - 能解析成“当前已生成的部分 JSON”就返回（`parse_json_markdown` 尝试提取并解析）。
>   - 还解析不了就**返回 `None`**，不抛异常，等后续 token 来了再继续解析。
> - `partial=False`（默认）：用于**最终结果**场景。要求输出必须是**完整合法 JSON**。
>   - 解析失败就抛 `OutputParserException`，提示 `Invalid json output`，便于你立刻发现模型没按格式输出。
>
> 一句话：`partial=True` 适配“边生成边解析”；`partial=False` 适配“生成完一次性严格校验”。

### 2.3 策略二：通过厂商原生能力

主流大模型厂商的API已经提供了专门的参数，在**API层**面强制模型输出符合指定Schema的结构化数据，比Prompt约束更加可靠。

**OpenAI示例**：

```python
from openai import OpenAI
from pydantic import BaseModel

client = OpenAI()

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

response = client.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "Alice and Bob are going to a science fair on Friday."}
    ],
    response_format=CalendarEvent
)

print(response.choices[0].message.parsed)
# CalendarEvent(name='Science Fair', date='Friday', participants=['Alice', 'Bob'])
```

> **注意：**它不是给模型“输入的参数”，而是告诉 API：把模型生成的内容按你提供的结构（这里是 `CalendarEvent` 这个 Pydantic 模型）进行**约束/引导生成 + 结果解析与校验**，并把解析后的对象放到 `response.choices[0].message.parsed`。
>
> 输入阶段仍然是 `messages`；`response_format` 影响的是模型该如何组织输出

**Google Gemini示例**：

```python
from google import genai
from pydantic import BaseModel

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

client = genai.Client(
        api_key="sk-OdqRypFlfJLKYvLmV6GsG9j0u6CRFBYKErn4xV1Wm0R3q0y9"，
        http_options={
            "base_url": "https://api.openai-proxy.org/google"
        },
    )

response = client.models.generate_content(
    model="gemini-2.5-flash-lite",
    contents="Alice and Bob are going to a science fair on Friday.",
    config={
        "response_mime_type": "application/json",
        "response_json_schema": CalendarEvent.model_json_schema(),
    },
)

event = CalendarEvent.model_validate_json(response.text)
print(event)
```

> `CalendarEvent.model_json_schema()`：**输出阶段的约束/说明**。把 Pydantic 模型 `CalendarEvent` 转成 **JSON Schema**（字段名、类型、必填项等），传给模型接口用于要求模型按该结构生成 JSON。
>
> `CalendarEvent.model_validate_json(response.text)`：**输出阶段的解析+校验**。把模型返回的 `response.text`（JSON 字符串）解析成 `CalendarEvent` 实例，并按模型规则做类型/必填校验；不符合会抛 `ValidationError`。

可以看到，不同厂商的调用方式各不相同。这正是LangChain封装 `with_structured_output` 的动机。

### 2.4 LangChain统一封装：with_structured_output

LangChain提供了 `with_structured_output()` 方法，将不同厂商的结构化输出能力统一到同一个接口下。无论底层使用哪家模型，调用方式完全一致：

```python
import os
from langchain_openai import ChatOpenAI
from pydantic import BaseModel

# 1. 初始化LLM
llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.0,
    base_url=os.getenv("OPENAI_BASE_URL"),
    api_key=os.getenv("OPENAI_API_KEY")
)

# 2. 定义Pydantic模型
class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

# 3. 使用with_structured_output，返回一个新的Runnable
structured_llm = llm.with_structured_output(schema=CalendarEvent)

# 4. 调用，直接返回Pydantic对象
result = structured_llm.invoke("Alice and Bob are going to a science fair on Friday.")
print(result)       # CalendarEvent(name='Science Fair', date='Friday', participants=['Alice', 'Bob'])
print(type(result)) # <class 'CalendarEvent'>
```

**核心优势**：如果后续需要更换模型（如从OpenAI切换到Anthropic），只需修改LLM的初始化代码，`with_structured_output` 的调用方式无需任何改动。

### 2.5 其他常用输出解析器

除了JSON解析，LangChain还提供了多种面向不同数据格式的输出解析器。

https://reference.langchain.com/python/langchain-core/output-parsers

#### 2.5.1 StrOutputParser

最简单的解析器，将模型输出的 `AIMessage` 对象提取为纯字符串。它是构建链时最常用的解析器：

```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()
# 通常在LCEL链中使用：chain = prompt | llm | StrOutputParser()
```

#### 2.5.2 PydanticOutputParser

在介绍这个解析器之前，我们需要先了解 **Pydantic** 是什么。

**Pydantic简介**

Pydantic 是 Python 中最流行的数据验证库，它通过 Python 的类型注解自动进行数据校验和转换。简单来说，你可以用它定义一个"数据模型类"，Pydantic 会确保传入的数据符合你定义的结构和类型。

**最小使用案例**：

```python
from pydantic import BaseModel, Field

# 定义一个数据模型
class User(BaseModel):
    name: str                    # 名字必须是字符串
    age: int                     # 年龄必须是整数
    email: str = Field(description="用户的邮箱地址")

# 创建实例 - 数据会自动校验和转换
user = User(name="张三", age=25, email="zhangsan@example.com")
print(user.name)   # 张三
print(user.age)    # 25

# 类型不匹配时，Pydantic会自动尝试转换
user2 = User(name="李四", age="30")  # age是字符串"30"，会自动转为整数30
print(user2.age)   # 30 (已自动转换为int)

# 数据不合法时会抛出错误
# User(name="王五", age="abc")  # 报错：无法将"abc"转为整数
```

Pydantic的核心价值：
- **类型安全**：自动校验数据类型
- **自动转换**：能转换的类型会自动处理（如字符串"30" → 整数30）
- **清晰的错误提示**：校验失败时给出详细的错误信息
- **JSON支持**：可以轻松与JSON数据互相转换

---

了解了Pydantic之后，`PydanticOutputParser` 的作用就很容易理解了——它让大模型输出符合Pydantic模型定义的结构化数据，并返回经过校验的Pydantic对象实例。

与 `JsonOutputParser` 返回普通字典不同，`PydanticOutputParser` 返回的是Pydantic对象，因此支持字段级别的校验规则：

```python
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field, field_validator
from langchain_openai import ChatOpenAI

class MovieReview(BaseModel):
    """电影评论结构"""
    title: str = Field(description="电影标题")
    rating: int = Field(description="评分，1-10分", ge=1, le=10)
    summary: str = Field(description="剧情简介")
    recommended: bool = Field(description="是否推荐")

    @field_validator('rating')
    @classmethod
    def rating_must_be_valid(cls, v):
        if v < 1 or v > 10:
            raise ValueError('评分必须在1-10之间')
        return v

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = PydanticOutputParser(pydantic_object=MovieReview)

# 将格式说明注入到Prompt中
prompt = ChatPromptTemplate.from_messages([
    ("system", parser.get_format_instructions()),
    ("human", "评价电影《盗梦空间》")
])

chain = prompt | llm | parser
result = chain.invoke({})
print(f"电影: {result.title}, 评分: {result.rating}/10")
```

> - `chain = prompt | llm | parser`
> - `parser`（`PydanticOutputParser`）拿到 LLM 输出后，会把文本解析成 dict/JSON，然后执行 `MovieReview.model_validate(...)`
> - 在 `model_validate` 过程中，Pydantic 会对字段做校验：先做内置约束（`ge=1, le=10`），再运行你定义的 `@field_validator('rating')`，因此 `rating_must_be_valid` 此时被调用
> - 校验失败会抛异常（LangChain 通常包装成 `OutputParserException`）

> `_PYDANTIC_FORMAT_INSTRUCTIONS` vs `JSON_FORMAT_INSTRUCTIONS` 区别
>
> 两者都是**给模型的“写作要求”**，目的都是让模型输出“符合 schema 的 JSON 文本”。
>
> - `_PYDANTIC_FORMAT_INSTRUCTIONS`：告诉你“按 schema 输出 JSON”，但**没强力禁止**你加解释、加 Markdown code block。
>
> - `JSON_FORMAT_INSTRUCTIONS`
>
>   ：更严格，明确要求：
>
>   - **只能输出 JSON**
>   - **不能有任何额外文字**
>   - **不能用 ``` 包起来**
>   - 必须是**单一顶层 JSON 值**
>
> 所以关键点是：
> **两个都是提示词（输入的一部分）**；它们约束的是模型的**文本输出格式**。输出先是 JSON 文本，之后才可能被解析成 Python 对象。

### 2.6 自定义输出解析器

当内置解析器无法满足需求时，可以继承 `BaseOutputParser` 创建自定义解析器：

```python
from langchain_core.output_parsers import BaseOutputParser

class CommaListParser(BaseOutputParser):
    """将逗号分隔的文本解析为列表"""

    def parse(self, text: str):
        # 去除空白后按逗号分割
        return [item.strip() for item in text.split(",")]

# 使用
parser = CommaListParser()
result = parser.parse("苹果, 香蕉, 橘子")
print(result)  # ['苹果', '香蕉', '橘子']
```

### 2.7 输出解析器选型指南

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| **生产环境结构化输出** | `with_structured_output()` | 厂商原生支持，格式有保证 |
| **通用JSON提取** | `JsonOutputParser` | 灵活，不依赖特定厂商 |
| **需要字段校验** | `PydanticOutputParser` | 支持Pydantic校验规则 |
| **简单文本输出** | `StrOutputParser` | 最轻量，无额外开销 |
| **特殊格式** | 自定义 `BaseOutputParser` | 完全可控 |

> **小结**：第二节围绕"如何解析模型输出"，介绍了从Prompt约束到厂商原生能力，再到LangChain统一封装的多种方案。在实际开发中，我们需要将提示词模板、模型调用、输出解析这三个步骤**串联**起来使用。如何优雅地组合这些组件？这就是第三节的核心主题——Chains链式调用。

---

## 三、Chains 链式调用

在开发大模型应用时，单一组件的能力有限。真正强大的应用需要将多个组件组合起来，形成完整的工作流程。Chain（链）正是解决这个问题的核心机制。

### 3.1 Chain的核心概念

**什么是Chain？**

简单来说，**Chain是将多个组件按特定顺序组合起来完成复杂任务的工作流或管道（Pipeline）**。

你可以把它想象成一个流水线：原材料（输入）经过一系列加工步骤（LLM调用、工具使用、数据转换等），最终成为成品（输出）。

**为什么需要Chain？**

虽然LLM本身很强大，但大多数实际应用需要多个步骤：

```
1. 接收用户输入
2. 根据输入构建提示（Prompt）
3. 将提示发送给LLM
4. 解析LLM的输出
5. 根据输出可能再执行其他操作（如调用API、查询数据库等）
6. 将最终结果返回给用户
```

Chain的本质是**自动化和封装**——让我们能像搭积木一样构建复杂AI应用，使用起来却像调用一个函数一样简单。

### 3.2 Runnable接口

LangChain所有核心组件都实现了统一的 **Runnable接口**。这是LangChain最底层的抽象，代表一个"可以被调用、批处理、流式传输和组合的工作单元"。

- **定位**：LangChain中的抽象基类（ABC）
- **核心理念**："一切可执行的对象都应该有统一的调用方式"

#### Runnable核心方法

| 方法 | 作用 | 适用场景 |
|------|------|----------|
| `invoke` | 同步调用单个输入 | 简单单次请求 |
| `batch` | 批量处理多个输入 | 批量数据处理 |
| `stream` | 流式处理输入 | 实时输出展示 |
| `ainvoke` | 异步调用单个输入 | 异步编程环境 |
| `abatch` | 异步批量处理 | 高性能批量处理 |

#### 哪些组件是Runnable？

几乎所有的LangChain核心组件都实现了Runnable接口：

| 组件类型 | 示例 |
|----------|------|
| **提示模板** | `PromptTemplate`, `ChatPromptTemplate` |
| **语言模型** | `ChatOpenAI`, `ChatOllama` |
| **输出解析器** | `StrOutputParser`, `JsonOutputParser` |
| **工具** | `Tool` |
| **整个链** | 任何LCEL链 |

#### 统一调用的价值

有了统一的Runnable接口，所有组件都可以用相同的方式调用：

```python
prompt.invoke({"topic": "AI"})        # 提示模板
model.invoke(prompt_value)            # 语言模型
parser.invoke(ai_message)             # 输出解析器
chain.invoke({"question": "你好"})    # 整个链
```

**本质**：接口统一让组件具备了"即插即用"的能力，这直接催生了LCEL。

### 3.3 LCEL表达式语言

**什么是LCEL？**

LCEL（LangChain Expression Language）是LangChain提供的**声明式组合语言**，专门用于组合Runnable组件。

- **核心操作符**：管道符 `|`
- **核心思想**：使用 `|` 将多个Runnable像拼积木一样组合起来

```python
# 典型的LCEL链式写法
chain = prompt | model | output_parser

# Chain本身也是Runnable，可以继续调用
result = chain.invoke({"topic": "编程"})# 1 prompt.invoke  2. model.invoke 3. output_parser.invoke

```

#### 基础语法示例

**两个组件连接**：

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model

prompt = ChatPromptTemplate.from_template("请用{tone}风格回答：{question}")
model = init_chat_model(model="gpt-3.5-turbo")

# LCEL组合 - 像拼积木一样简单
chain = prompt | model

chain.invoke({
    "tone": "幽默",
    "question": "什么是人工智能？"
})
```

**三个组件连接**：

```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()

# LCEL组合
chain = prompt | model | parser

chain.invoke({
    "tone": "幽默",
    "question": "什么是人工智能？"
})
```

#### LCEL的核心优势

| 优势 | 说明 |
|------|------|
| **组合后的链自动拥有所有Runnable能力** | 支持invoke/stream/batch等全部方法 |
| **链本身也是Runnable，可以继续组合** | 可以将链作为组件，构建更复杂的链 |
| **声明式方式，可读性更高** | 管道符连接，逻辑一目了然 |

### 3.4 Runnable组合器

LCEL之所以强大，是因为它背后有丰富的**Runnable组合器**。这些组合器让我们能够构建各种复杂逻辑。

#### 3.4.1 RunnableSequence - 顺序流水线

**作用**：将多个Runnable按顺序连接成处理管道，前一个的输出作为后一个的输入。

```python
from langchain_core.runnables import RunnableSequence, RunnableLambda

# 显式创建序列
sequence_chain = RunnableSequence(
    first=RunnableLambda(lambda x: x.upper()),           # 第一步：转换大写
    middle=[RunnableLambda(lambda x: f"HELLO {x} !")],   # 第二步：加装饰
    last=RunnableLambda(lambda x: f"最终：{x}")          # 第三步：加前缀
)

result = sequence_chain.invoke("world")
print(result)  # 最终：HELLO WORLD !
```

**LCEL等价写法**（推荐）：

```python
chain = (
    RunnableLambda(lambda x: x.upper())
    | RunnableLambda(lambda x: f"HELLO {x} !")
    | RunnableLambda(lambda x: f"最终：{x}")
)
```

> **说明**：LCEL的 `|` 运算符底层就是创建 `RunnableSequence`，推荐使用更简洁的LCEL写法。

#### 3.4.2 RunnableParallel - 并行分叉

**作用**：同时执行多个Runnable，将结果合并为一个字典。

```python
from langchain_core.runnables import RunnableParallel, RunnableLambda

# 创建并行任务
parallel_chain = RunnableParallel({
    "length": RunnableLambda(lambda x: len(x)),           # 计算长度
    "uppercase": RunnableLambda(lambda x: x.upper()),     # 转大写
    "reversed": RunnableLambda(lambda x: x[::-1]),        # 反转字符串
    "word_count": RunnableLambda(lambda x: len(x.split())) # 单词计数
})

result = parallel_chain.invoke("Hello World LangChain")
# 输出：
# {
#   'length': 24,
#   'uppercase': 'HELLO WORLD LANGCHAIN',
#   'reversed': 'niahCgnaL dlroW olleH',
#   'word_count': 3
# }
```

**工作原理：**

1. **同一份输入广播**
   调用 `invoke(x)` 时，它把**同一个输入 `x`** 传给字典里每个 runnable（`length/uppercase/...`）。

2. **并发执行**
   这些 runnable 会被调度为**并行/并发**运行（在 LangChain 的 runnable 运行时里实现；对纯 `RunnableLambda` 这种本地计算，通常是并发调度；对网络 I/O（LLM/检索）收益更明显）。

3. **收集并合并结果**
   等全部分支都完成后，把每个分支的输出按 key 组装成一个 dict

4. ```python
   {
     "length": <length的结果>,
     "uppercase": <uppercase的结果>,
     ...
    }
   ```

**LCEL等价写法**（推荐）：

```python
# 字典语法自动创建RunnableParallel
chain = {
    "length": RunnableLambda(lambda x: len(x)),
    "uppercase": RunnableLambda(lambda x: x.upper()),
    "reversed": RunnableLambda(lambda x: x[::-1]),
    "word_count": RunnableLambda(lambda x: len(x.split()))
}
# 不能单独用，单独用实际是个字典不能invoke,如果想使用一定要是一个链chain
```

**LCEL 字典语法为什么等价？**
在 LCEL 中，链里出现一个字典就会被自动“编译”为 `RunnableParallel`；也就是说 `{key: runnable}` 是一种语法糖，运行时机制与 `RunnableParallel({...})` 相同。

> **说明**：每个分支是**相互独立**的；如果你想“某个分支依赖另一个分支的结果”，就不适合并行分叉，而要用顺序链（`|`）或先并行后再用一个 `RunnableLambda` 做汇总/计算。



**实际应用**：先并行生成多个结果，再汇总：

```python
from langchain.chat_models import init_chat_model
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = init_chat_model(model="gpt-4o-mini", model_provider="openai")

# 两个并行的赏析链
paragraph_1_chain = (
    PromptTemplate.from_template("对这首诗做赏析，分析含义：{poem}")
    | llm | StrOutputParser()
)
paragraph_2_chain = (
    PromptTemplate.from_template("对这首诗做赏析，分析意境：{poem}")
    | llm | StrOutputParser()
)

# 汇总链
summary_chain = (
    PromptTemplate.from_template(
        "第一种赏析：{paragraph_1}\n\n第二种赏析：{paragraph_2}\n\n请比较哪个更好，为什么"
    )
    | llm | StrOutputParser()
)

# 先并行，后汇总
full_chain = {
    "paragraph_1": paragraph_1_chain,
    "paragraph_2": paragraph_2_chain,
} | summary_chain

resp = full_chain.invoke({"poem": "菩提本无树，明镜亦非台，本来无一物，何处惹尘埃。"})
print(resp)
```

#### 3.4.3 RunnablePassthrough - 输入传递

**作用**：将输入原样传递到输出，常用于保留原始输入的同时添加新字段。

```python
from langchain_core.runnables import RunnablePassthrough, RunnableParallel

# 场景1：直接传递输入
chain = RunnablePassthrough()
result = chain.invoke({"key": "value"})
# 输出: {"key": "value"}

# 场景2：保留原始输入 + 添加新字段
chain = RunnableParallel(
    original=RunnablePassthrough(),            # 保留原始输入
    uppercase=lambda x: x["text"].upper()      # 添加转换后的字段
)
result = chain.invoke({"text": "hello"})
# 输出: {"original": {"text": "hello"}, "uppercase": "HELLO"}
```

**RAG典型用法**——保留问题 + 检索上下文：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template(
    "请回答以下问题：{question}\n\n相关背景：{context}"
)

chain = (
    {
        "question": RunnablePassthrough(),           # 用户问题原样传递
        "context": lambda x: retrieve_context(x)     # 检索相关上下文
    }
    | prompt
    | llm
    | StrOutputParser()
)
```

#### 3.4.4 RunnableLambda - 自定义逻辑

**作用**：将普通Python函数包装为Runnable，使其可以在LCEL链中使用。

```python
from langchain_core.runnables import RunnableLambda

def extract_domain(url):
    """从URL中提取域名"""
    return url.split('//')[-1].split('/')[0]

def add_protocol(domain):
    """添加协议前缀"""
    return f"http://{domain}"

# 包装成Runnable
domain_extractor = RunnableLambda(extract_domain)
protocol_adder = RunnableLambda(add_protocol)

# 在链中使用
url_processor = domain_extractor | protocol_adder
result = url_processor.invoke("https://www.example.com/path")
# 输出："http://www.example.com"
```

### 3.5 Runnable方法一览

Runnable接口提供的完整方法集：

| 方法 | 说明 | 示例 |
|------|------|------|
| `invoke` | 同步单次调用 | `chain.invoke({"topic": "AI"})` |
| `stream` | 流式输出 | `for chunk in chain.stream({...})` |
| `batch` | 批量处理 | `chain.batch([{"topic": "AI"}, {"topic": "猫"}])` |
| `ainvoke` | 异步调用 | `await chain.ainvoke({...})` |
| `astream` | 异步流式 | `async for chunk in chain.astream({...})` |
| `abatch` | 异步批量 | `await chain.abatch([...])` |

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# 同步调用
response = llm.invoke("你好")

# 流式调用 —— 逐token输出
for chunk in llm.stream("讲一个故事"):
    print(chunk.content, end="")

# 批量调用 —— 同时处理多个输入
responses = llm.batch(["你好", "再见", "谢谢"])

# 异步调用
import asyncio
response = await llm.ainvoke("你好")
```

### 3.6 添加对话历史

构建对话系统时，需要让链"记住"之前的对话内容：

```python
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI

# 创建会话存储（以session_id为key）
store = {}


def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]


# 创建基础链
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是AI助手"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

llm = ChatOpenAI(model="gpt-4o-mini")
chain = prompt | llm

# 包装为带历史记录的链
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history"
)

# 使用时指定session_id
response_1 = chain_with_history.invoke(
    {"input": "我叫张三"},
    config={"configurable": {"session_id": "user123"}}
)
print(response_1.content)

# 后续对话会自动携带历史
response_2 = chain_with_history.invoke(
    {"input": "我叫什么名字？"},
    config={"configurable": {"session_id": "user123"}}
)
print(response_2.content)
# AI会回答"你叫张三"，因为历史记录中有这个信息
```

---

## 四、高级模式

### 4.1 错误处理与重试

```python
# 使用with_retry自动重试
chain_with_retry = prompt | llm.with_retry(stop_after_attempt=3) | parser
```

### 4.2 回退机制

主模型失败时自动切换备用模型：

```python
primary_llm = ChatOpenAI(model="gpt-4o")
fallback_llm = ChatOpenAI(model="gpt-4o-mini")

chain_with_fallback = prompt | primary_llm.with_fallbacks([fallback_llm]) | parser
```

---

## 五、小结

### 5.1 核心概念总结

| 主题 | 核心要点 | 关键API |
|------|----------|---------|
| **提示词模板** | 管理和参数化Prompt | `PromptTemplate`, `ChatPromptTemplate` |
| **输出解析器** | 将模型输出转为结构化数据 | `StrOutputParser`, `JsonOutputParser` |
| **结构化输出** | 厂商原生能力保证格式 | `with_structured_output()` |
| **Runnable接口** | 统一的调用方式 | `invoke`, `stream`, `batch` |
| **LCEL** | 管道符组合组件 | `chain = prompt | llm | parser` |
| **Runnable组合器** | 构建复杂逻辑 | `RunnableParallel`, `RunnablePassthrough` |

### 5.2 代码模板速查

```python
# 基础链
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"topic": "AI"})

# 结构化输出
class Result(BaseModel):
    answer: str
    confidence: float

structured_llm = llm.with_structured_output(Result)

# 并行链
parallel = {
    "en": translate_chain_en,
    "kr": translate_chain_kr
}

# 带对话历史
chain_with_history = RunnableWithMessageHistory(chain, get_session_history, ...)
```

### 5.3 选型建议

| 场景 | 推荐 |
|------|------|
| 简单文本生成 | `StrOutputParser` |
| 结构化数据 | `with_structured_output` |
| 多轮对话 | `ChatPromptTemplate` + `RunnableWithMessageHistory` |
| 批量处理 | `batch()` |

---

## 六、参考资料

- [LangChain Prompts 官方文档](https://python.langchain.com/docs/concepts/prompt_templates/)
- [LangChain Output Parsers 文档](https://python.langchain.com/docs/concepts/output_parsers/)
- [LangChain Expression Language (LCEL)](https://python.langchain.com/docs/concepts/lcel/)
- [Runnable 接口文档](https://python.langchain.com/docs/concepts/runnables/)
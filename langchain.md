---
title: langchain
date: 2023-10-10 21:03:33
tags:
    - AIGC
    - 工具链
---
lang chain简介
<!-- more -->
# LangChain是什么？

> LangChain是一个基于语言模型开发应用程序的框架。它可以实现以下应用程序：
> 数据感知：将语言模型连接到其他数据源
> 自主性：允许语言模型与其环境进行交互
> LangChain的主要价值在于：

> 组件化：为使用语言模型提供抽象层，以及每个抽象层的一组实现。组件是模块化且易于使用的，无论您是否使用LangChain框架的其余部分。
> 现成的链：结构化的组件集合，用于完成特定的高级任务

简而言之，就是提供基于各种大模型开发应用的工具链。
# LangChain 关键组件
## Models
### Chat Modals
目前支持 AIMessage、HumanMessage、SystemMessage 和 ChatMessage。
Chat可以是任意角色的， System是提供背景，设定AI角色的，Human是人类给的提问，AI是LLM给的回答
```python
# 导入OpenAI的聊天模型，及消息类型
from langchain.chat_models import ChatOpenAI
from langchain.schema import (
    AIMessage,
    HumanMessage,
    SystemMessage
)

# 初始化聊天对象
chat = ChatOpenAI(openai_api_key="...")

# 向聊天模型发问
chat([HumanMessage(content="Translate this sentence from English to French: I love programming.")])

# 多输入
messages = [
    SystemMessage(content="You are a helpful assistant that translates English to French."),
    HumanMessage(content="I love programming.")
]
chat(messages)
# 批量输入
batch_messages = [
    [
        SystemMessage(content="You are a helpful assistant that translates English to French."),
        HumanMessage(content="I love programming.")
    ],
    [
        SystemMessage(content="You are a helpful assistant that translates English to French."),
        HumanMessage(content="I love artificial intelligence.")
    ],
]
result = chat.generate(batch_messages)
result
```
省钱小妙招：缓存结果，LangChain提供了两种缓存方案，内存缓存方案和数据库缓存方案，当然支持的数据库缓存方案有很多种。
```python
# 导入聊天模型，SQLiteCache模块
import os
os.environ["OPENAI_API_KEY"] = 'your apikey'
import langchain
from langchain.chat_models import ChatOpenAI
from langchain.cache import SQLiteCache

# 设置语言模型的缓存数据存储的地址
langchain.llm_cache = SQLiteCache(database_path=".langchain.db")

# 加载 llm 模型
llm = ChatOpenAI()

# 第一次向模型提问
result = llm.predict('tell me a joke')
print(result)

# 第二次向模型提问同样的问题
result2 = llm.predict('tell me a joke')
print(result2)
```
### Embeddings
用于文档、文本或者大量数据的总结、问答场景，一般是和向量库一起使用，实现向量匹配。其实就是把文本等内容转成多维数组，可以后续进行相似性的计算和检索。他相比 fine-tuning 最大的优势就是，不用进行训练，并且可以实时添加新的内容，而不用加一次新的内容就训练一次，并且各方面成本要比 fine-tuning 低很多。
### LLMs
大模型
## Prompts

Prompts用来管理 LLM 输入的工具，在从 LLM 获得所需的输出之前需要对提示进行相当多的调整，最终的Promps可以是单个句子或多个句子的组合，它们可以包含变量和条件语句。
### Prompt Templates
生成一个模板，往里填关键字即可，例子如下：
```python
from langchain.llms import OpenAI

# 定义生成商店的方法
def generate_store_names(store_features):
    prompt_template = "我正在开一家新的商店，它的主要特点是{}。请帮我想出10个商店的名字。"
    prompt = prompt_template.format(store_features)

    llm = OpenAI()
    response = llm.generate(prompt, max_tokens=10, temperature=0.8)

    store_names = [gen[0].text.strip() for gen in response.generations]
    return store_names

store_features = "时尚、创意、独特"

store_names = generate_store_names(store_features)
print(store_names)
```
### Few-shot example
少量样本，即prompt templates的批量版本
```python
import os
os.environ["OPENAI_API_KEY"] = 'your apikey'
from langchain import PromptTemplate, FewShotPromptTemplate
from langchain.llms import OpenAI

examples = [
    {"word": "黑", "antonym": "白"},
    {"word": "伤心", "antonym": "开心"},
]

example_template = """
单词: {word}
反义词: {antonym}\\n
"""

# 创建提示词模版
example_prompt = PromptTemplate(
    input_variables=["word", "antonym"],
    template=example_template,
)

# 创建小样本提示词模版
few_shot_prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="给出每个单词的反义词",
    suffix="单词: {input}\\n反义词:",
    input_variables=["input"],
    example_separator="\\n",
)

# 格式化小样本提示词
prompt_text = few_shot_prompt.format(input="粗")

# 调用OpenAI
llm = OpenAI(temperature=0.9)

print(llm(prompt_text))
```
### Example Selector
这个有意思，可以选择距离较近的prompt,例子是：
```python
import os
os.environ["OPENAI_API_KEY"] = 'your apikey'
from langchain.prompts import PromptTemplate, FewShotPromptTemplate
from langchain.prompts.example_selector import LengthBasedExampleSelector
from langchain.prompts.example_selector import LengthBasedExampleSelector


# These are a lot of examples of a pretend task of creating antonyms.
examples = [
    {"word": "happy", "antonym": "sad"},
    {"word": "tall", "antonym": "short"},
    {"word": "energetic", "antonym": "lethargic"},
    {"word": "sunny", "antonym": "gloomy"},
    {"word": "windy", "antonym": "calm"},
]
# 例子格式化模版
example_formatter_template = """
Word: {word}
Antonym: {antonym}\n
"""
example_prompt = PromptTemplate(
    input_variables=["word", "antonym"],
    template=example_formatter_template,
)

# 使用 LengthBasedExampleSelector来选择例子
example_selector = LengthBasedExampleSelector(
    examples=examples,
    example_prompt=example_prompt,
    # 最大长度
    max_length=25,
)

# 使用'example_selector'创建小样本提示词模版
dynamic_prompt = FewShotPromptTemplate(
    example_selector=example_selector,
    example_prompt=example_prompt,
    prefix="Give the antonym of every input",
    suffix="Word: {input}\nAntonym:",
    input_variables=["input"],
    example_separator="\n\n",
)

longString = "big and huge and massive and large and gigantic and tall and much much much much much bigger than everything else"

print(dynamic_prompt.format(input=longString))
```
官方也提供了根据最大边际相关性、文法重叠、语义相似性来选择示例
## Indexes
索引是指对文档进行结构化的方法，也就是把乱七八糟的数据结构化的过程。包括Document Loaders（文档加载器）、Text Splitters（文本拆分器）、VectorStores（向量存储器）以及 Retrievers（检索器）四个组件
### Document Loaders
指定源进行加载数据的。将特定格式的数据，转换为文本。如 CSV、File Directory、HTML、JSON、Markdown、PDF。另外使用相关接口处理本地知识，或者在线知识。如 AirbyteJSON、Airtable、Alibaba Cloud MaxCompute、wikipedia、BiliBili、GitHub、GitBook 等等。
### Text Splitters
由于模型对输入的字符长度有限制，我们在碰到很长的文本时，需要把文本分割成多个小的文本片段。
文本分割最简单的方式是按照字符长度进行分割，但是这会带来很多问题，比如说如果文本是一段代码，一个函数被分割到两段之后就成了没有意义的字符，所以整体的原则是把**语义相关**的文本片段放在一起。
LangChain 中最基本的文本分割器是 CharacterTextSplitter ，它按照指定的分隔符（默认“\n\n”）进行分割，并且考虑文本片段的最大长度。例子：
```python
from langchain.text_splitter import CharacterTextSplitter

# 初始字符串
state_of_the_union = "..."

text_splitter = CharacterTextSplitter(
    separator = "\\n\\n",
    chunk_size = 1000,
    chunk_overlap  = 200,
    length_function = len,
)

texts = text_splitter.create_documents([state_of_the_union])
```
这个仅仅是例子，LangChain支持多种高级文本分割器，包括根据代码语义、openAI的token数、markdown结构等切割文本。
### VectorStores
存储提取的文本向量，包括 Faiss、Milvus、Pinecone、Chroma 等。
### Retrievers
检索器是一种便于模型查询的存储数据的方式，LangChain 约定检索器组件至少有一个方法 get_relevant_texts，这个方法接收查询字符串，返回一组文档
```python
from langchain.chains import RetrievalQA
from langchain.llms import OpenAI
from langchain.document_loaders import TextLoader
from langchain.indexes import VectorstoreIndexCreator
loader = TextLoader('../state_of_the_union.txt', encoding='utf8')

# 对加载的内容进行索引
index = VectorstoreIndexCreator().from_loaders([loader])

query = "What did the president say about Ketanji Brown Jackson"

# 通过query的方式找到语义检索的结果
index.query(query)
```
## Chains

是一种将LLM和其他多个组件连接在一起的工具，以实现复杂的任务。类似于pytorch的sequence
链允许我们将多个组件组合在一起以创建一个单一的、连贯的任务。例如，我们可以创建一个链，它接受用户输入，使用 PromptTemplate 对其进行格式化，然后将格式化的响应传递给 LLM。另外我们也可以通过将多个链组合在一起，或者将链与其他组件组合来构建更复杂的链。
所有的chain方法共享call和run方法，call方法返回输入输出键值对（字典），run方法仅返回输出字符串
### LLMChain
LLMChain 是一个简单的链，它围绕语言模型添加了一些功能。它接受一个**提示模板**，将其与用户输入进行格式化，并返回 LLM 的响应。
```python
from langchain import PromptTemplate, OpenAI, LLMChain

prompt_template = "What is a good name for a company that makes {product}?"

llm = OpenAI(temperature=0)
llm_chain = LLMChain(
    llm=llm,
    prompt=PromptTemplate.from_template(prompt_template)
)
llm_chain("colorful socks")
```
LLMChain独有的方法
* apply 方法允许对一个输入列表进行调用，即批处理的call方法
* generate 方法跟apply方法类似，但返回LLMResult，包含更多的元信息，如令牌使用情况和完成原因
* predict 方法类似于 run 方法，不同之处在于输入键被指定为关键字参数，而不是一个 Python 字典。`llm_chain.predict(product="colorful socks")`

### SimpleSequentialChain
顺序链的最简单形式，其中每个步骤都有一个**单一的输入/输出**，并且一个步骤的输出是下一步的输入。
就是把LLMChain线性组合的工具，例如：
```python
from langchain.llms import OpenAI
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain.chains import SimpleSequentialChain

# 定义第一个chain
llm = OpenAI(temperature=.7)
template = """You are a playwright. Given the title of play, it is your job to write a synopsis for that title.

Title: {title}
Playwright: This is a synopsis for the above play:"""
prompt_template = PromptTemplate(input_variables=["title"], template=template)
synopsis_chain = LLMChain(llm=llm, prompt=prompt_template)

# 定义第二个chain

llm = OpenAI(temperature=.7)
template = """You are a play critic from the New York Times. Given the synopsis of play, it is your job to write a review for that play.

Play Synopsis:
{synopsis}
Review from a New York Times play critic of the above play:"""
prompt_template = PromptTemplate(input_variables=["synopsis"], template=template)
review_chain = LLMChain(llm=llm, prompt=prompt_template)

# 通过简单顺序链组合两个LLMChain
overall_chain = SimpleSequentialChain(chains=[synopsis_chain, review_chain], verbose=True)

# 执行顺序链
review = overall_chain.run("Tragedy at sunset on the beach")
```
### SequentialChain
相比 SimpleSequentialChain 只允许有单个输入输出，它是一种更通用的顺序链形式，允许**多个输入/输出**。

特别重要的是： 我们如何命名输入/输出变量名称。在上面的示例中，我们不必考虑这一点，因为我们只是将一个链的输出直接作为输入传递给下一个链，但在这里我们确实需要担心这一点，因为我们有多个输入。
例如：
```python
# 这是第一个 LLMChain，根据戏剧的标题和设定的时代，生成一个简介。
llm = OpenAI(temperature=.7)
template = """You are a playwright. Given the title of play and the era it is set in, it is your job to write a synopsis for that title.
# 这里定义了两个输入变量title和era，并定义一个输出变量：synopsis
Title: {title}
Era: {era}
Playwright: This is a synopsis for the above play:"""
prompt_template = PromptTemplate(input_variables=["title", "era"], template=template)
synopsis_chain = LLMChain(llm=llm, prompt=prompt_template, output_key="synopsis")


# 这是第二个 LLMChain，根据剧情简介撰写一篇戏剧评论。
llm = OpenAI(temperature=.7)
template = """You are a play critic from the New York Times. Given the synopsis of play, it is your job to write a review for that play.
# 定义了一个输入变量：synopsis，输出变量：review
Play Synopsis:
{synopsis}
Review from a New York Times play critic of the above play:"""
prompt_template = PromptTemplate(input_variables=["synopsis"], template=template)
review_chain = LLMChain(llm=llm, prompt=prompt_template, output_key="review")

# 执行顺序链：

overall_chain({"title":"Tragedy at sunset on the beach", "era": "Victorian England"})
```
执行结果会将每一步的输出都打印出来
```python
    > Entering new SequentialChain chain...

    > Finished chain.

    {'title': 'Tragedy at sunset on the beach',
     'era': 'Victorian England',
     'synopsis': "xxxxxx",
     'review': "xxxxxxx"}
```
### TransformChain
转换链允许我们创建一个自定义的**转换函数**来处理输入，将处理后的结果用作**下一个链的输入**。其实是类似于LLMChain的一个东西。
如下示例我们将创建一个转换函数，它接受超长文本，将文本过滤为仅前 3 段，然后将其传递到 LLMChain 中以总结这些内容。
```python
from langchain.chains import TransformChain, LLMChain, SimpleSequentialChain
from langchain.llms import OpenAI
from langchain.prompts import PromptTemplate

# 模拟超长文本
with open("../../state_of_the_union.txt") as f:
    state_of_the_union = f.read()

# 定义转换方法，入参和出参都是字典，取前三段
def transform_func(inputs: dict) -> dict:
    text = inputs["text"]
    shortened_text = "\n\n".join(text.split("\n\n")[:3])
    return {"output_text": shortened_text}

# 转换链：输入变量：text，输出变量：output_text
transform_chain = TransformChain(
    input_variables=["text"], output_variables=["output_text"], transform=transform_func
)
# prompt模板描述
template = """Summarize this text:

{output_text}

Summary:"""
# prompt模板
prompt = PromptTemplate(input_variables=["output_text"], template=template)
# llm链
llm_chain = LLMChain(llm=OpenAI(), prompt=prompt)
# 使用顺序链
sequential_chain = SimpleSequentialChain(chains=[transform_chain, llm_chain])
# 开始执行
sequential_chain.run(state_of_the_union)
# 结果
"""
    ' The speaker addresses the nation, noting that while last year they were kept apart due to COVID-19, this year they are together again.
    They are reminded that regardless of their political affiliations, they are all Americans.'

"""
```
## Agents

是一种使用LLM做出决策的工具，它们可以执行特定的任务并生成文本输出。Agents通常由三个部分组成：Action、Observation和Decision。Action是代理执行的操作，Observation是代理接收到的信息，Decision是代理基于Action和Observation做出的决策。
可以简单的理解为他可以**动态**的帮我们选择和调用 chain 或者已有的工具。代理主要有两种类型 Action agents 和 Plan-and-execute agents。
### 动态 Action agents
行为代理: 在每个时间步，使用所有先前动作的输出来决定下一个动作。下图展示了行为代理执行的流程。
### 静态 Plan-and-execute agents
预先决定完整的操作顺序，然后执行所有操作而不更新计划，下面是其流程。

1. 接收用户输入
2. 计划要采取的完整步骤顺序
3. 按顺序执行步骤，将过去步骤的输出作为未来步骤的输入传递
## Memory

是一种用于存储数据的工具，由于LLM 没有任何长期记忆，它有助于在多次调用之间保持状态。
langchain 提供了不同的 Memory 组件完成内容记忆，如下是目前提供的组件。
*  **ConversationBufferMemory**：将聊天内存缓存到内存中，不必每次手动拼接
* **ConversationBufferWindowMemory**：相比较第一个记忆组件，该组件增加了一个窗口参数，会保存最近k轮聊天内容。
* **ConversationTokenBufferMemory**：相比第一个，使用token而不是轮数来刷新缓冲区
* **ConversationSummaryMemory**：存储摘要而不是具体聊天内容
* **ConversationSummaryBufferMemory**：存储摘要，且根据token数量决定何时刷新
* **VectorStoreRetrieverMemory**：将聊天缓存到向量数据库中，并根据用户输入信息返回向量数据库中最相似的k组对话

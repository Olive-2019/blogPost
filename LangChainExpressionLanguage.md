---
title: LangChain Expression Language
date: 2023-10-16 15:27:33
tags:
    - LCEL
    - LangChain
---
LCEL
<!-- more -->
# LCEL是什么？有什么优点？
## 官方文档
先讲一下官方文档介绍的优点
1. 异步 批处理 流处理：设计、编码、小型测试（原型系统）的时候用同步接口，部署的时候用异步接口
2. fallback：LLM结果不可控，异常处理机制
3. 并行：LCEL默认组件都可以并行运行
4. 自动LangSmith（日志）：所有步骤都会自动打一些日志到langSmith中
## 直观感受
balabala这么一大通，最直观能感受到的优点是**简单**，简化了开发代码。先来直观感受一下LCEL
如果不用LCEL，搞一个根据用户输入的主题作诗的LLM应用，要写的代码如下：
```python
#1.导入需要用到的工具
from langchain import OpenAI, LLMChain
from langchain.prompts import ChatPromptTemplate

#2. 创建一个prompt template模板
template = """/
你是一个诗人，你需要根据用户输入的背景写一首诗。
这是用户输入的内容：
{content}
"""

prompt = PromptTemplate(
       template=template,
       input_variables = ["content"]
)

#3. 创建一个llm实例，
llm = OpenAI(temperature=0.7,model_name = 'gpt-3.5-turbo',openai_api_key='your api key')

#4. 创建一个llmchain实例 并插入这上述的配置
llm_chain = LLMChain(
    llm=llm,
    prompt=prompt,
)

#5. 调用一下这个llm实例
llm_chain("春天去郊游")
```
用了LCEL以后，要写的代码如下：
```python
from langchain import PromptTemplate, OpenAI, LLMChain

#1. 初始化prompt
prompt = ChatPromptTemplate.from_template("""/
你是一个诗人，你需要根据用户输入的背景写一首诗。
这是用户输入的内容：
{content}
""")

#2. 初始化llm model
llm = OpenAI(temperature=0.7,model_name = 'gpt-3.5-turbo',openai_api_key='your api key')

#3. 使用LCEL编排Chain, 使用bind函数修改llm的参数
llm_chain = prompt | llm

#4. 调用一下这个llm实例
llm_chain.invoke({"content": "春天去郊游"})
```
主要区别在于LCEL的第3和第4步，在第三步，LCEL使用“|”将llm和prompt以一种类似管道的方式串联了起来，在第4步，则使用invoke方法来进行输入和调用llm_chain。
根据Langchain的官方文档，LCEL可以认为是对Langchain之前所定义的一些组件（prompt，chain，agent等）进行了近一步的抽象，在这些组件的基础上抽象出了一个叫作“Runnable对象”的概念
# 接口
Runnable对象是LCEL的核心组成部分，Langchain官方对这样的对象统一定义了三个共同的接口，当对象和函数满足这三个接口时，便可被称之为一个Runnable对象。
1. **stream**方法：调用Runnable时以**流式**返回输出结果（格式好看一点？）
2. **invoke**方法：基于**单个**input调用Runnable
3. **batch**方法：基于一个list的input**批量**调用Runnable

上述方法都是同步的，前面加上a就是异步方法。
对于上述的三种方法，LCEL不同的Runnable对象支持接收不同的输入，并产生不同的输出，在Python中，能够使用“|”将前一个Runnable对象的输出传递到下一个Runnable对象作为输入，以此达到灵活自定义Prompt并实现“简洁地构建复杂LLM应用”的目的。
举个例子，对于LLM这种类型的Runnable，LLM本质是一个大语言模型，该类Runnable对象支持接收字符串类型的输入，并产生字符串类型的输入；对于Tool这种类型的Runnable，则接收一个字符串或者一个字典作为输入，依据Tool本身的构造形成不同的输出；这里值得注意的是，为了实现使用“|”将不同Runnable进行串联，我们还需要记住每个Runnable类支持的输入和输出，这就好比我们把不同型号的水管顺序接起来，我们需要知道每个水管两头的大小型号一样。



# 绑定运行时参数
可以通过`runnable.bind()`接口，给chain加入一个参数。
例如，用`prompt|model|OutputParser`的chain建立解数学题的应用：
```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema import StrOutputParser
from langchain.schema.runnable import RunnablePassthrough

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "Write out the following equation using algebraic symbols then solve it. Use the format\n\nEQUATION:...\nSOLUTION:...\n\n"),
        ("human", "{equation_statement}")
    ]
)
model = ChatOpenAI(temperature=0)
runnable = {"equation_statement": RunnablePassthrough()} | prompt | model | StrOutputParser()

print(runnable.invoke("x raised to the third plus seven equals 12"))
```
输出会很啰嗦，如下所示
```bash
    EQUATION: x^3 + 7 = 12
    
    SOLUTION:
    Subtracting 7 from both sides of the equation, we get:
    x^3 = 12 - 7
    x^3 = 5
    
    Taking the cube root of both sides, we get:
    x = ∛5
    
    Therefore, the solution to the equation x^3 + 7 = 12 is x = ∛5.
```
我现在只想要`SOLUTION`以上的输出，不要下面那一堆东西，那就给model加一个`stop`参数，让他的输出停在某个地方，例如：
```python
runnable = (
    {"equation_statement": RunnablePassthrough()} 
    | prompt 
    | model.bind(stop="SOLUTION") 
    | StrOutputParser()
)
print(runnable.invoke("x raised to the third plus seven equals 12"))
```
这个时候，输出就会变成
```bash
EQUATION: x^3 + 7 = 12
```
# fallback
## 单对象级别的异常处理
LLM的调用/输出可能会因为各种原因崩坏，比方说网络问题（伟大的城墙）、基座模型太拉（你可能调用了文心一言。。。）
加入fallback相当于一个try throw catch中的catch，在出错时给出一个后备选项。
给出下面的例子：
```python
from langchain.chat_models import ChatOpenAI, ChatAnthropic
from unittest.mock import patch
from openai.error import RateLimitError

# retry次数设成0，保证openai_llm肯定会出错
openai_llm = ChatOpenAI(max_retries=0)
anthropic_llm = ChatAnthropic()
# llm加入了一个fallback模型（后备模型）
llm = openai_llm.with_fallbacks([anthropic_llm])

# 直接调用openai model接口
with patch('openai.ChatCompletion.create', side_effect=RateLimitError()):
    try:
         print(openai_llm.invoke("Why did the chicken cross the road?"))
    except:
        print("Hit error")
'''
输出:
 Hit error
'''

# 调用加入了fallback的chain
with patch('openai.ChatCompletion.create', side_effect=RateLimitError()):
    try:
         print(llm.invoke("Why did the the chicken cross the road?"))
    except:
        print("Hit error")
'''
输出:
     content=" I don't actually know why the kangaroo crossed the road, but I'm happy to take a guess! Maybe the kangaroo was trying to get to the other side to find some tasty grass to eat. Or maybe it was trying to get away from a predator or other danger. Kangaroos do need to cross roads and other open areas sometimes as part of their normal activities. Whatever the reason, I'm sure the kangaroo looked both ways before hopping across!" additional_kwargs={} example=False
'''    
```
### 指定错误类型
LCEL允许指定处理何种类型的错误，如：
```python
llm = openai_llm.with_fallbacks([anthropic_llm], exceptions_to_handle=(KeyboardInterrupt,))

chain = prompt | llm
with patch('openai.ChatCompletion.create', side_effect=RateLimitError()):
    try:
         print(chain.invoke({"animal": "kangaroo"}))
    except:
        print("Hit error")
```

## 链级别的异常处理
进一步的异常处理，直接把序列给换了
例子：
```python
# 创建一个会发生异常的chain——bad_chain
from langchain.schema.output_parser import StrOutputParser

chat_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "You're a nice assistant who always includes a compliment in your response"),
        ("human", "Why did the {animal} cross the road"),
    ]
)

chat_model = ChatOpenAI(model_name="gpt-fake")
bad_chain = chat_prompt | chat_model | StrOutputParser()

# 创建一个不会发生异常的chain——good_chain
from langchain.llms import OpenAI
from langchain.prompts import PromptTemplate

prompt_template = """Instructions: You should always include a compliment in your response.

Question: Why did the {animal} cross the road?"""
prompt = PromptTemplate.from_template(prompt_template)
llm = OpenAI()
good_chain = prompt | llm

# 为bad chain添加后备序列good chain
chain = bad_chain.with_fallbacks([good_chain])

chain.invoke({"animal": "turtle"})
```

# 调用函数 
在chain中可以调用任意函数，用RunnableLambda声明即可，但是调用的函数必须是单输入的，如果要用多输入的话，多写一个adaptor调整成单输入的。
以下是一个用函数对输入进行映射处理的例子：
```python
from langchain.schema.runnable import RunnableLambda
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import ChatOpenAI
from operator import itemgetter

# 计算字符串长度
def length_function(text):
    return len(text)
# 计算两个字符串长度乘积
def _multiple_length_function(text1, text2):
    return len(text1) * len(text2)

def multiple_length_function(_dict):
    return _multiple_length_function(_dict["text1"], _dict["text2"])

# 提示词模板
prompt = ChatPromptTemplate.from_template("what is {a} + {b}")
# 模型
model = ChatOpenAI()
# 应该是个没用的东西
chain1 = prompt | model
# 定义chain
chain = {
    "a": itemgetter("foo") | RunnableLambda(length_function),
    "b": {"text1": itemgetter("foo"), "text2": itemgetter("bar")} | RunnableLambda(multiple_length_function)
} | prompt | model
# 调用
chain.invoke({"foo": "bar", "bar": "gah"})
```
输出是
```bash
AIMessage(content='3 + 9 equals 12.', additional_kwargs={}, example=False)
```
# 并行计算 Parallel
提供的接口是RunnableParallel/RunnableMap，可以允许多个runnable对象并行调用，并以map的形式返回数据。
例如：
```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.runnable import RunnableParallel

# 选择模型
model = ChatOpenAI()
# 定义第一个chain
joke_chain = ChatPromptTemplate.from_template("tell me a joke about {topic}") | model
# 定义第二个chain
poem_chain = ChatPromptTemplate.from_template("write a 2-line poem about {topic}") | model

# 定义并行chain
map_chain = RunnableParallel(joke=joke_chain, poem=poem_chain)
# 调用并行chain
map_chain.invoke({"topic": "bear"})
```
输出是：
```bash
    {'joke': AIMessage(content="Why don't bears wear shoes? \n\nBecause they have bear feet!", additional_kwargs={}, example=False),
     'poem': AIMessage(content="In woodland depths, bear prowls with might,\nSilent strength, nature's sovereign, day and night.", additional_kwargs={}, example=False)}
```
# 动态路由 Route
路由允许创建非确定性的链，即根据上一步的输出定义下一步。LCEL提供两种方法执行路由
1. 使用`.RunnableBranch`
2. 自定义函数

有点switch case的感觉
## RunnableBranch

```python
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatAnthropic
from langchain.schema.output_parser import StrOutputParser
# 创建第一个链，作用是对提问进行分类
chain = PromptTemplate.from_template("""Given the user question below, classify it as either being about `LangChain`, `Anthropic`, or `Other`.
                                     
Do not respond with more than one word.

<question>
{question}
</question>

Classification:""") | ChatAnthropic() | StrOutputParser()


# 创建子链，给模型不同的模板（要求）
langchain_chain = PromptTemplate.from_template("""You are an expert in langchain. \
Always answer questions starting with "As Harrison Chase told me". \
Respond to the following question:

Question: {question}
Answer:""") | ChatAnthropic()
anthropic_chain = PromptTemplate.from_template("""You are an expert in anthropic. \
Always answer questions starting with "As Dario Amodei told me". \
Respond to the following question:

Question: {question}
Answer:""") | ChatAnthropic()
general_chain = PromptTemplate.from_template("""Respond to the following question:

Question: {question}
Answer:""") | ChatAnthropic()

from langchain.schema.runnable import RunnableBranch
# 定义一个可运行的branch，用lambda表达式写条件判断
branch = RunnableBranch(
  (lambda x: "anthropic" in x["topic"].lower(), anthropic_chain),
  (lambda x: "langchain" in x["topic"].lower(), langchain_chain),
  general_chain
)

# 一个包含branch的链
full_chain = {
    "topic": chain,
    "question": lambda x: x["question"]
} | branch
full_chain.invoke({"question": "how do I use Anthropic?"})
full_chain.invoke({"question": "how do I use LangChain?"})
full_chain.invoke({"question": "whats 2 + 2"})
```
输出：
```bash
    AIMessage(content=" As Dario Amodei told me, ~", additional_kwargs={}, example=False)
    AIMessage(content=' As Harrison Chase told me, ~', additional_kwargs={}, example=False)
    AIMessage(content=' 2 + 2 = 4', additional_kwargs={}, example=False)
```
## 自定义路由函数

```python
# 路由函数
def route(info):
    if "anthropic" in info["topic"].lower():
        return anthropic_chain
    elif "langchain" in info["topic"].lower():
        return langchain_chain
    else:
        return general_chain
from langchain.schema.runnable import RunnableLambda

# 调用路由函数
full_chain = {
    "topic": chain,
    "question": lambda x: x["question"]
} | RunnableLambda(route)
```
会有同样的输出
---
title: LangChain Expression Language
date: 2023-10-16 15:27:33
tags:
---
LCEL
<!-- more -->
# LCEL是什么？有什么优点？
## 官方文档
先讲一下官方文档介绍的优点
1. 异步 批处理 流处理：设计、编码、小型测试的时候用同步接口，可以自动地切换到异步接口发布？
2. fallback：LLM结果不可控，发生错误时，可以回退到chain的上一个环节
3. 并行：LCEL默认组件都可以并行运行
4. 自动LangSmith（日志）：所有步骤都会自动打一些日志到langSmith中
## 直观感受

balabala这么一大通，最直观能感受到的优点是**简单**，简化了开发代码。
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
Runnable对象是LCEL 的核心组成部分，如上文所示，Langchain在此前定义了一系列的对象和函数，Langchain官方对这样的对象和函数统一定义了三个共同的接口，当对象和函数满足这三个接口时，便可被称之为 一个Runnable对象。
1. **stream**方法：调用Runnable时以**流式**返回输出结果（什么叫做流式返回？）
2. **invoke**方法：基于一个input调用Runnable
3. **batch**方法：基于一个list的input**批量**调用Runnable
上述方法都是同步的，前面加上a就是异步方法。
对于上述的三种方法，LCEL不同的Runnable对象支持接收不同的输入，并产生不同的输出，在Python中，能够使用“|”将前一个Runnable对象的输出传递到下一个Runnable对象作为输入，以此达到灵活自定义Prompt并实现“简洁地构建复杂LLM应用”的目的。
LCEL中定义的Runnable对象包括之前提到6大module，其输入输出跟原来介绍的无异。
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
```
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
```
EQUATION: x^3 + 7 = 12
```
# fallback
## 异常处理
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

## 回退
其实是进一步的异常处理，直接把序列给换了
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
在chain中调用的函数必须是单输入的，如果要用多输入的话，多写一个adaptor调整成单输入的
```python
from langchain.schema.runnable import RunnableLambda
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import ChatOpenAI
from operator import itemgetter

def length_function(text):
    return len(text)

def _multiple_length_function(text1, text2):
    return len(text1) * len(text2)

def multiple_length_function(_dict):
    return _multiple_length_function(_dict["text1"], _dict["text2"])

prompt = ChatPromptTemplate.from_template("what is {a} + {b}")
model = ChatOpenAI()

chain1 = prompt | model

chain = {
    "a": itemgetter("foo") | RunnableLambda(length_function),
    "b": {"text1": itemgetter("foo"), "text2": itemgetter("bar")} | RunnableLambda(multiple_length_function)
} | prompt | model
chain.invoke({"foo": "bar", "bar": "gah"})
```
输出是
```
AIMessage(content='3 + 9 equals 12.', additional_kwargs={}, example=False)
```
[toc]



# | 基础



## LLM

Large Language Model

- 流程：Prompt --> LLM --> Completion



## LangChain

- 解决 LLM 的挑战

  - 提示词管理、调整、模板（利于重用）
  - LLM request management: 调用不同的大模型
  - Output management: 解析
  - Multi LLM call management: 链式调用、多次交互

- 职责：类似 Java 开发中的 Spring 框架

  - **Prompts**

  - - Templatize, dynamically select, and manage model inputs

    - `prompt_template.format`

      ```
      template = "Tell me a {adjective} joke about {content}."
      
      prompt_template = PromptTemplate.from_template(template)
      prompt_template.input_variables
      # -> ['adjective', 'content']
      prompt_template.format(adjective="funny", content="chickens")
      # -> Tell me a funny joke about chickens.
      
      ```

      

  - **Language models**

  - - Make calls to language models through common interfaces

      ```
      from langchain.llms import OpenAI
      llm = OpenAI()
      # 切换LLM
      # llm = AzureOpenAI( deployment_name="td2", model_name="text-davinci-002")
      llm("Tell me a joke")
      ```

      

  - **Output parsers**

  - - Extract information from model outputs

    - `from langchain.output_parsers`

      ```
      from langchain.output_parsers import CommaSeparatedListOutputParser
      
      output_parser = CommaSeparatedListOutputParser()
      format_instructions = output_parser.get_format_instructions()
      prompt = PromptTemplate(
          template="List five {subject}.\n{format_instructions}",
          input_variables=["subject"],
          partial_variables={"format_instructions": format_instructions}
      )
      
      model = OpenAI(temperature=0)
      _input = prompt.format(subject="ice cream flavors")
      output = model(_input)
      output_parser.parse(output)
      ```

![Image](../img/ai/langchain-model.png)



- 更多功能

  - **Model I/O**

  - **Chains**

    > A single sequence of steps or actions that can be executed with predefined prompt template, output parser etc.
    >
    > 不同的chain预置好了不同的提示词，以解决不同的场景

    - LLM chain
    - RetrivelQA chain
    - Router chain
    - Math chain
    - Bash chain

  - **Data connection**

    ![Image](../img/ai/langchain-data-connection.png)

    - Source: DB, File, etc

  - **Agents**

    - Give a set of **tools** and let LLM to **choose the right tool** to answer input question, like how a human decide

    - > ReAct Mode: Reasoning <-> Action

    - https://www.pinecone.io/learn/series/langchain/langchain-agents/

    - 例子：ZeroShotAgent https://github.com/hwchase17/langchain/blob/master/langchain/agents/mrkl/prompt.py

      

  - **Memory** 

    - 记住对话上下文



# | 开发



API https://learn.microsoft.com/en-us/azure/cognitive-services/openai/reference

### Completion API

https://platform.openai.com/docs/api-reference/completions/create

openai.Completion.create

- 参数
  - `engine`: 用openai的哪一个引擎；
  - `prompt`: 提示语；
  - `max_token`: 调用生成的内容允许的最大 token 数量。你可以简单地把 token 理解成一个单词。注意它包含提示语长度！
  - `n`: 生成几条内容供选择；
  - `stop`: 希望模型输出的内容在遇到什么内容的时候就停下来。

```python
import openai
import os

openai.api_key = os.environ.get("OPENAI_API_KEY")

prompt = '请你用朋友的语气回复给到客户，并称他为“亲”，他的订单已经发货在路上了，预计在3天之内会送达，订单号2021AEDG，我们很抱歉因为天气的原因物流时间比原来长，感谢他选购我们的商品。'

def get_response(prompt, temperature = 1.0):
    completions = openai.Completion.create (
        engine="text-davinci-003",
        prompt=prompt,
        max_tokens=1024,
        n=1,
        stop=None,
        temperature=temperature,
    )
    message = completions.choices[0].text
    return message
    
```



### Chat completion API
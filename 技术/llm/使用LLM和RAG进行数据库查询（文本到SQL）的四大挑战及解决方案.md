#review/大模型
大型语言模型（LLM）的出现展示了机器理解自然语言的能力。这些能力帮助工程师完成了许多令人惊叹的工作，比如编写代码文档和代码审查，而最常见的用例之一是代码生成；GitHub Copilot展示了AI理解工程师代码生成意图的能力，例如Python、JavaScript和SQL。

## 使用LLM解决文本到SQL的问题

基于LLM的代码生成能力，许多人开始考虑使用LLM解决使用自然语言从数据库检索数据的长期难题，有时被称为“文本到SQL”。“文本到SQL”的概念并不新鲜；在“检索增强生成（RAG）”和最新的LLM模型突破之后，文本到SQL有了新的机会，利用LLM的理解力和RAG技术来理解内部数据和知识。

![图片](https://mmbiz.qpic.cn/mmbiz_png/USdKNuVuKuk3w2JKJq0eTiaWQxIiaEibHtwgrXYwbbiaHnzBRWN4wHgI8JQVqXhzJBPJrZibwnpicRe7gwHH95pnOXOg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过RAG架构进行文本到SQL

## 文本到SQL使用RAG的挑战

在文本到SQL的场景中，用户必须有精确度、安全性和稳定性才能信任LLM生成的结果。然而，追求一个可执行、准确、受控于安全性的文本到SQL解决方案并不那么简单；在这里，我们总结了使用LLM和RAG通过自然语言查询数据库的四个关键技术挑战：**上下文收集、检索、SQL生成和协作**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/USdKNuVuKuk3w2JKJq0eTiaWQxIiaEibHtwLiaBPliav27ABqOHfM0lsGlILVKLEkcsu22G2icCGySrdciaL2z1GbgaFQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用LLM和RAG进行文本到SQL的四个关键挑战

### 挑战1：上下文收集挑战

- **跨不同来源的互操作性**：为了无缝地概括和规范化跨不同来源、元数据服务和API搜索和集成的信息。
    
- **数据和元数据的复杂链接**：这涉及将数据与其元数据关联在文档存储中。它涉及存储元数据、模式和上下文，如关系、计算和聚合。
    

### 挑战2：检索挑战

- **向量存储的优化**：开发和实施向量存储的优化技术，如索引和分块，对于提高搜索效率和精度至关重要。
    
- **语义搜索的精确度**：挑战在于理解查询的上下文细微差别，这可以显著影响结果的准确性。这通常涉及查询重写、重新排名等技术。
    

### 挑战3：SQL生成挑战

- **SQL查询的准确性和可执行性**：生成既准确又可执行的SQL查询是一个重大挑战。这要求LLM深入理解SQL语法、数据库模式以及不同数据库系统特定方言。
    
- **适应查询引擎方言**：数据库通常在SQL实现中有独特的方言和细微差别。设计能够适应这些差异并生成跨各种系统兼容查询的LLM，为挑战增加了另一层复杂性。
    

### 挑战4：协作挑战

- **集体知识积累**：挑战在于创建一个机制，可以有效地收集、整合和利用来自多样化用户群的集体洞察力和反馈，以提高LLM检索的数据的准确性和相关性。
    
- **访问控制**：在我们最终检索数据的同时，下一个最重要的挑战是确保现有的组织数据访问策略和隐私法规也适用于新的LLM和RAG架构。
    

## 我们如何解决它？LLM的语义层

为了解决上述挑战，我们需要在LLM和数据源之间建立一个层，允许LLM学习数据源中的业务语义和元数据的上下文；这一层还需要将语义与物理数据结构映射，通常称为“语义层”。语义层必须解决语义和数据结构之间的连接，并协调访问控制和身份管理，确保只有合适的人访问合适的数据。

LLM的语义层应该包括什么？在这里，我们将其概括为几个方面。

### 数据解释和呈现

1. **业务术语和概念**：语义层包括业务术语和概念的定义。例如，“收入”一词在语义层中定义，因此当业务用户在他们的BI工具中查询“收入”时，系统确切知道要检索什么数据以及如何根据底层数据源计算它。
    
2. **数据关系**：它定义了不同数据实体之间的关系。例如，客户数据如何与销售数据相关，或者产品数据如何与库存数据链接。这些关系对于执行复杂分析和生成洞察至关重要。
    
3. **计算和聚合**：语义层通常包括预定义的计算和聚合规则。这意味着用户不需要知道如何编写复杂的公式来计算，例如，年初至今的销售；语义层根据其包含的定义和规则处理这些操作。
    

### 数据访问和安全

1. **安全和访问控制**：它还可以管理谁可以访问什么数据，确保用户只能查看和分析他们被授权访问的数据。这对于维护数据隐私和遵守法规至关重要。
    

### 数据结构和组织

1. **数据源映射**：语义层将业务术语和概念映射到实际数据源。这包括指定哪个数据库表和列对应于每个业务术语，允许BI工具检索正确的数据。
    
2. **多维模型**：在某些BI系统中，语义层包括多维模型（如OLAP立方体），允许进行复杂分析和数据切片/切块。这些模型将数据组织成用户可以轻松探索和分析的维度和度量。
    

### 元数据

1. **元数据管理**：它管理元数据，即关于数据的数据。这包括数据源、转换、数据血统以及任何其他有助于用户理解他们正在使用的数据的信息。
    

## 介绍WrenAI

![图片](https://mmbiz.qpic.cn/mmbiz_png/USdKNuVuKuk3w2JKJq0eTiaWQxIiaEibHtwT1vDMtp6JVOic8q8K83NpG8PehBOJaP3fN2gvCOfKAwZdCQ6Rv88OvA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

WrenAI是您与数据的自然语言接口

> WrenAI是您与数据的自然语言接口

WrenAI是开源的。您可以在您的数据、LLM API和环境中的任何地方部署WrenAI。它带有直观的入门和用户界面，允许您在几分钟内连接和建模数据源中的数据模型。

在WrenAI的底层，**开发了一个名为“Wren Engine”的框架——LLM的语义层**。Wren Engine也在GitHub上开源。

### WrenAI中的建模

在连接到您的数据源后，它将自动收集所有元数据，并通过WrenAI UI，您可以通过用户界面添加业务语义和关系；它将自动更新您的向量存储，以便将来进行语义搜索。

![图片](https://mmbiz.qpic.cn/mmbiz_png/USdKNuVuKuk3w2JKJq0eTiaWQxIiaEibHtwLsCSxCv886meLMlib4bdOKJYDJ0wVkRAzQ7I08OYdfbg4IbMspQMFrQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

WrenAI中的数据建模

### 询问和后续问题

建模完成后，您可以开始询问您的业务问题；WrenAI将搜索检索最相关的3个结果供您选择。一旦您选择了其中一个选项，它将分解为数据来源的逐步解释，并提供摘要，以便您可以更有信心地使用WrenAI建议的结果。

从WrenAI获得结果后，您可以根据返回的结果询问后续问题，以获得更深入的洞察力或分析。

![图片](https://mmbiz.qpic.cn/mmbiz_png/USdKNuVuKuk3w2JKJq0eTiaWQxIiaEibHtwnKwtiaW3OBQ3cJT5WGWhkibGPqLBiaSnGPFNXJRfTV6Im3n4la1cOia4uA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在WrenAI中询问和后续问题

---

### 今天在GitHub上尝试WrenAI！

👉  GitHub: https://github.com/Canner/WrenAI

# Reference
https://mp.weixin.qq.com/s/W_t1Km7laxAqkeQi-DBX3w
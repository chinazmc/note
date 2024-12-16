#review/大模型 

## 什么是 Genkit

[Genkit](https://github.com/firebase/genkit) 是一个 Google Firebase 团队开发的 AI Agent 开发框架，用于构建现代、高效的 AI 应用。它目前包含一个 [Node.js 的实现](https://firebase.google.com/docs/genkit?hl=zh-cn) 和一个 [Go 语言的实现](https://firebase.google.com/docs/genkit-go/get-started-go)。之所以注意到这个框架是因为 Go 团队在他们的[十五周年博客](https://go.dev/blog/15years)中提到了它。Go 团队在博客中提到，他们正在努力使 Go 成为构建生产 AI 系统的优秀语言，并且他们正在为流行的 Go 语言框架提供支持，包括 [LangChainGo](https://github.com/tmc/langchaingo) 和 [Genkit](https://developers.googleblog.com/en/introducing-genkit-for-go-build-scalable-ai-powered-apps-in-go/)。如果你了解过 AI 开发，那么 [LangChain](https://www.langchain.com/) 一定并不陌生。LangChainGo 就是 LangChain 的 Go 语言实现。但是对应的，我们还需要一个单独的 AI Agent 开发框架，帮助我们组织 AI Agent 的开发。LangChain 的生态位中，对应的是 [LangGraph](https://www.langchain.com/langgraph)，在 Go 语言的生态位中，你可以选择 Genkit 或者 [LangGraphGo](https://github.com/tmc/langgraphgo)。

但是，值得注意的是，目前 Genkit 仍旧是在 Alpha 的状态，所以，**如果你希望将 Genkit 用于生产环境，在 2024 年这个时间点请注意未来可能的 API 变动。** 目前 Genkit 官方支持的模型功能仍旧是以 Google 家的产品为主，如果你需要使用其他 AI 模型，你可能需要暂时先使用第三方 plugin（比如 OpenAI）。

## 环境需求

Genkit 提供了一套 CLI 工具协助快速开发，这套工具使用 Node.js 开发，所以，你需要安装 Node.js 环境。在这个例子中，我使用了下面的环境，在开始之前，你可以参考这个环境准备你的开发环境：

- Node.js: **v22.11.0** （最低应该 v20+）
- Go: **1.23.3** (最低应该 1.22+)

## Genkit Demo 项目演示

我们可以使用 Genkit 提供的 CLI 工具快速创建一个项目，由于最新的 Genkit CLI 工具[仍然有一些疑问](https://github.com/firebase/genkit/issues/1295)，这里我锁定在了一个老版本上进行执行：

```bash
> mkdir genkit-hello-world && cd genkit-hello-world && npx genkit@0.5.17 init
```

Copy

接下来选择语言模版：

```bash
? Select a runtime to initialize a Genkit project:
  Node.js
❯ Go (preview)
? Select a runtime to initialize a Genkit project: Go (preview)
? Select a model provider: (Use arrow keys)
❯ Google AI
  Google Cloud Vertex AI
  Ollama (e.g. Gemma)
  None
? Select a model provider: Google AI
? Enter the Go module name (e.g. github.com/user/genkit-go-app): <输入你的模块名，这里我使用了genkit-hello-world>
? Enter the Go module name (e.g. github.com/user/genkit-go-app): genkit-hello-world
✔ Successfully installed Go packages
? Would you like to generate a sample flow? (Y/n) y
? Would you like to generate a sample flow? Yes
✔ Successfully generated sample file (main.go)
Warn: Google AI is currently available in limited regions. For a complete list, see https://ai.google.dev/available_regions#available_regions
Genkit successfully initialized.
```

Copy

这里我们选择了 Google AI 驱动的 Go 语言模版，并且生成了一个示例的 flow。我们接下来就可以直接使用 `npx genkit-cli@latest start` 来启动开发服务器。

```bash
❯ npx genkit@0.5.17 start
Starting app at `.`...
Genkit Tools API: http://localhost:4000/api
time=2024-11-14T09:18:25.771+08:00 level=INFO msg="googleai.Init: Google AI requires setting GOOGLE_GENAI_API_KEY or GOOGLE_API_KEY in the environment. You can get an API key at https://ai.google.dev"
exit status 1
App process exited with code 1, signal null
```

Copy

这里指的注意的是我们需要设置 Google AI 的 API Key，我们可以通过 `export GOOGLE_GENAI_API_KEY=<your-api-key>` 来设置。如果不设置，则会出现上面的错误。这个 API Key 可以通过 [Google AI](https://ai.google.dev/) 获取。由于目前 Gemini 提供了每天的免费额度，所以我们日常的开发和测试可以在免费额度内完成。不过需要注意的是，主流的国际 AI 服务都限制了中国地区的访问，请根据你的情况选择可以访问的方式。

```bash
❯ export GOOGLE_GENAI_API_KEY=<your-api-key>
❯ npx genkit-cli@latest start
Starting app at `.`...
Genkit Tools API: http://localhost:4000/api
time=2024-11-14T09:22:58.907+08:00 level=INFO msg=RegisterAction type=model name=googleai/gemini-1.5-flash
time=2024-11-14T09:22:58.907+08:00 level=INFO msg=RegisterAction type=model name=googleai/gemini-1.0-pro
time=2024-11-14T09:22:58.907+08:00 level=INFO msg=RegisterAction type=model name=googleai/gemini-1.5-pro
time=2024-11-14T09:22:58.907+08:00 level=INFO msg=RegisterAction type=embedder name=googleai/text-embedding-004
time=2024-11-14T09:22:58.907+08:00 level=INFO msg=RegisterAction type=embedder name=googleai/embedding-001
time=2024-11-14T09:22:58.907+08:00 level=INFO msg=RegisterAction type=flow name=menuSuggestionFlow
time=2024-11-14T09:22:58.907+08:00 level=INFO msg="starting reflection server"
time=2024-11-14T09:22:58.908+08:00 level=INFO msg="starting flow server"
time=2024-11-14T09:22:58.908+08:00 level=INFO msg="all servers started successfully"
time=2024-11-14T09:22:58.908+08:00 level=INFO msg="server listening" addr=127.0.0.1:3100
time=2024-11-14T09:22:58.908+08:00 level=INFO msg="server listening" addr=127.0.0.1:3400
time=2024-11-14T09:22:58.933+08:00 level=INFO msg="request start" reqID=1 method=GET path=/api/__health
time=2024-11-14T09:22:58.933+08:00 level=INFO msg="request end" reqID=1
Genkit Tools UI: http://localhost:4000
```

Copy

通过上面的输出，我们可以看到 Genkit 的开发服务器已经启动，并且我们可以通过 `http://localhost:4000` 访问 Genkit 的 UI 界面。这是 Genkit 提供的工具界面，通过这个界面，我们可以方便的调试和测试我们的 AI Agent。

[![Genkit UI](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-ui.png)](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-ui.png)

Genkit UI

这个默认的项目模版中定义了一个默认的 flow，名叫 `menuSuggestionFlow`。我们可以选择这个 flow 来，来进行测试。比如如下图所示，我们输入 `apple`，让 AI 帮我推荐一个菜品：

[![Genkit Debug Flow](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-debug-flow.png)](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-debug-flow.png)

Genkit Debug Flow

注意一下，在这个页面的最下方包含了一个 `View Trace` 的按钮，这个按钮可以帮助我们查看 AI Agent 的执行 trace，这对于我们调试和优化 AI Agent 非常重要。这个追踪借助了 [OpenTelemetry](https://opentelemetry.io/) 的实现，也非常方便在你的代码中进行扩展，方便追踪 Agent 的执行过程。

[![Genkit Trace](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-trace.png)](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-trace.png)

Genkit Trace

接下来，我们看一下代码是如何定义整个 AI Agent 的。

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"

    // 导入 Genkit 核心库
    "github.com/firebase/genkit/go/ai"
    "github.com/firebase/genkit/go/genkit"

    // 导入 Google AI 插件
    "github.com/firebase/genkit/go/plugins/googleai"
)

func main() {
    ctx := context.Background()

    // 初始化 Google AI 插件。留空 apiKey 参数时，插件会从推荐使用的 GOOGLE_GENAI_API_KEY 环境变量读取值。
    if err := googleai.Init(ctx, nil); err != nil {
        log.Fatal(err)
    }

    // 定义一个简单流程，提示大型语言模型 (LLM) 生成菜单建议。
    genkit.DefineFlow("menuSuggestionFlow", func(ctx context.Context, input string) (string, error) {
        // Google AI API 提供访问多个生成模型的功能。这里我们指定 gemini-1.5-flash。
        m := googleai.Model("gemini-1.5-flash")
        if m == nil {
            return "", errors.New("menuSuggestionFlow: failed to find model")
        }

        // 构建请求并发送到模型 API。
        resp, err := m.Generate(ctx,
            ai.NewGenerateRequest(
                &ai.GenerationCommonConfig{Temperature: 1},
                ai.NewUserTextMessage(fmt.Sprintf("Suggest an item for the menu of a %s themed restaurant", input))),
            nil)
        if err != nil {
            return "", err
        }

        // 处理来自模型 API 的响应。在这个例子中，我们只将其转换为字符串，但更复杂的流程可能会将响应转换为结构化输出或将响应链接到另一个 LLM 调用等。
        text := resp.Text()
        return text, nil
    })

    // 初始化 Genkit 并启动流程服务器。此调用必须放在最后，在所有插件配置和流程定义之后。将 nil 配置项传递给 Init 时，Genkit 将启动本地流程服务器，您可以使用开发者界面进行交互。
    if err := genkit.Init(ctx, nil); err != nil {
        log.Fatal(err)
    }
}
```

Copy

这其中比较重要的流程在 `genkit.DefineFlow` 中定义。这个定义了流程名和对应的实现方式。在这个例子中，我们定义了一个名为 `menuSuggestionFlow` 的流程，这个流程接收一个字符串输入，然后返回一个字符串输出。在最开始我们初始化了一个 Google AI 的模型，然后通过这个模型来生成对应的输出。当然，正如代码注释中提到的，正常的 AI Agent 要远比这个 Hello World 级别的程序更加复杂。不过没关系，接下来，我们就尝试把这个应用修改一下，实现一个基础的，你自己的 ChatPDF 工具。

## 实现 ChatPDF 工具

如何实现一个自己的 ChatPDF 工具？思路当然很简单，我们需要一个流程，这个流程接收一个 PDF 文件然后进行保存。同时，我们还需要另外一个流程，接收一个用户的问题，查询保存的数据，然后返回一个答案。在第一个过程中，我们需要解析这个 PDF 文件，然后根据这个 PDF 文件的内容来回答用户的问题。在这个过程中我们会需要用到一些 AI 的能力，比如文本的摘要，文本的问答，文本的总结等等。

所以首先，我们需要定义一个提取 PDF 文件内容的方法，这个功能需要使用到 [`github.com/ledongthuc/pdf`](https://github.com/ledongthuc/pdf) 这个库，我们可以偷懒直接复制它的示例代码：

```go
func readPdf(path string) (string, error) {
	f, r, err := pdf.Open(path)
	// remember close file
	defer f.Close()
	if err != nil {
		return "", err
	}
	var buf bytes.Buffer
	b, err := r.GetPlainText()
	if err != nil {
		return "", err
	}
	buf.ReadFrom(b)
	return buf.String(), nil
}
```

Copy

由于 AI 上下文长度的限制，我们很难把整个 PDF 文件的内容都交给 AI 来作文上下文（不过虽然我们在用的 Gemini 的 API 上下文长度确实足以满足这个需求，但是我们现在讨论的是一种更通用的解决方案），所以我们需要对 PDF 文件拆分进行向量化，仅把需要的文本内容交给 AI 来作文上下文。这里解析到文本内容后，我们还需要使用 LangChainGo 中的 [github.com/tmc/langchaingo/textsplitter](https://pkg.go.dev/github.com/tmc/langchaingo/textsplitter) 这个库来对文本内容进行裁剪，确保文本长度不会超过 AI 的上下文限制。

```go
splitter := textsplitter.NewRecursiveCharacter(
	textsplitter.WithChunkSize(200),
	textsplitter.WithChunkOverlap(20),
)
```

Copy

内容基本完成，现在我们可以定义一个单独的工作流，这个工作流接受一个 PDF 文件路径，并且拆分解析这个 PDF 中的文本，并将其向量化后保存，以供后续使用。为了方便，我们在这里使用一个调试用 VectorDB，这个 DB 是基于本地文件使用的 [github.com/firebase/genkit/go/plugins/localvec](https://pkg.go.dev/github.com/firebase/genkit/go/plugins/localvec)。请注意，我这里使用的 LangchainGo 库的版本为 `v0.1.13-pre.0`，可能会和你的使用 API 有一些出入。

```go
	if err := localvec.Init(); err != nil {
		log.Fatal(err)
	}

	pdfIndexer, _, err := localvec.DefineIndexerAndRetriever(
		"pdfIndexer",
		localvec.Config{
			Embedder: googleai.Embedder("text-embedding-004"),
		},
	)
	if err != nil {
		log.Fatal(err)
	}

	splitter := textsplitter.NewRecursiveCharacter(
		textsplitter.WithChunkSize(200),
		textsplitter.WithChunkOverlap(20),
	)

	genkit.DefineFlow("indexPDF", func(ctx context.Context, input string) (string, error) {
		pdfText, err := genkit.Run(ctx, "readPdf", func() (string, error) {
			return readPdf(input)
		})
		if err != nil {
			return "", err
		}

		chunks, err := genkit.Run(ctx, "chunk", func() ([]*ai.Document, error) {
			chunks, err := splitter.SplitText(pdfText)
			if err != nil {
				return nil, err
			}

			var docs []*ai.Document
			for _, chunk := range chunks {
				docs = append(docs, ai.DocumentFromText(chunk, nil))
			}
			return docs, nil
		})
		if err != nil {
			return "", err
		}

		err = ai.Index(ctx, pdfIndexer, ai.WithIndexerDocs(chunks...))
		return "", err
	})
```

Copy

如果这个时候你还在运行 Genkit 的开发服务器，那么它会自动尝试构建这段代码，这时候我们回到 Genkit 的 UI 界面，选择我们刚刚定义的 `indexPDF` 工作流，然后输入一个 PDF 文件路径，点击 `Run` 按钮，就可以看到这个工作流的执行结果。注意上面的代码中，`genkit.Run` 方法的第一个参数是 `ctx`，这个参数是 Genkit 提供的上下文，可以用于追踪和日志记录。为了方便检查，我们也可以通过 `View Trace` 查看一下我们当前的执行调用情况：

[![Genkit Trace](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-trace-index-pdf.png)](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-trace-index-pdf.png)

Genkit Trace

这里我使用了 Nuki 这家公司的介绍故事 PDF 作为测试文件，你可以根据你的需要修改这个文件路径。

在完成文件处理流程之后，我们就需要实现一个基础的，面向问答的 Flow。毕竟，前面的文件解析流程对普通用户来说只是前置步骤，我们最终的目的还是希望用户可以上传一个 PDF 文件，然后可以向这个文件提问，并得到答案。

接下来就是实现一个 `chatPDF` 的 Flow，这个 Flow 接收一个用户的问题，然后根据这个问题的内容，从向量数据库中检索出相关的文本内容，然后交给 AI 来作文回答。这部分也就是我们熟悉的 [RAG](https://en.wikipedia.org/wiki/Retrieval-augmented_generation) 回答环节了。

```go

	// 我们要额外修改一个地方，确保我们能够从向量数据库中检索出相关的文本内容。
	pdfIndexer, pdfRetriever, err := localvec.DefineIndexerAndRetriever(
		"pdfIndexer",
		localvec.Config{
			Embedder: googleai.Embedder("text-embedding-004"),
		},
	)
	if err != nil {
		log.Fatal(err)
	}

	// ...保持其他代码不变...

	genkit.DefineFlow("qaPDF", func(ctx context.Context, question string) (string, error) {
		model := googleai.Model("gemini-1.5-flash")

		// 从向量数据库中检索出相关的文本内容
		docs, err := genkit.Run(ctx, "retrieve", func() (*ai.RetrieverResponse, error) {
			return pdfRetriever.Retrieve(ctx, &ai.RetrieverRequest{
				Document: ai.DocumentFromText(question, nil),
			})
		})
		if err != nil {
			return "", err
		}

		// 在此之前我们声明嵌入信息：
		embededInfo := ai.NewSystemTextMessage("Here is the context:")
		for _, doc := range docs.Documents {
			embededInfo.Content = append(embededInfo.Content, doc.Content...)
		}

		// 现在可以让 AI 回答问题了
		resp, err := ai.GenerateText(ctx, model, ai.WithMessages(
			ai.NewSystemTextMessage(`You are acting as a helpful AI assistant that can answer questions about the provided context.
Use only the context provided to answer the question. If you don't know, do not make up an answer. Do not add or change anything in the context.`),
			embededInfo,
			ai.NewUserTextMessage(question),
		))
		return resp, err
	})
```

Copy

现在我们已经定义了一个完整的工具流程，刚刚的工作流用于处理文件内容，这个工作流用于检索这些相关信息并进行回答。因为我使用的文件是 Nuki 公司的介绍文件，那么我的问题自然也和这家公司相关：

[![Genkit QA PDF](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-qa-pdf.png)](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-qa-pdf.png)

Genkit QA PDF

为了方便调试，我们还可以点击 `View Trace` 按钮，查看当前的执行调用情况，比如说我们从向量数据库中到底取出了什么样的数据，这会帮助我们决策是否需要优化 RAG 的实现：

[![Genkit Trace](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-trace-qa-pdf.png)](https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/genkit-trace-qa-pdf.png)

Genkit Trace

## 其他

在完成这个 Demo 之后，我们应该已经初步了解了 Genkit 的开发流程，并且可以基于这个框架开发出自己的 AI Agent 应用。虽然这个工具目前仍旧是在早期阶段，但应对基础的 Agent 开发已经基本足够。当然，从另外一方面说，这个框架仍旧需要生态进一步完善，比如以 RAG 为例，我们刚刚使用的仅仅是最基础的 RAG 功能， 为了提升 RAG 的成功率，业内还有很多其他的实现方式提升搜索结果准确率，比如 GraphRAG 等等，这部分功能暂时没有开箱即用的方式，仍然需要进一步提升生态。

不过今天的文章就先到这里结束吧。
# Reference
https://www.4async.com/2024/11/building-ai-agent-with-genkit-for-go/
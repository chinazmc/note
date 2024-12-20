Go vs Rust: Which To Learn In 2024?  
Subtitle Machine Translation  
Go versus Rust : Which is the better language to learn between Go and Rust? 
Go 与 Rust： Go 和 Rust 哪种语言更值得学习？  
Which offers the best performance? 
哪个提供最好的性能？  
Which offers the best opportunities? Which should you learn? 哪个提供了最好的机会？ 你应该学哪一个？  
We’ll dive into all that and more. 我们将深入探讨所有这些以及更多内容。  
First, let me explain how we’ll be approaching this. 首先，让我解释一下 我们将如何解决这个问题。  
First, we’ll look at the differences in **philosophy and mindset** between each language. 
首先，我们将了解  每种语言之间**哲学和思维方式**的差异。 

Second, we’ll explore the **features**  of both languages and their relative strengths and weaknesses 其次，我们将探讨  两种语言的**功能**以及它们的 相对优势和劣势。  

Third, we’ll examine what the **statistics** say about the difference between the two languages in terms of developer experience, attractiveness and salary.
第三，我们将研究 **统计数据**说明  两种语言在 开发者体验、吸引力和薪水方面的差异。

And then I’ll tell you what I recommend. But whatever language you choose,
然后我会告诉你我的建议。 但无论您选择哪种语言，  
I’ve provided links in the description to helpful resources for learning them. 
我都在说明中提供了 有助于学习这些语言的有用资源的链接。  
Let’s dive in 让我们深入探讨  
What are the differences in philosophy between the two languages?
两种语言之间的哲学有何差异？  
We have our contenders. In the blue corner, Go’s friendly gopher. In the.. well, brownish-red corner, Rust’s unofficial mascot, Ferris the crab. And there you have it already. These logos, names,  mascots and even colours tell us a lot about the differences between the two languages. 
我们有我们的竞争者。 在蓝色角落里，是 Go 的友好地鼠。 在......嗯，棕红色的  角落里，Rust 的非官方吉祥物，螃蟹。 现在你已经有了它。 这些标志、名称、  吉祥物甚至颜色告诉我们很多关于 两种语言之间的差异。  
Let’s start with Go. Go’s name and logo express speed and efficiency. 
让我们从 Go 开始。 Go 的名称和 徽标表达了速度和效率。  
The three lines make the text look like two wheels in rapid motion. 
这三行使文字看起来 像两个快速移动的轮子。  
Go’s mission, as expressed by their brand book, is to: * Bring order to the complexity * of creating * and running software at scale. 
正如  其品牌手册所表达的那样，Go 的使命是： * 为  大规模创建 * 和运行软件的复杂性带来秩序。  
Let’s unpack that. _Creating_ and _running_ at scale. What does that mean? 
让我们来解开它。 大规模_创建_和 _运行_。 这意味着什么？  
Go’s goal is to be : * 1/ simple enough to be used for prototyping and * 2/ efficient enough to be “The Language of the Cloud”. 
Go 的目标是： * 1/ 足够简单，可用于原型设计；  * 2/ 足够高效，成为 “云语言”。  
Go’s [brand book] also says it wants to be Thoughtful, Simple,Efficient, Reliable, Productive, and Friendly.  
Go 的[品牌手册]还表示它 希望变得深思熟虑、简单、  高效、可靠、富有成效和友好。  
These values are expressed in the friendly, cuddly mascot, the Gopher. 
这些价值观在 友好、可爱的吉祥物地鼠身上得到了体现。  
Rust’s mascot, Ferris, certainly has a friendly smile. 
Rust 的吉祥物 Ferris 当然有着友好的微笑。  
It’s prickly, though. As is Rust’s logo. 
不过，这很刺痛。 Rust 的徽标也是如此。  
Rust’s goal is _not_ to be friendly or simple. After all… Crabs walk sideways. 
Rust 的目标_不_是友好或简单。 毕竟……螃蟹是侧着走的。  
The best way to explain Rust’s values is to tell its [origin story] .
解释 Rust 价值观的最佳方式 是讲述它的[起源故事]。  
Graydon Hoare was a programmer at Mozilla. Coming home one night, 
Graydon Hoare 是 Mozilla 的一名程序员。 一天晚上回到家，  
he found an out-of-service elevator. The software had crashed due to a memory error. 
他发现一部停止运行的电梯。 由于内存错误，该软件崩溃了。  
This goaded Hoare into creating a language that prevents memory management crashes.
这促使霍尔创建一种可以 防止内存管理崩溃的语言。  
Rust is designed to be **solid**, like a crab. And the name “Rust” also conveys this. 
Rust 被设计为**坚固**，就像螃蟹一样。 “Rust”这个名字也传达了这一点。  
It doesn’t refer to the oxidation of iron but to an exceedingly resilient [fungi]. 
它不是指铁的氧化， 而是指一种极有弹性的[真菌]。  

 A closer look at the features What are the differences and common strengths between Go and Rust’s features? 
  仔细查看功能   Go 和 Rust 的功能之间有哪些差异和共同优势？  
Go and Rust Both focus on: * memory management,* concurrency and * performance.
Go 和 Rust 都专注于： * 内存管理、   * 并发性和 * 性能。  
However, their approach is _very_ different. 
然而，他们的方法却_非常_不同。  
This, too, reflects their philosophy. Allow me to demonstrate.
这也反映了他们的哲学。 请允许我演示一下。  
First, let’s talk about memory management. Go has a helpful **garbage collector** that tries to clean up unused memory references for you. 
首先，我们来谈谈内存管理。 Go 有一个有用的**垃圾收集器**，它会尝试为  您清理未使用的内存引用。 
Rust, on the other hand, enforces the  concept of ownership and borrowing to prevent you from coding memory bugs. 
另一方面，Rust 强制执行  所有权和借用的概念，以 防止您编写内存错误。  
Go helps you. Rust forces you to do the right thing. 
Go 帮助你。 Rust 迫使 你做正确的事。  
Second, concurrency and multithreading. 
第二，并发和多线程。  
In a similar fashion, Go provides easy-to-use Goroutines. 
以类似的方式，Go 提供了易于使用的 Goroutine。  
These are lightweight threads that allow you to write concurrent code without worrying about thread management. Rust, on the other hand, uses system threads. 
这些是轻量级线程， 允许您编写并发代码，而  无需担心线程管理。 另一方面，Rust 使用系统线程。  
Here, Rust’s ownership concept helps prevent thread-locking errors and data races. 
在这里，Rust 的所有权概念有助于防止 线程锁定错误和数据竞争。  
Again, Go makes your life easier. Rust teaches you to do the right thing. 
再说一次，Go 让你的生活更轻松。 Rust 教你做正确的事。  
Third, performance. Go has a small memory footprint and is optimised for modern multi-core processors. Its syntax is simple and concise.
第三，性能。 Go 的内存占用很小，并且   针对现代多核处理器进行了优化。 它的语法简单、简洁。  
This makes it easy to learn and start working with. 
这使得学习 和开始工作变得容易。  
Go does the right thing for you by default. 
默认情况下，Go 会为你做正确的事情。  
Rust provides something called Zero-Cost Abstractions. 
Rust 提供了一种 称为零成本抽象的东西。  
There is no “magic”. 
不存在“魔法”。  
The code you write faithfully reflects the code that the compiler produces. 
您编写的代码忠实地反映了 编译器生成的代码。  
This means less runtime overhead, which helps you build high-performance applications. 
这意味着更少的运行时开销，有助于 您构建高性能应用程序。  
In short, Go hides the complexity for you. Rust helps you understand it by exposing it.
简而言之，Go 为您隐藏了复杂性。 Rust 通过暴露它来帮助你理解它。  
Now… what do developers think of the languages? Let’s look at the statistics provided by the Stack Overflow Developer survey. Stack Overflow   
Go and Rust have similar popularity levels, at about 13% overall among all developers.
现在……开发人员 对这些语言有何看法？ 让我们看一下   Stack Overflow 开发者调查提供的统计数据。Go 和 Rust 的受欢迎程度相似，在 所有开发者中的总体受欢迎程度约为 13%。 然而，  
Developer _opinions_ of the languages, however, show a difference. 开发者对这些语言的看法却 存在差异。  
84% of the developers who used Rust last year want to use it again this year. 
去年使用 Rust 的开发者中有 84% 希望今年再次使用它。  
For Go, that number is lower at 60%. The survey also shows 30% of those who did not work with Rust last year want to work with it. That number is 20% for Go. 
对于 Go 来说，这个数字较低，为 60%。 该调查还显示，去年未使用 Rust 的人中有 30%  想要使用它。 对于 Go 来说这个数字是 20%。  
In short, Rust is more desirable among developers than Go.
简而言之，Rust 比 Go 更受开发者欢迎。  
But what about on the job market? 
但就业市场上又如何呢？  
Finally, if we look at the reported salaries, developers working in both languages report a 
mean yearly salary of 90 thousand dollars. Go developers are slightly above,
最后，如果我们查看报告的薪资，使用 两种语言工作的开发人员报告的  平均年薪为 9 万美元。 Go 开发人员略高于，  
and Rust developers are slightly below. 
Rust 开发人员略低于。  
A word of caution here: these values are reported globally. 
请注意：这些 值是在全球范围内报告的。  
The situation in your local job market will vary. 
您当地就业市场的情况会有所不同。  
After having explored the philosophies,features and statistics of Go and Rust, what are my recommendations?  
在探索了  Go 和 Rust 的哲学、功能和统计数据之后 ，我有什么建议？  
Let’s recap. 
让我们回顾一下。  
Go is designed to be simple and friendly. Rust is designed to be as hard as possible to break.
Go 的设计简单且友好。 Rust 被设计为尽可能难以损坏。  
They’re two teachers. One is encouraging and warm, a kindergarten teacher. The other is demanding and harsh, a drill sergeant. 
他们是两位老师。 一个是鼓励和温暖的，  一位幼儿园老师。 另一个 要求严格、严厉，是一名教官。  
One makes life easier for you. The other makes you work harder for your own good. 
一个让您的生活更轻松。 另一个 让你为了自己的利益而更加努力地工作。  
Go and Rust also have different primary use cases. Go is meant for the cloud, for server applications. Rust is meant for low-level or embedded applications. 
Go 和 Rust 也有不同的主要 用例。 Go 是为云服务器应用程序设计的。 Rust 适用 于低级或嵌入式应用程序。  
Which is the best for you? Well, that will depend on two things: 
哪个最适合您？ 嗯，这取决于两件事：  
who you are and what you are trying to build. Go is easier to pick up. Rust requires more effort 
您是谁以及您想要构建什么。 Go 更容易上手。 Rust 需要更多的努力，  
but provides greater rewards in the long run. What kind of teacher do you need? Do you need encouragement? Or are you building for the cloud? I recommend Go. 
但从长远来看会带来更大的回报。 你需要什么样的老师？ 你需要  鼓励吗？ 或者您正在 为云进行构建？ 我推荐去。  
Do you need a demanding teacher who will push you forward? Do you want to build performance-intensive or low-level embedded software? I recommend Rust. 
您需要一位要求严格的老师来 推动您前进吗？ 您想要构建  性能密集型或低级 嵌入式软件吗？ 我推荐铁锈。  
Whatever language you choose, if you’re looking to pick up the basics, I recommend you go to 
[Exercism] which gives you simple challenges to help you learn. And if you’re looking for projects to help you dive deep while learning how reference technologies like Git, Docker or HTTP servers work, I recommend [CodeCrafters]. I’ve provided links for both in the description. 
无论您选择哪种语言，如果您想 学习基础知识，我建议您访问  [Exercism]，它为您提供了简单的挑战 来帮助您学习。 如果您正在寻找  项目来帮助您深入了解 Git、Docker 或  HTTP 服务器等参考技术的工作原理，我推荐 [CodeCrafters]。 我在描述中提供了两者的链接。  
And whatever path you choose, I’ll see you in the next video. 
无论您选择哪条路， 我们都会在下一个视频中见到您。
在计算机上，一切都只是比特和字节，可以自由复制。你需要一种语言，为你提供所有权和稀缺性方面的必要抽象，就像在现实世界中一样。你需要这些基本的安全保证。这就是 Move 所做的，也是我们创建新语言的原因。这些东西很难在其他语言（包括现有的智能合约语言）中重现，我们希望围绕提供这些基元来设计整个语言，这样程序员就可以安全高效地编写代码，而不必在每次要编写代码时都重新发明轮子。

## 

 Sam Blackshear

Move 编程语言的创作者，Mysten Labs 的联合创始人，Sui 的初始贡献者

### 

 Move 语言简介

Move 语言 (Move[-on-Diem]) 诞生于 Meta 公司，它是一种为区块链和智能合约设计的编程语言，最初用于支持 Libra（后改名为 Diem）项目。Move 的设计和实现参考了 Rust 语言，具有丰富的基础类型和数组。

正如 Move 语言的作者 Sam Blackshear 而言，Move 语言解决了以下的问题：

●所有权和稀缺性：Move 语言为计算机中的比特和字节提供了必要的抽象，使得它们可以像现实世界中的物品一样拥有所有权和稀缺性。

●安全保证：Move 设计了基础的安全保障，使得程序员可以在不重新发明轮子的情况下，安全高效地编写代码。

●资源类型系统：通过资源类型系统确保资源不能被复制或丢弃，只能通过显式转移来管理。

### 

 Move[-on-Sui] 是什么

Move[-on-Sui]（Sui）是由 Mysten Labs 为 Sui 区块链开发的智能合约语言，它是对 Move 语言的改进版本，专门为与 Sui 的以对象为中心的数据模型集成而设计。

下图展示了 Move 语言的演化过程，Sui 是基于 Move 语言设计的扩展和优化，进一步提升了 Move 语言的性能和适用性。对象数据模型提高了安全性并实现了扩展，可以创建丰富的对象层次结构，从而带来无与伦比的创造性和开发体验。

![image](https://hackquest-s3-prod-apne1.s3.ap-northeast-1.amazonaws.com/courses/b7ac8a16-c482-449b-8f06-86fadefc569b/5b263322-5194-4715-9610-f22dff91b6e2/077c0ab3-c07d-4913-a338-32f2c8cb1913/bf7hNn5n0gJR04KJq6R_1.webp?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAYCTGVDAPHARJARG5%2F20240803%2Fap-northeast-1%2Fs3%2Faws4_request&X-Amz-Date=20240803T053426Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjELT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaDmFwLW5vcnRoZWFzdC0xIkYwRAIgUXvvFeU4%2BrcS%2BHUZ7%2BHP8vVSo4dGHPfRGxP4LC4d9xkCIHjT3ukFnv2DdnmYv%2BJqZ1GMP81qyex6Yiw8NQYlKD3qKqcECJ3%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEQARoMNTU1MzM5ODE0OTQyIgyHqjjOGrR1eauAbJAq%2BwPKX0hggH3UePfdyYbnpVm%2BdC0itvf7tXc2Rfi1hZCJti6zr5zxukXkDOobNC0veJIaKlz%2F1VtQt7xQvncJhbERLH%2BZUbXO56UE%2FAmsAuv7Ek2FV8bcTbK1DsjLY%2Bi8b%2BhcPFYCYuN1n7wsM6MtGndbS2R%2Fj8YKxNMyWL9gA3R%2BD0j62VRKaxEEe73Ofr1B5B7CqyXKpxFcQuFfaT4xhyg5Lzez4ZwHcvHQLCoInZ60Rhotm2n1tIHWcYhrfksfP0UI7sq%2FoqM4fCMqNRZ3%2FTzg2w4Reqx0AhIi%2Fs8SlJKdOl8PQaxAP1jeAoEalhslFjw4IFelY1sJ5wFUxEytx6Z7E6Py8i6JPYo18%2Fywj7m7ZngrUWWoRDZRtL%2Bqjt7kbJtiC4m%2FtIynHCi7Nw6FK%2FWSRctK8raLRYaEtp92dPB%2FclrmwmdtJW3ju57ljak1DRGtI7h%2BZpGmqbZpnOPcn08k%2BgTyagOaZl4%2BMRvLPYA%2Bc85Vx0CNCZeuS8CjM8Fn7HjcH9W%2FkaJRTpqOT%2F1qaURVACfj1VSkTqGPQ86j%2BX%2FaFvSTBgikfs5k3MlsXIal6zskIrhYaIPqB0wsLzzX7aVrjKaTUJI7B209GoGgAnFnUJXwfqfyZZ6HSghelwx0IjOy0QNy%2B3JJ%2ByCiTuo5DOWzebrHotJDPuklVPQwv962tQY6pwFTRhLafoqGHf2oqPFzJ6DYPfOcmjqXB9rEygL1QzKc%2BkQeZE4FldsBSIHNYH8%2Ba2qjHLQahqZVorBxT2DAI57o%2Ft%2Fls7w%2BwJWF%2FMjzW8oC0SQyR1%2BznLd69PblkBlMycrTVgqkem%2FmG%2BMeyrNzK01R14a6KO9ebYZrI94LT5dcrl3xvzV8w4dgkgBl6czGFiGOb%2FLEHpyp1gm9uVP3Y7Rjz%2FtOuDPUsg%3D%3D&X-Amz-Signature=c6c702b865ca1e2d0d4d6bf6e75af1280740a518a66a6b1917a39ef201be8969&X-Amz-SignedHeaders=host&x-id=GetObject)

Preview

安全性特点

●字节码验证: 在 Move on Sui 源代码编译为字节码后，Move 字节码验证器会进行静态分析，确保字节码遵循 Sui 的类型、内存和资源安全规则。

●资源管理: Move on Sui 提供对资源的本地支持，通过字节码验证来确保资源的安全性和完整性，防止资源被复制或丢弃。

●模块化设计: Move on Sui 支持模块化设计，方便开发人员构建复杂的 dApp，并提高合约的可扩展性和可维护性。

### 

 关键区别

与其他区块链上的 Move 相比，Move on Sui 有以下关键区别：

●以对象为中心的全局存储: 在 Move on Sui 中，每个对象和包都被赋予唯一标识符，所有交易的输入都使用这些标识符明确指定，从而使区块链能够并行调度具有不重叠输入的交易。

●地址表示对象 ID: 在 Move on Sui 中，地址被用作 32 字节标识符，用于表示对象和账户。

●具有 Ability 能力的对象和全局唯一 ID: 在 Sui 中，Ability 能力表示结构体是对象类型，并要求结构体的第一个字段具有签名 id: UID，以包含对象在链上的唯一地址。标识符绝不被重复用。

●模块初始化器: 在模块发布时，Move on Sui 运行时会执行一次模块中定义的特殊初始化函数init。

●入口点函数以对象引用作为输入: 我们可以从 Move on Sui 事务中调用公共函数，这些函数可以通过值、不可变引用或可变引用获取对象。

请注意：Move on Sui 并不是一门新的智能合约语言，它是 Move 语言的改进版，为了简化表述，后续文章统一使用 Sui 术语代表 Move on Sui。

### 

 小结

Sui 语言通过一系列的改进和优化，满足了 Sui 区块链对高效、安全和灵活编程语言的需求。其资源安全性、模块化设计和强类型系统，为开发者提供了强大的工具和保障。

希望通过这节课，你能够更好地理解 Sui 的技术特点和安全优势，并将这些知识应用到实际开发中！

## 以对象为中心的数据模型-对象组成

Reading

欢迎来到本节课！今天我们将一起探索：在以对象为中心的数据模型中，对象到底由什么组成？

### 

 面向资源的设计理念

Sui 区块链的独特之处在于其面向资源的设计理念。这种设计通过将链上资源视为可独立管理的对象，实现了对资源的高效利用和精细管理。

●对象（Objects）：在 Sui 中，每一个链上资源都被视为一个对象。这些对象可以是代币、智能合约、NFT 或其他类型的资产。

●对象的状态和生命周期：每个对象都有自己的状态和生命周期。对象的状态可以被交易或智能合约操作修改，而生命周期则决定了对象的创建、使用和销毁过程。

### 

 对象的组成

在 Sui 中，所有交易都需要对象作为输入，并生成新的或修改后的对象作为输出。每个对象都记录了产生它的最后一笔交易的哈希值。这些可用作输入的对象被称为“活动”对象。通过观察这些活动对象，我们可以了解整个区块链的当前状态。

●唯一标识符（objectId）：每个对象都有一个唯一标识符，用以区分其他对象。这个 ID 在对象创建时生成，并且是不可变的。它用于在系统内跟踪和识别对象。

●对象类型（objType）：每个对象都有一个类型，它定义了对象的结构和行为。不同类型的对象不能混合使用或互换，确保对象根据其类型系统正确使用。

●所有者（owner）：每个对象都与一个所有者相关联，所有者对对象有控制权。在 Sui 上，对象可以归某个账户独有，也可以所有账户共享，或者被冻结，允许只读访问而不允许修改或转移。

●版本（version) ：在 Sui 中对象每次变化都会生成一个新的版本号充当随机数，防止重放攻击。

●摘要（digest）：每个对象也都有一个摘要，即对象数据的哈希值。用于验证对象数据的完整性，并确保数据未被篡改。摘要在对象创建时计算，并在对象数据发生变化时更新。

●内容字段: 这是一个类别特定的可变大小区域，包含以二进制规范序列化（BCS）编码的数据，记录了对象的具体数据。这个区域可以根据对象的类型而变化，比如存储文件数据或智能合约代码。

如下为测试链的 [](https://suiscan.xyz/testnet/object/0x00bc6a9bfa84153db2d5113c10329e6fcd5c9617b83db81364d56d0096b7ce95)[0x00bc…ce95](https://suiscan.xyz/testnet/object/0x00bc6a9bfa84153db2d5113c10329e6fcd5c9617b83db81364d56d0096b7ce95) [对象](https://suiscan.xyz/testnet/object/0x00bc6a9bfa84153db2d5113c10329e6fcd5c9617b83db81364d56d0096b7ce95)，类型为 0x2::coin::Coin<0x2::sui::SUI>，即 Sui 的原生代币，归账户 0xc494…521e 所有，余额为 1.8 SUI。

![image](https://hackquest-s3-prod-apne1.s3.ap-northeast-1.amazonaws.com/courses/b7ac8a16-c482-449b-8f06-86fadefc569b/5b263322-5194-4715-9610-f22dff91b6e2/664e1c6a-32a5-44b2-a887-2e7cc04eede2/N0avSMQc9IA_Bht_vHfnm.webp?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAYCTGVDAPHARJARG5%2F20240803%2Fap-northeast-1%2Fs3%2Faws4_request&X-Amz-Date=20240803T053904Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjELT%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaDmFwLW5vcnRoZWFzdC0xIkYwRAIgUXvvFeU4%2BrcS%2BHUZ7%2BHP8vVSo4dGHPfRGxP4LC4d9xkCIHjT3ukFnv2DdnmYv%2BJqZ1GMP81qyex6Yiw8NQYlKD3qKqcECJ3%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEQARoMNTU1MzM5ODE0OTQyIgyHqjjOGrR1eauAbJAq%2BwPKX0hggH3UePfdyYbnpVm%2BdC0itvf7tXc2Rfi1hZCJti6zr5zxukXkDOobNC0veJIaKlz%2F1VtQt7xQvncJhbERLH%2BZUbXO56UE%2FAmsAuv7Ek2FV8bcTbK1DsjLY%2Bi8b%2BhcPFYCYuN1n7wsM6MtGndbS2R%2Fj8YKxNMyWL9gA3R%2BD0j62VRKaxEEe73Ofr1B5B7CqyXKpxFcQuFfaT4xhyg5Lzez4ZwHcvHQLCoInZ60Rhotm2n1tIHWcYhrfksfP0UI7sq%2FoqM4fCMqNRZ3%2FTzg2w4Reqx0AhIi%2Fs8SlJKdOl8PQaxAP1jeAoEalhslFjw4IFelY1sJ5wFUxEytx6Z7E6Py8i6JPYo18%2Fywj7m7ZngrUWWoRDZRtL%2Bqjt7kbJtiC4m%2FtIynHCi7Nw6FK%2FWSRctK8raLRYaEtp92dPB%2FclrmwmdtJW3ju57ljak1DRGtI7h%2BZpGmqbZpnOPcn08k%2BgTyagOaZl4%2BMRvLPYA%2Bc85Vx0CNCZeuS8CjM8Fn7HjcH9W%2FkaJRTpqOT%2F1qaURVACfj1VSkTqGPQ86j%2BX%2FaFvSTBgikfs5k3MlsXIal6zskIrhYaIPqB0wsLzzX7aVrjKaTUJI7B209GoGgAnFnUJXwfqfyZZ6HSghelwx0IjOy0QNy%2B3JJ%2ByCiTuo5DOWzebrHotJDPuklVPQwv962tQY6pwFTRhLafoqGHf2oqPFzJ6DYPfOcmjqXB9rEygL1QzKc%2BkQeZE4FldsBSIHNYH8%2Ba2qjHLQahqZVorBxT2DAI57o%2Ft%2Fls7w%2BwJWF%2FMjzW8oC0SQyR1%2BznLd69PblkBlMycrTVgqkem%2FmG%2BMeyrNzK01R14a6KO9ebYZrI94LT5dcrl3xvzV8w4dgkgBl6czGFiGOb%2FLEHpyp1gm9uVP3Y7Rjz%2FtOuDPUsg%3D%3D&X-Amz-Signature=23b9a6d77ae948ea2524d9830067f8f4cac7e89a3f5fe101e4c0e75c0d55b660&X-Amz-SignedHeaders=host&x-id=GetObject)

Preview

### 

 小结

可以把每个对象想象成一个独特的小盒子，里面有它的身份证（全局唯一 ID）、档案（元数据）、DNA 信息（类型化数据）和物品（内容字段）。这些组成部分让我们能够清楚地了解每个对象的详细信息和历史记录，使得 Sui 的数据模型更加灵活和可扩展。

希望通过这节课，你能够更好地理解 Sui 中对象的构成，并将这些知识应用到实际开发中！


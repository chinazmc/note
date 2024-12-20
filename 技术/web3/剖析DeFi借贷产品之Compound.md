# 剖析DeFi借贷产品之Compound：概述篇

原创 Keegan小钢 Keegan小钢 _2021年06月07日 08:30_

## 前言

我前段时间一直在研究 Compound，走过一点弯路，也趟过一些坑，最终把它啃了下来。最近有些小伙伴也在咨询我相关的一些问题，那我本着乐善好施的优良传统，决定将我所学的知识整理成文字分享出来。我打算分几篇文章来讲解，分别为：**概述篇、合约篇、Subgraph篇、清算篇、延伸篇**。

那么，先从概述篇开始。

## DeFi

DeFi 是 **Decentralized Finance** 的简称，即**去中心化金融**，是由**区块链、数字资产、金融服务**三者交叉形成的一个发展领域。DeFi 是指在区块链结算层上提供金融服务的去中心化应用程序（Dapps）的通用术语，包括支付、贷款、交易、投资、保险和资产管理等。

从 2020 年开始，DeFi 市场经历了爆炸性的增长。根据数据平台 **DeFi Pulse** 的数据，DeFi 服务的数字资产的**总锁定价值（TVL）** 从 2019 年的不到 10 亿美元增长到 2020 年底的 150 亿美元，2021 年 5 月中旬又达到了 880 亿美元的高峰。然而，这跟传统金融相比，还只是一个零头，所以 DeFi 依然还是处于非常早期的阶段。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmiclLOXrmRnrfanJmiajnJrNcyXFgD1cYw1HWiciaAiaSNBahlicxwlKb0vBZgYqfWrCkGerSKjA3SEibuiaYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

传统金融依赖于中心化的金融机构，比如银行。但 2008 年的金融危机，致使存款超过 1880 亿美元的**华盛顿互惠银行 (Washington Mutual)** 和资产超过 6390 亿美元的**雷曼兄弟 (Lehman Brothers)** 都双双倒闭。自 2008 年以来，人们就越来越关注现有的金融体系存在的**效率低下、结构性不平等**的问题，并关注其潜在风险。最近，诸如 GameStop 做空（散户投资者在市场波动期间被禁止交易）等争议突显出传统金融基础设施的其他缺陷：**结算周期缓慢、价格发现效率低下、流动性面临挑战，以及对基础资产缺乏保证**。DeFi 的目标就是解决其中一些挑战，尽管上述问题在目前的 DeFi 生态系统中也一样存在。DeFi 利用区块链技术来促进对传统服务提供商和市场结构的替代，它具有提高金融市场效率的潜力。

在整个 DeFi 行业中，已经有很多 Dapps 在提供服务，主要包含了：**稳定币（Stabelcoins）、交易所服务（DEX）、借贷服务（Lending）、衍生品服务（Derivatives）、保险服务（Insurance）、资产管理服务（Asset Management），以及辅助性的服务，如钱包和预言机（Oracle）**。在每个细分板块，都已经有多个服务提供者，也有很多产品提供了多种服务，整个 DeFi 行业已经呈现出了百花齐放的景象。

## 借贷

根据 **DeFi Pulse** 的统计数据显示，截止到本文发稿时，未偿还总债务为 15 亿美刀，而收录的借贷产品有 28 款之多，而没被收录的，市场上其实还有很多。其中，排名前三的借贷产品为 **Compound、Aave、Maker**，借款分别达到了 5.28 亿、5.1 亿、4.79 亿，三者总和已经占据了整个市场的 99% 以上。当然，跟传统借贷市场相比，连个零头都不到，所以才不断有新的玩家参与进来抢占蛋糕。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmiclLOXrmRnrfanJmiajnJrNcyT5vdYIAXsuYUm2ibfMxLPOYpX5tN911nEfX1J5PCElMiaO0lJabP17Hw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

目前，DeFi 中的借贷平台基本都为抵押贷款方式，通过超额抵押一种数字资产而借出另一种资产，比如抵押价值 2000 美刀的 ETH，可借出 1800 美刀的 USDT。这与传统的抵押贷款方式类似，不过，传统的抵押贷款本质上是将房子、车、土地等非流动性资产作为抵押，借出钱这种能够高度流动的资产。但 DeFi 中的抵押贷款，抵押的资产和借出的资产，都属于高流动性的资产，这为什么能有市场？初看起来有点难以理解。然而，因为 DeFi 借贷产品有着高额的抵押利率以及 DeFi 市场前期巨额的收益率吸引了较高的市场资金。比如，Compound 的流动性挖矿模式，用户就算借款也能得到挖矿的平台币奖励，这奖励扣减掉借款利息后净收益还是正的，所以才吸引了众多用户参与其中。

目前 DeFi 借贷市场的需求主要有几点：

1. **满足交易活动的资金需求**：包括套利、杠杆、做市等交易活动，这是最主要的刚需。
    
2. **获得被动收入**：主要满足那些长期持有数字资产又希望能够产生额外收益的投资者。
    
3. **获得一定的流动性资金**：主要是矿工或一些行业内的初创企业存在一些短期性流动资金的需求。
    

第三点是比较符合传统借贷业务逻辑的，但在 DeFi 借贷市场中，占比较低。最主要的需求还是第一种情况，比如，我手上持有 ETH，而且我看好 ETH 在接下来的时间内还会上涨，那我可以抵押我的 ETH，借出 USDT，用来买更多 ETH，而且买回来的 ETH 还可以再抵押进去，再借出一些 USDT，再买更多 ETH。这样子，一份资金还可以获得多项收益。本质上，这种需求其实就是为了增加杠杆，也因此，开始有不少 DeFi 产品直接提供杠杆挖矿、杠杆交易的服务。

DeFi 借贷这种超额抵押贷款的模式，其主要缺点就是资金利用率低。信用贷款显然比抵押贷款会更有效率，但在目前的区块链匿名环境下，难以实现。Aave 的**闪电贷**是在区块链世界中实现无抵押贷款的首创者，利用了区块链独有的特性而实现，需要在一个区块内完成借款和还款，否则就失效。适用场景非常有限，且存在技术门槛，普通用户无法参与。

## Compound

Compound 是目前借贷市场中的龙头，据我们做过的调研，有不少同类产品都是基于 Compound 做的修改，我也是对 Compound 研究得比较深，从产品到技术。

Compound 的模式比较简单，就是每种借贷资产都会开设一个资金池，存取借还都是从资金池里流入和流出。要理解 Compound 的业务逻辑，有几个相关概念也需要理解：

- **标的资产（Underlying Token）**：即借贷资产，比如 ETH、USDT、USDC、WBTC 等，目前 Compound 只开设了 14 种标的资产。
    
- **cToken**：也称为生息代币，是用户在 Compound 上存入资产的凭证。每一种标的资产都有对应的一种 cToken，比如，ETH 对应 cETH，USDT 对应 cUSDT，当用户向 Compound 存入 ETH 则会返回 cETH。取款时就可以用 cToken 换回标的资产。
    
- **兑换率（Exchange Rate）**：cToken 与标的资产的兑换比例，比如 cETH 的兑换率为 0.02，即 1 个 cETH 可以兑换 0.02 个 ETH。兑换率会随着时间推移不断上涨，因此，持有 cToken 就等于不断生息，所以也才叫生息代币。计算公式为：exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
    
- **抵押因子（Collateral Factor）**：每种标的资产都有一个抵押因子，代表用户抵押的资产价值对应可得到的借款的比率，即用来衡量可借额度的。取值范围 0-1，当为 0 时，表示该类资产不能作为抵押品去借贷其他资产。一般最高设为 0.75，比如 ETH，假如用户存入了 0.1 个 ETH 并开启作为抵押品，当时的 ETH 价值为 2000 美元，则可借额度为 0.1 * 2000 * 0.75 = 150 美元，可最多借出价值 150 美元的其他资产。
    

当用户存入**标的资产**后，Compound 会根据**兑换率**返回与标的资产相对应的 **cToken** 给到用户，作为一种存款凭证。当需要赎回存款时，将 cToken 还回去，Compound 会根据最新的兑换率计算出需要赎回的标的资产的数量并返还给用户。比如，用户存入 1 个 **ETH**，当时的兑换率为 0.1，则返回 1/0.1 = 10 个 **cETH**，等到赎回时，假设兑换率已经升到了 0.15，则那 10 个 cETH 可赎回 10*0.15 = 1.5 个 ETH，多出的 0.5 个 ETH 就是利息所得。

当用户将存入的资产开启作为抵押品之后，则可以进行借款了。不过，不是所有资产都可以作为抵押品，比如 USDT 就不可以作为抵押品，其抵押因子为 0。ETH、DAI、USDC 的抵押因子都是 0.75，即 75%，表示价值 100 美元的抵押资产，可借额度为 75 美元，即最多可以借出价值 75 美元的数字资产。不过，不建议用完所有额度，不然，存在被清算的风险。

当用户的借款价值已经超过借款额度的时候，就可以被清算了。不过，智能合约没办法自动清算，所以需要外部的清算人调用智能合约的清算函数来执行清算。而为了激励第三方清算人来执行清算，就会有个清算激励，该激励由被清算人（即借款人）来承担。另外，清算人一般都是由程序化的清算服务来承担。

Compound 的清算模式属于**代还款清算**的方式，即清算人会帮借款人进行部分还款，并得到与还款资产价值同等的抵押资产，同时加上一定比例的清算激励，清算激励也是抵押资产。比如，借款人 A 有 100 USDT 的借款，而且抵押物为 ETH；那清算人 B 对 A 进行清算时，则需要帮 A 部分还款，目前 Compound 每次清算最多可代还款 50%，即 B 可以帮 A 还 50 USDT，假设清算激励为 0.05，那 B 将可以得到 A 的价值为 50 * (1 + 0.05) = 52.5 USDT 的 ETH。

## 利率模型

Compound 的利率模型主要有两种，第一种，我称为**直线型**的，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmiclLOXrmRnrfanJmiajnJrNcynceG8crQRweoHYrS5bHIOvt8GaQdGjWPWTiaKJBer4dzkT9iaAAOEdjg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其中，黑色线为横轴，表示**使用率**，紫色线表示**借款利率**，绿色线表示**存款利率**。

Compound 早期的时候都是这种利率模型，目前，依然有少数几种资产保留着这种模型。

使用率即是资金的使用率，计算公式为：

```
utilizationRate = borrows / (cash + borrows - reserves)
```

其中，borrows 表示借款额，cash 表示资金池余额，reserves 表示储备金。借款利率由使用率决定，存款利率由借款利率决定。

使用率本质上就是反映借贷供求关系的一个量化指标，使用率低说明存款的多，但借款的太少，即供给大于需求。这时候就需要鼓励用户多借款少存款，所以借款利率偏低，存款利率也偏低。而使用率高的时候则相反，借款利率和存款利率偏高，就能鼓励大家多存款少借款。

使用率偏高的话，那说明资金池里剩下的钱就比较少了，会面临资金池枯竭的风险。资金池枯竭的话，那存款用户就没资金可取，也没资金可借，这就可能会导致系统性风险了。一般资金使用率在 80% 以内比较安全。

而为了控制使用率在安全范围内，Compound 后来就升级了利率模型，我称之为**拐点型**，看下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmiclLOXrmRnrfanJmiajnJrNcy4kBzsJ8AFI1HJDzqKEkbSDC7d9IxM2ABekrrzDFDl57jLbmo8LB8fw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可看到，以使用率 80% 为拐点，之后的斜率陡然剧增，超高的借款利率一般就能有效降低借款需求，而超高的存款利率则能鼓励大家提高供给，从而能有效降低整体的使用率，避免资金池枯竭。

目前，Compound 大部分资金池都是采用这种利率模型。

## 技术架构

在技术层面，整个 Compound 借贷系统的整体架构大概如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmiclLOXrmRnrfanJmiajnJrNcytz2yA7MW3Y3aDPXooIOak2S6fEcNLxfMhMIKeY8lHYaRLibUpRxnL1w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图中的每个模块我简单介绍一下：

- **Ethereum**：底层的区块链系统，Compound 是部署在以太坊上的，但稍作修改其实也可以部署到 Heco、BSC 等链上。
    
- **智能合约**：所有核心业务都是使用智能合约实现的，代码也都已开源。
    
- **PriceOracle**：价格预言机，会定时将那些数字资产的价格设置到智能合约上，从而智能合约可以获取资产价格做一些相应的计算。Compound 使用自己设计的 Open Price Feed 作为价格预言机。
    
- **Subgraph**：数据索引服务，需部署到 Graph 节点才能使用，为前端和清算服务提供一些结构化的数据查询功能，比从链上查数据更方便快捷。
    
- **清算服务**：承担清算人的角色，会定时查询出待清算账户并调用智能合约执行清算操作。
    
- **钱包**：如 MetaMask 这样的钱包工具。
    
- **Web客户端**：给普通用户使用的前端页面，集成 ApolloClient 连接 Subgraph，再通过 Web3 API 连接 MetaMask 等钱包进行操作，也有一些数据无法从 Subgraph 读取，而需要直接查询合约数据。
    

## 总结

关于 DeFi、借贷、Compound 的内容，暂时就讲这么多了，概述篇主要还是从整体上做一个介绍，包括了解 DeFi 和借贷市场的一些现状和前景，以及 Compound 产品层面的一些基本概念和核心业务逻辑，还有对 Compound 的整体技术架构有个初步认识。

后面再陆续深入每一个模块的技术实现，敬请期待！



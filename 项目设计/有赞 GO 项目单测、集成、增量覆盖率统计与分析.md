# 有赞 GO 项目单测、集成、增量覆盖率统计与分析

![](https://tech.youzan.com/you-zan-go-xiang-mu-dan-ce-ji-cheng-zeng-liang-fu-gai-lu-tong-ji-yu-fen-xi/)

## 一、引言

我是一名中间件 QA，我对应的研发团队是有赞 PaaS，目前我们团队有很多产品是使用 go 语言开发，因此我对 go 语言项目的单测覆盖率、集成以及增量测试覆盖率统计与分析做了探索。

## 二、单测覆盖率以及静态代码分析

### 2.1､单测覆盖率分析

Go 语言自身提供了单元测试工具 `go test`，单元测试文件必须以 `*_test.go` 形式存在，`go test` 工具同时也提供了分析单测覆盖率的功能。因为需要将单测覆盖率上传到 sonar 平台展示，所以必须将覆盖率文件转换成能被 sonar 识别的格式，因此，还需要另外一个命令行工具 [gocov](https://github.com/axw/gocov)。

首先我们使用 `go test` 生成覆盖率输出文件 `cover.out`，并通过 gocov 工具来将生成的覆盖率文件 `cover.out` 转换成可以被 sonar 识别的 Cobertura 格式的 xml 文件。 如下所示：

```shell
go test -v ./... -coverprofile=cover.out #生成覆盖率输出  
gocov convert cover.out | gocov-xml > coverage.xml #将覆盖率输出转换成xml格式的报告  
```

将生成的单测覆盖率报告发送到 sonar 平台上来展示。

### 2.2､静态代码分析

Go 静态代码分析工具有两个，分别是 [gometalinter](https://github.com/alecthomas/gometalinter) 和 [golangci-lint](https://github.com/golangci/golangci-lint)，我们现在使用的是 **golangci-lint**，因为 **gometalinter** 已经停止维护，而且作者也推荐去使用 **golangci-lint**。

#### 2.2.1 golangci-lint 的安装

以下是安装 golangci-lint 推荐的两种方法：

- 将二进制文件安装在 (go env GOPATH)/bin/golangci-lint 目录下 `curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s -- -b $(go env GOPATH)/bin vX.Y.Z`
- 或者将二进制文件安装在 ./bin/ 目录下 `curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s vX.Y.Z`

安装完成之后可以通过使用`golangci-lint --version`来查看它的版本。

#### 2.2.2 golangci-lint 的使用

在需要进行静态代码扫描的目录下执行 `golangci-lint run`，此命令和 `golangci-lint run ./…` 命令等效，表示扫描整个项目文件代码，并进行监测，也可以通过指定 go 文件或者文件目录名来对特定的代码文件或者目录进行代码扫描，例如 `golangci-lint run dir1 dir2/... dir3/file1.go`。

> ps：扫描指定目录的时候是不支持递归扫描的，如果要进行递归扫描需要在目录路径后面追加`/…`

默认情况下 golangci-lint 只启用以下的 linters：

**Enabled by default linters:**

- **deadcode**: 发现没有使用的代码
- **errcheck**: 用于检查 go 程序中有 error 返回的函数，却没有做判断检查
- **gosimple**: 检测代码是否可以简化
- **govet (vet, vetshadow)**: 检查 go 源代码并报告可疑结构，例如 Printf 调用，其参数与格式字符串不一致
- **ineffassign**: 检测是否有未使用的代码、变量、常量、类型、结构体、函数、函数参数等
- **staticcheck**: 提供了巨多的静态检查，检查 bug，分析性能等
- **structcheck**:发现未使用的结构体字段
- **typecheck**: 对 go 代码进行解析和类型检查
- **unused**: 检查未使用的常量，变量，函数和类型
- **varcheck**: 查找未使用的全局变量和常量

**Disabled by default linters:**

- **bodyclose**: 对 HTTP 响应是否 close 成功检测
- **dupl**: 代码克隆监测工具
- **gochecknoglobals**: 检查 go 代码中是否存在全局变量
- **goimports**: 做所有 gofmt 做的事. 此外还检查未使用的导入
- **golint**: 打印出 go 代码的格式错误
- **gofmt**: 检测代码是否都已经格式化, 默认情况下使用 `-s` 来检查代码是否简化
- **…………………………..**

未启用的还有很多工具，可以通过使用 `golangci-lint help linters` 命令查看还有哪些工具可以使用，如果想要启用没有默认开启的工具，可以在执行命令时使用 `-E` 参数来启用，比如要启用 golint 的话，只需要执行一下命令 `golangci-lint run -E=golint`。除了用 `-E` 来启动参数外，还可以指定最长执行时间 `—deadline`、跳过要扫描的目录 `--skip-dirs` 等等。如果要了解更多，请使用 `golangci-lint run -h` 来查看。

特别注意 `—-exclude-use-default` 参数，golangci-lint 对于上面默认的启用 linters 中做了一些过滤措施，比如对于 `errcheck` ，它不会扫描 `((os\.)?std(out|err)\..*|.*Close|.*Flush|os\.Remove(All)?|.*printf?|os\.(Un)?Setenv)` 这些函数返回的 error 是否被 checked，所以如果代码中使用到这些函数，并且没有接收 error 的话是不会被扫描到的。类似的还有`golint`、`govet`、`staticcheck`、`gosec` 需要注意。如果想要不过滤这些就需要使用 `--exclude-use-default=false` 来启用。

### 2.3､接入sonar

go 接入 sonar 需要 sonar-scanner 工具以及 sonar-project.properties 文件。

#### 2.3.1 sonar-scanner

sonar-scanner 是 sonar 官方提供的代码扫描器，下载地址是 `https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner`。下载好之后解压，解压后的目录下有四个文件夹，分别是 bin、conf、jre、lib，然后将 bin 文件夹路径添加到 $PATH 环境变量下，使用 `sonar-scanner -v` 来查看版本。

#### 2.3.2 sonar-project.properties

sonar-project.properties 文件的作用主要是配置 sonar 扫描器扫描哪些类型的文件以及文件目录，最后将报表结果上报到 sonar 服务器，sonar-project.propertie 内容如下：

```properties
#sonar安装的服务器地址
sonar.host.url=http://ip:port  
#服务器账号
sonar.login=root  
#服务器密码
sonar.password=root  
#项目使用的语言
sonar.language=go  
#项目的独特关键字,maven 项目是 <groupId>:<artiactId>，go 项目自己定义就可以
sonar.projectKey=projectKey  
#将在web界面上显示的名字
sonar.projectName=demo  
#项目版本
sonar.projectVersion=1.0  
#需要分析的源码目录的路径
sonar.sources=.  
sonar.exclusions=**/*_test.go,**/vendor/**  
sonar.tests=.  
sonar.test.inclusions=**/*_test.go  
sonar.test.exclusions=**/vendor/**  
#golangci-lint 报告路径
sonar.go.golangci-lint.reportPaths=report.xml  
#单测覆盖率报告地址
sonar.go.coverage.reportPaths=cover.out  
```

在项目目录下分别执行`go test -v ./... -coverprofile=cover.out`以及`golangci-lint run --out-format checkstyle ./... > report.xml`等生产报告，并执行sonar-scan 来将生成的报告上传到服务器。这里默认在使用的是sonar8.1 已经支持了`golangci-lint`

**报告主页**

![sonar](https://tech.youzan.com/content/images/2020/02/sonar-1.png)

## 三、集成测试覆盖率分析

对于 Go 项目没有类似 java jacoco 这样的第三方测试工具，就算是开源的第三方工具，一般单元测试执行以及单测覆盖率分析都是使用 Go 自带的测试工具 `go test` 来执行的。

阅读了[GO的官方博客](https://blog.golang.org/cover)之后发现其实针对二进制文件是有类似的工具 gcov。在文章中作者也说了，对于在 go 1.2 之前，其实也是使用类似 gcov 的方式对二进制程序在分支上设置断点，在每个分支执行时，将断点清除并将分支的目标语句标记为 “covered” 。

但是通过文章可以知道，在 go 1.2 之后是不支持使用此种方式，而且也不推荐使用 gcov 来统计覆盖率，因为执行二进制分析是很有挑战且很困难的，它还需要一种可靠的方式来执行跟踪绑定到源代码，这也很困难，这些问题包括不准确的调试信息和类似内联函数使分析复杂化，最重要的是，这种方法非常不便携。

### 3.1､解决方法

通过查找[资料](https://www.elastic.co/cn/blog/code-coverage-for-your-golang-system-tests)，发现了一个并不完美但是可以解决这个问题的方法。go test 中有一个 -c 的 flag，可以将单测的代码和被单测调用的代码编译成二进制包执行，但是这种方式并没有将整个项目的代码包含进去，不过可以通过增加一个测试文件 main_test.go，文件内容如下：

```go
func TestMainStart(t *testing.T) {  
    var args []string
    for _, arg := range os.Args {
        if !strings.HasPrefix(arg, "-test") {
            args = append(args, arg)
        }
    }
    os.Args = args
    main()
}
```

将主函数放在此测试代码中，由于 Go 的入口函数是 main 函数，所以这样就会将整个 Go 项目都打包成一个已经插桩的二进制文件，如果项目启动的时候需要传入参数，则会将其中程序启动时传入的不是 -test标记的参数放入到os.Args 中传递给main 函数。以上代码也可以自己在测试文件中增加消息通知监听，来退出测试函数。

当集成测试跑完后就可以得到覆盖率代码，整个流程可参考下图：

![ci_test](https://tech.youzan.com/content/images/2020/02/ci_test.png)

```shell
#第一步：执行集成测试，并将此函数编译成二进制文件
go test -coverpkg="./..." -c -o cover.test  
#第二步：运行二进制文件，指定运行的测试方法是 TestMainStart，并将覆盖率报告输出
./cover.test -test.run "TestMainStart" -test.coverprofile=cover.out
#第三步：将输出的覆盖率报告转换成 html 文件（html 文件查看效果比较好）
go tool cover -html cover.out -o cover.html  
#第四步：生成 Cobertura 格式的 xml 文件
gocov convert cover.out | gocov-xml > cover.xml  
```

### 3.2､缺点

1. 必须所有 Go 语言项目中新增一个这样的测试代码文件，才可以使用
    
2. 必须退出进程才可以获得报告，但是如果测试程序是在 k8s 的 pod 中，一旦程序退出，pod 就会自动退出无法获取到文件
    
3. 想要得到测试覆盖率数据不能像 jacoco 那样直接调用接口可以 dump 到本地，程序必须增加一个接收信号量的参数，保证主函数的退出，不然集成测试代码跑完，覆盖率信息是不会写到磁盘的
    
4. 由于上面的原因，报告储存在远端，无法下载到当前 Jenkins 上，要去远端 dump 文件下来分析
    
5. 不能将分布式的应用的数据结合起来之后做全量统计(只能跑单个应用)
    
    **以上缺陷在有赞paas团队通过一些不是特别优雅的方式解决,以下是解决方案**
    

### 3.3､优化

> ps：由于当前有赞 PaaS 的 ci 环境是在 k8s 集群中实现的，所以这里就针对 k8s中 的优化方案

**3.3.1、针对编译前需要新增一个测试文件，包裹main函数**

测试函数也是要求所有项目中增加一个测试文件，或者 Jenkins 编译部署镜像之前在 pipline 中生成一个文件

**3.3.2、针对以上必须程序退出才可以或许到测试覆盖率报告的缺点：**

假设 k8s 基础镜像中已经装好 python，我在启动 pod 的时候默认启动两个服务，一个是被测试的服务，一个是 python 启动的 http 服务。

然后将项目服务的启动写入脚本中，并在 deployment 中通过 nohup 启动服务，并再启动一个 python 服务

```yaml
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - (nohup /data/project/start.sh &);(cd python && -m SimpleHTTPServer 12345)
        image: $imageAddress
```

杀死项目服务后，因为还有 python 服务在，pod 不会退出，可以拿到覆盖率测试报告

**3.3.3、覆盖率报告在远端，如何在跑完Jenkins任务后来直接获取到报告：**

可以在跑集成测试后通过执行 http 请求来获取容器内的 cover.out，比如 `wget http://{ip}:{port}/{path}/cover.out`，并将此覆盖率报告编译成 Cobertura 格式的 xml，放入到 Jenkins 中统计。

如果是执行了多个服务端，需要合并覆盖率报告，可以使用 [gocovmerge](https://github.com/wadey/gocovmerge)

**3.3.4、如何在k8s中自动化kill程序让其退出：**

对于退出程序可以直接在集成测试代码中使用 kubectl 命令将 pod 中的程序 kill

```shell
pid=`kubectl exec $podname -c $container -n dts -- ps -ef | grep $process | grep -v grep | awk '{print $2}'`  
kubectl exec $podname -c $container -n $namespace -- kill $pid  
```

### 3.4､jenkins 报告

![xml_cover](https://tech.youzan.com/content/images/2020/02/xml_cover.png)

## 四、集成测试增量覆盖率分析

### 4.1､diff_cover

增量覆盖率分析我们选择了开源工具 [diff_over_](https://www.github.com/Bachmann1234/diff_cover)_，diff_cover 是用 python 开发，通过 git diff 来对比当前分支和需要比对的分支，主要针对新增代码做覆盖率分析。

### 4.2､安装

安装 diff_cover的机器需要有 python 的环境，有两种安装方式:

1、通过pip 来直接下载安装

`pip install diff_cover`

2、通过源代码安装

`pip install diff_covers`

### 4.3､使用方式

> ps：必须在需要对比的项目目录下运行！！！

#### 4.3.1 生成单元测试覆盖率报告

`go test -v ./... -coverprofile=cover.out gocov convert cover.out | gocov-xml > coverage.xml`

#### 4.3.2 增量覆盖率分析

`diff-cover coverage.xml --compare-branch=xxxx --html-report report.html`

> --compare-branch：是选择需要对比的分支号
> 
> --html-report：是将增量测试报告生成 html 的报告模式
> 
> 除了以上参数，此工具还有很多其他参数，比如
> 
> --fail-under：覆盖率低于某个值，返回非零状态代码
> 
> --diff-range-notation：设置 diff 的范围,就是 `git diff {compare-branch} {diff-range-notation}` 的作用等等。
> 
> 具体可以通过 `diff_cover -h` 来获得更多详细的信息

### 4.4､报告

1. 命令行展示  
    ![diff_cover1](https://tech.youzan.com/content/images/2020/02/diff_cover1.png)
2. HTML展示

![diff_cover2](https://tech.youzan.com/content/images/2020/02/diff_cover2.png)

表格中可以看到当前分支覆盖率与选定分支覆盖率的差异。


# Reference
https://tech.youzan.com/you-zan-go-xiang-mu-dan-ce-ji-cheng-zeng-liang-fu-gai-lu-tong-ji-yu-fen-xi/
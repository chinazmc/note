Golang的IO读写提供了很多种方式，目前本人知道的有io库、os库、ioutil库、bufio库、bytes/strings库等。

虽然库多是一件好事，意味着选择性多，但让我困惑的一点是：什么场景下该用哪个库？ 为什么？

在给出结论前，我先想给出Golang内置IO库的项目结构，主要方便理解和引用：
```go
# 只列举了核心的目录及文件
src:
  - bufio
    - bufio.go
  - bytes
    - buffer.go
    - reader.go
  - io
    - ioutil
      - ioutil.go
    - io.go
  - os
    - file.go
  - strings  
    - reader.go
```

1.`io库`属于底层接口定义库，其作用是是**定义一些基本接口和一些基本常量**，并对这些接口的作用给出说明，常见的接口有Reader、Writer等。一般用这个库只是为了调用它的一些常量，比如`io.EOF`。

2.`ioutil库`包含在io目录下，它的主要作用是**作为一个工具包**，里面有一些比较**实用的函数**，比如 ReadAll(从某个源读取数据)、ReadFile（读取文件内容）、WriteFile（将数据写入文件）、ReadDir（获取目录）

3.`os库`主要是跟**操作系统打交道**，所以文件操作基本都会跟os库挂钩，比如创建文件、打开一个文件等。这个库往往会和ioutil库、bufio库等**配合使用**

4.`bufio库`可以理解为在`io库`上再封装一层，加上了缓存功能。它可能会和`ioutil库`和`bytes.Buffer`搞混。  
4.1 `bufio VS ioutil`库：两者都提供了对文件的读写功能，唯一的不同就是bufio多了一层缓存的功能，这个优势主要体现读取大文件的时候（ioutil.ReadFile是一次性将内容加载到内存，如果内容过大，很容易爆内存）

4.2 `bufio VS bytes.Buffer`：两者都提供一层缓存功能，它们的不同主要在于 bufio 针对的是**文件到内存**的缓存，而 bytes.Buffer 的针对的是**内存到内存**的缓存（个人感觉有点像channel，你也可以发现 bytes.Buffer 并没有提供接口将数据写到文件）。

5.`bytes和strings`库：这两个库有点迷，首先它们都实现了Reader接口，所以它们的不同主要在于**针对的对象不同**，bytes针对的是字节，strings针对的是字符串（它们的方法实现原理很相似）。另一个区别就是 bytes还带有Buffer的功能，但是 strings没提供。

注：关于Reader和Writer接口，可以简单理解为读取源和写入源，即只要实现Reader里面的Read方法，这个东西就可以作为一个读取源，里面可以包含数据并被我们读取；Writer亦是如此。

以上就是个人的一些结论，下面会针对以上结论做进一步说明，如果有错误的地方麻烦请留言指正，比心❤️！

## 窥探 io 库

io库比较常用的接口有三个，分别是`Reader`，`Writer`和`Close`。
```go
// Read方法会接收一个字节数组p，并将读取到的数据存进该数组，最后返回读取的字节数n。
// 注意n不一定等于读取的数据长度，比如字节数组p的容量太小，n会等于数组的长度
type Reader interface {
	Read(p []byte) (n int, err error)
}

// Write 方法同样接收一个字节数组p，并将接收的数据保存至文件或者标准输出等，返回的n表示写入的数据长度。
// 当n不等于len(p)时，返回一个错误。
type Writer interface {
	Write(p []byte) (n int, err error)
}

// 关闭操作
type Closer interface {
	Close() error
}
```
关于 Read 方法的具体实现，可以在strings库中看到：
```go
// 定义一个Reader接口体
type Reader struct {
	s        string
	i        int64 // current reading index
	prevRune int   // index of previous rune; or < 0
}

// 通过NewReader方法得到 reader 对象，这里有个关键的地方是传入的字符串被赋值到 s 变量中
func NewReader(s string) *Reader { 
  return &Reader{s, 0, -1} 
}

// Read方法： 核心是 copy 方法，参数b虽然是切片，但是copy方法会影响到它的底层数组
func (r *Reader) Read(b []byte) (n int, err error) {
	if r.i >= int64(len(r.s)) {
		return 0, io.EOF
	}
  r.prevRune = -1
  // 核心方法
	n = copy(b, r.s[r.i:])
	r.i += int64(n)
	return
}
```

## 窥探 ioutil 库

上面提到，ioutil 库就是一个工具包，里面主要是比较实用的函数，比如ReadFile、WriteFile等，唯一需要注意的是它们都是一次性读取和一次性写入，所以当读取的时候**注意文件不能过大**。

从文件读取数据：
```go
func readByFile()  {
	data, err := ioutil.ReadFile( "./lab8_io/file/test.txt")
	if err != nil {
		log.Fatal("err:", err)
		return
	}
	fmt.Println("data", string(data)) // hello world！
}
```

把数据写入到文件：
```go
func writeFile() {
	err := ioutil.WriteFile("./lab8_io/file/write_test.txt", []byte("hello world!"), 0644)
	if err != nil {
		panic(err)
		return
	}
}
```
遍历目录：遍历目录有一个需要注意的是它的排序并不是自然排序方式。

## 窥探bufio库

bufio 库在上面也提到过，它主要是在io库上加了一层缓存的功能，以下是bufio读取大文件的例子：
```go
func readBigFile(filePath string) error {
	f, err := os.Open(filePath)
	defer f.Close()

	if err != nil {
		log.Fatal(err)
		return err
	}

	buf := bufio.NewReader(f)
	count := 0
	for {
		count += 1
		line, err := buf.ReadString('\n')
		line = strings.TrimSpace(line)
		if err != nil {
			return err
		}
    fmt.Println("line", line)
    // 这里是避免全部打印
		if count > 100 {
			break
		}
	}
	return nil
}


```

注：  
1.bufio 的ReadLine/ReadBytes/ReadString/ReadSlice： ReadString和ReadBytes等同，ReadBytes和ReadLine都调用了ReadSlice

## 窥探bytes/strings库

前面提过，就单纯实现Reader接口，bytes和strings底层函数的实现方式是差不多的，可以查看其源码得证：
```go
// bytes/reader.go
// Read implements the io.Reader interface.
func (r *Reader) Read(b []byte) (n int, err error) {
	if r.i >= int64(len(r.s)) {
		return 0, io.EOF
	}
	r.prevRune = -1
	n = copy(b, r.s[r.i:])
	r.i += int64(n)
	return
}

// strings/reader.go
func (r *Reader) Read(b []byte) (n int, err error) {
	if r.i >= int64(len(r.s)) {
		return 0, io.EOF
	}
	r.prevRune = -1
	n = copy(b, r.s[r.i:])
	r.i += int64(n)
	return
}
```

# 硬核，图解bufio包系列之读取原理

### 01 Go中普通的文件读写

首先我们来看看在Go中对文件的普通读取方式是怎么样的。下面是普通的读取文件内容的示例代码：

```text
package main

import (
    "fmt"
    "io/ioutil"
    "os"
    "sync/atomic"
)

func main() {
    filename := "./test.txt"
    //以读写模式打开文件
    fd, _ := os.OpenFile(filename, os.O_RDWR, 0666)
    b := make([]byte, 2)
    // 从文件中读取最多len(b)个字节的内容到b中
    n, err := fd.Read(b)
    fmt.Printf("n:=%d, b:=%s, err=%+v\n", n, b, err)
}
```

上面的读取方式是通过文件系统的IO进行读取的，每次都需要一次底层的系统调用，若需要连续多次读取，那么这种方式的效率就会大大降低。如下图：
![](https://pic1.zhimg.com/80/v2-ee52c26dd59cbe555a25c895e9847748_720w.webp)
在Go中，将读写文件的操作抽象成了接口io.Reader和io.Writer，只要实现对应接口的方法即可。如示例中通过os.OpenFile函数返回的File对象即实现了这两个接口。

那有没有什么办法提高读写效率呢？那就是编程中常用的技术--**缓存**。

### 02 将文件内容预读取到缓存--bufio

这里的思想很简单，当用户从文件中读取数据的时候，先从文件中读取一大块内容到内存缓冲区，以供后面的读取操作直接从内存缓冲区进行读取，以降低从文件中读取的系统调用次数。如下图所示：

![](https://pic4.zhimg.com/80/v2-e122616e4045efb65c51dc4d3ca1a17f_720w.webp)
整体思想比较简单。但在bufio中的具体实现中针对不同的场景使用了不同的策略机制。下面我们先看下缓冲区的几种状态，然后再针对缓冲区的每一类读取操作来深入分析其具体的实现策略。

### 03 缓冲区的状态

缓冲区有三种状态，分别是缓冲区为空、缓冲区未满但有可读取的数据以及缓冲区满的状态。在bufio中，缓冲区本质上是一个字节切片，并通过两个整型变量r和w分别表示可读取以及可写入的索引位置。从文件中每加载一个字节的内容到缓冲区则w+1，从缓冲区每读走一个字节的内容，则r+1。下面我们分别看下三种状态。

### 1）缓冲区为空的状态

缓冲区为空的状态本质上是指没有内容可读。其判断标准如下：

> r\==w  

r和w相等，意味着已经将写入到缓冲区的内容都读完了。其中最简单的就是r和w都等于0，缓冲区中没有任何内容，如下图所示：
![](https://pic4.zhimg.com/80/v2-3e030dbb86153948077f5f65452940b3_720w.webp)
缓冲区为空的状态还有一种情况是缓冲区中有内容，但已经都被读取走了，即r和w相等，如下图：

![](https://pic1.zhimg.com/80/v2-35e540b71aa7364ec2605ee81204f73c_720w.webp)

在这种状态下，当再需要读取内容时，会首先将r和w都置为0，然后从文件中加载新的数据填充到缓冲区中以供下次调用方读取。

### 2）缓冲区为非空的状态

这种状态是指在缓冲区中有可读的内容，其判断标准如下：

> r != w && (w-r) < len(buf)  

r != w 说明buf[r:w]这段内容还没被调用方读取。(w-r) < len(buf) 说明buf不是满的状态，还有空间可以继续填充内容。 在这种状态下，当程序执行读操作时，会直接从缓冲区中读取。如下图：

![](https://pic3.zhimg.com/80/v2-f9c124d0c6fbb5c5c9df6fbc5ae817d6_720w.webp)

上图中，虚线部分表示已经被读走的内容。buf[r:w]这段切片是可读的缓冲内容，在该示例中为buf[2:8]。

### 3）缓冲区满状态

还有最后一种缓冲区状态，即缓冲区满。其通用的判断标准如下：

> (w-r) >= len(buf)  

如果要满足上述公式，只有一种情况，即 r=0，表示还没有从缓冲区读走任何内容。w=len(buf)，表示从文件中读取的内容已经填满了整个缓冲区。该示例中为w=10，即表示没有任何空闲的空间。如下图所示：

![](https://pic3.zhimg.com/80/v2-5b59f6de73c98181a75a4863396ad662_720w.webp)

以上就是缓冲区的三种状态，下面我们来开始看看bufio的具体是如何从缓冲区读取数据的。

### 04 读取特定字节数的操作-- Read([]byte)

我们还是从最简单的读取操作开始。还是上面的例子，我们将其改写成使用bufio的读取操作：
```text
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    filename := "./test.txt"
    //以读写模式打开文件
    fd, _ := os.OpenFile(filename, os.O_RDWR, 0666)

    //将fd包装到buffer Reader中
    bufioReader := bufio.NewReader(fd)

    p := make([]byte, 2)
    n, _ := bufioReader.Read(p)

    fmt.Printf("n=%d,p=%s", n, p)
}
```

基本用法看起来和直接从文件中读取差不多，只不过是多包装了一层buffer Reader。下面我们看看其内部的具体实现。

上面提到bufio的基本思想是有一个缓冲区，调用方直接从缓冲区中读取。下图是其初始的状态：
![](https://pic2.zhimg.com/80/v2-143f35c0e4e8d95698ce97f956c6b2a9_720w.webp)

为了方便演示，上图中是一块大小只有5个字节的缓冲区，**当然在bufio中实际默认的缓冲区大小是4096字节，即4KB**。下面我们看该读取函数在缓冲区的三种状态下各自的读取策略。

### 场景一：当缓冲区为空状态时的读取逻辑（即r等于w）
在缓冲区为空时，进行读取也有两种情况：
- 若调用方要读取的字节数 小于 缓冲区的长度，则先填充缓冲区，再从缓冲区中读取。
- 若调用方要读取的字节数 ≥ 缓冲区的长度，则直接从文件中读取，不填充缓冲区。

下面我们先来看第一种情况：要读取的字节数小于缓冲区的长度。这种情况的读取逻辑是从文件中将内容读取到缓冲区中，将缓冲区填满。然后再从缓冲区读区想要的字节内容。

我们看下下面语句的具体执行过程：

```text
p := make([]byte, 2)
n, _ := bufioReader.Read(p)
```

这里是要从文件中读取2个字节。即**要读取的字节数少于缓冲区的内容字节数**。我们看下下图：
![](https://pic2.zhimg.com/80/v2-663a0ad140465a225a6e1d7e2761006d_720w.webp)

第一步是填充缓冲区。因为缓冲区是空的状态，所以先将文件的内容读取到缓冲区。示例中缓冲区的容量为5，所以，会从缓冲区的位置0开始，将缓冲区尽量填满。所谓尽量填满，有以下两种情况：
- 若文件的可读取的内容字节数**大于等于**缓冲区容量，那就先将缓冲区填满。
- 若文件的可读取的内容字节数**小于**缓冲区容量，则将可读取内容全部读取到缓冲区即可。

第二步，从缓冲区中读取数据。示例中，需要拷贝2个字节到调用者的变量 p 中。如下图：
![](https://pic1.zhimg.com/80/v2-bbf419df5d3d88359dc08c4ac0845c0c_720w.webp)

  

第三步，移动 r 的位置，以便标记下次从缓存区读取时的开始位置。这次是从缓冲区索引0的位置开始读取的，并且只读取了2个字节，所以下次读取应该是从缓冲区的索引位置2处开始读取。
![](https://pic3.zhimg.com/80/v2-5d0af622737107ca5602671bafb6d056_720w.webp)
以上就是要读取的字节数比缓冲区的缓存内容字节数少时的实现逻辑。

现在，我们来看这样一种场景，如果调用者要读取的字节数和缓冲区的字节数相等，按照上面的逻辑，其读取过程如下：

- 先从文件读取5个字节到缓冲区，这时 w=5，代表下次再写入缓冲区的位置。缓冲区从空的状态转换到满的状态。
- 然后再将缓冲区的5个字节全部拷贝到 p 中，这时r = 5，代表下次再从缓冲区读取数据的位置。这时缓冲区中的内容都已经被读走了， r 和 w相等。这时缓冲区从满的状态又变成了空的状态。
![](https://pic1.zhimg.com/80/v2-731a9d6325ea352dd0cfa67de20dfbf4_720w.webp)

我们看到，最终缓冲区的状态还是空，数据只不过是在缓冲区中中转了一下而已。因此，**更有效的做法应该是从文件直接读取对应的字节数到 调用者的 p 中就行，而不再经过缓冲区**。因为最终读取的效果是一样的，但直接读取效率还会更高，因为少了一次从缓冲区到 调用者 p 的拷贝操作。如下图所示：

![](https://pic1.zhimg.com/80/v2-1738be17f5a645fadba106b21acb28c8_720w.webp)
同样，如果当要读取的字节数长度大于缓冲区的长度，也是同样的原理。这里就不再过多獒述了。

### 场景二：当缓冲区为非空状态的读取逻辑

如果在缓冲区非空状态下进行读取操作时，唯一需要注意的点就是**当缓冲区中可读取的内容字节数小于调用者要读取的字节数时，则只能读取缓冲区中的内容**。如下是bufio中缓冲区的初始状态，目前只有1字节的内容可读，如下图：

![](https://pic3.zhimg.com/80/v2-500f04a2e5e912ae380f55e1eabf482e_720w.webp)
调用者期望读取2个字节的内容，但缓冲区中只有1个字节可读，这时是不再从文件中继续读取的，而是只读取缓冲区的内容即可，如下图：
![](https://pic1.zhimg.com/80/v2-5ba40605666822b919b0511dccddca18_720w.webp)
最后，将实际读取到的字节数返回给调用者，并将下次可读取的索引位置 r 进行更新，如下图：

![](https://pic3.zhimg.com/80/v2-9240ab05352bf2b71b1eefcaee6c1c6e_720w.webp)

这时缓冲区的状态实际上是变成了空的状态。如果再继续读取的话，r和w就会复位成0，并从文件中再读取一大块内容填充到缓冲区中。

另外还有一种就是缓冲区满的状态下的读取逻辑，这种场景下就结合场景二进行读取即可。以上实际上就是**bufio包中的Read([]byte)函数**的逻辑，按字节读取。其实在实际编程中，我们经常会遇到的是按行读取，更通用一点就是一直读取到指定的字符为止。下面我们来看看这种读取操作的实现。

### 05 从缓冲区中读取到指定位置

这种读取方式是从缓冲区中读取数据，直到遇到指定的字符为止（实际上是指定字符所在的切片索引位置）。按行读取是最常见的场景之一，即一直读取到换行符为止。这种读取方式中也分两种情况：

- 情况1：当缓冲区中包含指定的字符，则从缓冲区中直接返回包含该字符及之前的有效内容。
- 情况2：当缓冲区中没有指定的字符，又分两种情况：
- 2.1 若缓冲区是满的状态，则返回整个缓冲的内容
- 2.2 若缓冲区处于非空状态（也非满的状态），则将缓冲区填满内容，再从该缓冲区中查找是否存在指定的字符。若在缓冲区中能够查找到指定的字符，则返回该指定字符及以前的内容，否则，返回整个缓冲区的内容，即2.1的情况。

情况1比较简单，假设我们在缓冲区中读取内容，直到遇到指定字符E

为止。缓冲区中的状态如下：

![](https://pic4.zhimg.com/80/v2-788850c4e4e95868671311e8f122187b_720w.webp)

缓冲区中的buf[1:5]这段内容中包含字符E，那么直接返回buf[1:2]的内容即可。

  

情况2稍微复杂下。 下面我们稍微拆解下在缓冲区各种状态下第一次未找到对应的字符的情况。

首先我们看当缓冲区处于满的状态下，第一次未找到对应字符的逻辑。如下：

  

![](https://pic2.zhimg.com/80/v2-4b8156b2842fafcb2a9e706ec37bbe95_720w.webp)

若在缓冲区中查找字符B

，发现没有对应的字符，同时又发现**缓冲区状态是满的状态**，所以就直接将缓冲区中的**所有内容**都返回，同时将**缓冲区满的错误**返回给调用者。如下图，则返回给调用者

```text
HELLOGO！
```

，同时返回errors.New("buffer is full")：

![](https://pic3.zhimg.com/80/v2-a8b5f02f7275fffa3a60a39c61c593a6_720w.webp)

其次，我们看下如果缓冲区里有内容，但未满的状态的查找逻辑。假设缓冲区中下次可读的位置在第5个，如下图：

![](https://pic1.zhimg.com/80/v2-de213e0d61ec299399ec4cbb77bf091c_720w.webp)

我们从缓冲区的索引5的位置开始查找字符B

，发现没有找到。但同时也发现缓冲区处于非满的状态，因为从索引0到索引5之间的内容已经被读取走了，所以这段内存相当于处于空闲的状态。 因此，这里会先将缓冲区中5到8之间的内容移动到0到3之间，然后再从文件中读取一段内容填充到缓冲3到8的位置上，最后继续查找。如下图：

![](https://pic3.zhimg.com/80/v2-cc1606dc348f559a41b31affb728dbc6_720w.webp)
上图是移动完内容之后的结果。然后从文件中读取内容填充剩余的缓冲区，如下图：

![](https://pic4.zhimg.com/80/v2-707ed1b78c8fc54188ab7e7134ed9bf3_720w.webp)

这样，缓冲区中又有了新的内容，则会从新的内容中继续查找指定的字符B，如下图：

![](https://pic2.zhimg.com/80/v2-7c81e672e995e6ab40513e6275dd18f9_720w.webp)

这里要注意的是0-3之间的内容不再重复查找，只会从3-8之间查找。这里找到了字符B的位置在buf[3:8]这段内容的位置0处。最后返回buf[0:4]的这段内容，因为之前的内容也是搜索的一部分。如下图：

![](https://pic3.zhimg.com/80/v2-3409099146ab2cd062290cecf4332f56_720w.webp)

以上在缓冲区中移动内容到开始位置，并重新填充内容到缓冲区的过程实际上就是**bufio包中的fill方法**。而整个按指定字符读取的过程是bufio包中的**ReadLine和ReadSlice函数**的对应实现（ReadLine函数调用了ReadSlice函数）。ReadLine函数默认是读取内容，直到遇到第一个换行符\n为止。

我们注意到以上的ReadLine和ReadSlice函数都是在缓冲区中的内容中搜索。只要在缓冲区满的状态下，无论是能否搜索到对应的字符，都会返回。 我们知道，文件内容的大小一般都会远远大于缓冲区的大小，那如果在缓冲区满的状态下没有找到对应的字符，如何继续往下查找呢？

### 06 从全文件中读取到指定位置

这种读取方式是从缓冲区中读取，如果该缓冲区中没有读到指定的字符，那么就将该缓冲区的内容暂存到一个临时区，然后再读取文件将缓冲区填满，再次查找，依次循环，直到读到指定的字符为止或读到文件的末尾，将所有的结果返回给调用者。

假设缓冲区处于满的状态，我们要查找指定的字符

```text
r
```

，第一步先从缓冲区中查找，如下：

![](https://pic3.zhimg.com/80/v2-6c629e423e99d960eae07a59bffb46da_720w.webp)

  

第二步，在缓冲区中未找到指定的字符 r，所以需要将缓冲区中的内容移动到暂存区存储起来，以备后续返回时用，如下图：

![](https://pic2.zhimg.com/80/v2-f1882fe18b7258ae5cde7071cdb5a851_720w.webp)

  

第三步，这时缓冲区实际出于空的状态，然后需要从文件中读取内容再次填充缓冲区，继续查找 是否有 r字符。如下图：

![](https://pic2.zhimg.com/80/v2-d8855fbd02b19f747b8531b1b282b651_720w.webp)

第四步，继续从缓冲区中查找字符字符r，如果找到了，则将暂存区中的内容及缓冲区中r及之前的内容都返回给调用者。如下图：

![](https://pic1.zhimg.com/80/v2-1e3afb89c35cd9634ee4f380556755c4_720w.webp)

  

如果在第四步中依然没找到指定的字符r，那么就会调回第二步，依次循环，直到找到指定的字符或将文件中所有的内容都扫描完为止，最终将暂存区即缓冲区中的内容都返回。

此过程就是bufio中的ReadString函数即对应的collectFragments函数的实现。有兴趣的同学可以查看对应的源码。

### 说在最后

由以上可知，bufio是利用局部性原理，通过将文件的内容预先加载到缓存中，以减少IO的系统调用来提高读取性能的。也就是说当读取数据时，只有在缓冲区中能够命中才能提高读取的性能。好了，以上就是我们分享的读取逻辑。下一篇文章我们继续解析bufio中的写入逻辑的实现。

# Golang 超大文件读取的两个方案
流处理方式
分片处理
去年的面试中我被问到超大文件你怎么处理，这个问题确实当时没多想，回来之后仔细研究和讨论了下这个问题，对大文件读取做了一个分析

比如我们有一个 log 文件，运行了几年，有 100G 之大。按照我们之前的操作可能代码会这样写：

```
func ReadFile(filePath string) []byte{
    content, err := ioutil.ReadFile(filePath)
    if err != nil {
        log.Println("Read error")
    }
    return content
} 
```
上面的代码读取几兆的文件可以，但是如果大于你本身及其内存，那就直接翻车了。因为上面的代码，是把文件所有的内容全部都读取到内存之后返回，几兆的文件，你内存够大可以处理，但是一旦上几百兆的文件，就没那么好处理了。那么，正确的方法有两种，第一个是使用流处理方式代码如下：

```
func ReadFile(filePath string, handle func(string)) error {
    f, err := os.Open(filePath)
    defer f.Close()
    if err != nil {
        return err
    }
    buf := bufio.NewReader(f)

    for {
        line, err := buf.ReadLine("\n")
        line = strings.TrimSpace(line)
        handle(line)
        if err != nil {
            if err == io.EOF{
                return nil
            }
            return err
        }
        return nil
    }
}
```
第二个方案就是分片处理，当读取的是二进制文件，没有换行符的时候，使用下面的方案一样处理大文件

```
func ReadBigFile(fileName string, handle func([]byte)) error {
    f, err := os.Open(fileName)
    if err != nil {
        fmt.Println("can't opened this file")
        return err
    }
    defer f.Close()
    s := make([]byte, 4096)
    for {
        switch nr, err := f.Read(s[:]); true {
        case nr < 0:
            fmt.Fprintf(os.Stderr, "cat: error reading: %s\n", err.Error())
            os.Exit(1)
        case nr == 0: // EOF
            return nil
        case nr > 0:
            handle(s[0:nr])
        }
    }
    return nil
}
```

# 1. 1.4 bufio — 缓存IO

bufio 包实现了缓存IO。它包装了 io.Reader 和 io.Writer 对象，创建了另外的Reader和Writer对象，它们也实现了 io.Reader 和 io.Writer 接口，不过它们是有缓存的。该包同时为文本I/O提供了一些便利操作。

## 1.1. 1.4.1 Reader 类型和方法

bufio.Reader 结构包装了一个 io.Reader 对象，提供缓存功能，同时实现了 io.Reader 接口。

Reader 结构没有任何导出的字段，结构定义如下：

```
    type Reader struct {
        buf          []byte        // 缓存
        rd           io.Reader    // 底层的io.Reader
        // r:从buf中读走的字节（偏移）；w:buf中填充内容的偏移；
        // w - r 是buf中可被读的长度（缓存数据的大小），也是Buffered()方法的返回值
        r, w         int
        err          error        // 读过程中遇到的错误
        lastByte     int        // 最后一次读到的字节（ReadByte/UnreadByte)
        lastRuneSize int        // 最后一次读到的Rune的大小 (ReadRune/UnreadRune)
    }
```

### 1.1.1. 1.4.1.1 实例化

bufio 包提供了两个实例化 bufio.Reader 对象的函数：NewReader 和 NewReaderSize。其中，NewReader 函数是调用 NewReaderSize 函数实现的：

```
    func NewReader(rd io.Reader) *Reader {
        // 默认缓存大小：defaultBufSize=4096
        return NewReaderSize(rd, defaultBufSize)
    }
```

我们看一下NewReaderSize的源码：

```
    func NewReaderSize(rd io.Reader, size int) *Reader {
        // 已经是bufio.Reader类型，且缓存大小不小于 size，则直接返回
        b, ok := rd.(*Reader)
        if ok && len(b.buf) >= size {
            return b
        }
        // 缓存大小不会小于 minReadBufferSize （16字节）
        if size < minReadBufferSize {
            size = minReadBufferSize
        }
        // 构造一个bufio.Reader实例
        return &Reader{
            buf:          make([]byte, size),
            rd:           rd,
            lastByte:     -1,
            lastRuneSize: -1,
        }
    }
```

### 1.1.2. 1.4.1.2 ReadSlice、ReadBytes、ReadString 和 ReadLine 方法

之所以将这几个方法放在一起，是因为他们有着类似的行为。事实上，后三个方法最终都是调用ReadSlice来实现的。所以，我们先来看看ReadSlice方法。(感觉这一段直接看源码较好)

**ReadSlice方法签名**如下：

```
    func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
```

ReadSlice 从输入中读取，直到遇到第一个界定符（delim）为止，返回一个指向缓存中字节的 slice，在下次调用读操作（read）时，这些字节会无效。举例说明：

```
    reader := bufio.NewReader(strings.NewReader("http://studygolang.com. \nIt is the home of gophers"))
    line, _ := reader.ReadSlice('\n')
    fmt.Printf("the line:%s\n", line)
    // 这里可以换上任意的 bufio 的 Read/Write 操作
    n, _ := reader.ReadSlice('\n')
    fmt.Printf("the line:%s\n", line)
    fmt.Println(string(n))
```

输出：

```
    the line:http://studygolang.com. 

    the line:It is the home of gophers
    It is the home of gophers
```

从结果可以看出，第一次ReadSlice的结果（line），在第二次调用读操作后，内容发生了变化。也就是说，ReadSlice 返回的 []byte 是指向 Reader 中的 buffer ，而不是 copy 一份返回。正因为ReadSlice 返回的数据会被下次的 I/O 操作重写，因此许多的客户端会选择使用 ReadBytes 或者 ReadString 来代替。读者可以将上面代码中的 ReadSlice 改为 ReadBytes 或 ReadString ，看看结果有什么不同。

注意，这里的界定符可以是任意的字符，可以将上面代码中的'\n'改为'm'试试。同时，返回的结果是包含界定符本身的，上例中，输出结果有一空行就是'\n'本身(line携带一个'\n',printf又追加了一个'\n')。

如果 ReadSlice 在找到界定符之前遇到了 error ，它就会返回缓存中所有的数据和错误本身（经常是 io.EOF）。如果在找到界定符之前缓存已经满了，ReadSlice 会返回 bufio.ErrBufferFull 错误。当且仅当返回的结果（line）没有以界定符结束的时候，ReadSlice 返回err != nil，也就是说，如果ReadSlice 返回的结果 line 不是以界定符 delim 结尾，那么返回的 er r也一定不等于 nil（可能是bufio.ErrBufferFull或io.EOF）。 例子代码：

```
    reader := bufio.NewReaderSize(strings.NewReader("http://studygolang.com"),16)
    line, err := reader.ReadSlice('\n')
    fmt.Printf("line:%s\terror:%s\n", line, err)
    line, err = reader.ReadSlice('\n')
    fmt.Printf("line:%s\terror:%s\n", line, err)
```

输出：

```
    line:http://studygola    error:bufio: buffer full
    line:ng.com    error:EOF
```

**ReadBytes方法签名**如下：

```
    func (b *Reader) ReadBytes(delim byte) (line []byte, err error)
```

该方法的参数和返回值类型与 ReadSlice 都一样。 ReadBytes 从输入中读取直到遇到界定符（delim）为止，返回的 slice 包含了从当前到界定符的内容 **（包括界定符）**。如果 ReadBytes 在遇到界定符之前就捕获到一个错误，它会返回遇到错误之前已经读取的数据，和这个捕获到的错误（经常是 io.EOF）。跟 ReadSlice 一样，如果 ReadBytes 返回的结果 line 不是以界定符 delim 结尾，那么返回的 err 也一定不等于 nil（可能是bufio.ErrBufferFull 或 io.EOF）。

从这个说明可以看出，ReadBytes和ReadSlice功能和用法都很像，那他们有什么不同呢？

在讲解ReadSlice时说到，它返回的 []byte 是指向 Reader 中的 buffer，而不是 copy 一份返回，也正因为如此，通常我们会使用 ReadBytes 或 ReadString。很显然，ReadBytes 返回的 []byte 不会是指向 Reader 中的 buffer，通过[查看源码](http://docscn.studygolang.com/src/bufio/bufio.go?s=10277:10340#L338)可以证实这一点。

还是上面的例子，我们将 ReadSlice 改为 ReadBytes：

```
    reader := bufio.NewReader(strings.NewReader("http://studygolang.com. \nIt is the home of gophers"))
    line, _ := reader.ReadBytes('\n')
    fmt.Printf("the line:%s\n", line)
    // 这里可以换上任意的 bufio 的 Read/Write 操作
    n, _ := reader.ReadBytes('\n')
    fmt.Printf("the line:%s\n", line)
    fmt.Println(string(n))
```

输出：

```
    the line:http://studygolang.com. 

    the line:http://studygolang.com. 

    It is the home of gophers
```

**ReadString方法**

看一下该方法的源码：

```
    func (b *Reader) ReadString(delim byte) (line string, err error) {
        bytes, err := b.ReadBytes(delim)
        return string(bytes), err
    }
```

它调用了 ReadBytes 方法，并将结果的 []byte 转为 string 类型。

**ReadLine方法签名**如下

```
    func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
```

ReadLine 是一个底层的原始行读取命令。许多调用者或许会使用 ReadBytes('\n') 或者 ReadString('\n') 来代替这个方法。

ReadLine 尝试返回单独的行，不包括行尾的换行符。如果一行大于缓存，isPrefix 会被设置为 true，同时返回该行的开始部分（等于缓存大小的部分）。该行剩余的部分就会在下次调用的时候返回。当下次调用返回该行剩余部分时，isPrefix 将会是 false 。跟 ReadSlice 一样，返回的 line 只是 buffer 的引用，在下次执行IO操作时，line 会无效。可以将 ReadSlice 中的例子该为 ReadLine 试试。

注意，返回值中，要么 line 不是 nil，要么 err 非 nil，两者不会同时非 nil。

ReadLine 返回的文本不会包含行结尾（"\r\n"或者"\n"）。如果输入中没有行尾标识符，不会返回任何指示或者错误。

从上面的讲解中，我们知道，读取一行，通常会选择 ReadBytes 或 ReadString。不过，正常人的思维，应该用 ReadLine，只是不明白为啥 ReadLine 的实现不是通过 ReadBytes，然后清除掉行尾的\n（或\r\n），它现在的实现，用不好会出现意想不到的问题，比如丢数据。个人建议可以这么实现读取一行：

```
    line, err := reader.ReadBytes('\n')
    line = bytes.TrimRight(line, "\r\n")
```

这样既读取了一行，也去掉了行尾结束符（当然，如果你希望留下行尾结束符，只用ReadBytes即可）。

### 1.1.3. 1.4.1.3 Peek 方法

从方法的名称可以猜到，该方法只是“窥探”一下 Reader 中没有读取的 n 个字节。好比栈数据结构中的取栈顶元素，但不出栈。

方法的签名如下：

```
    func (b *Reader) Peek(n int) ([]byte, error)
```

同上面介绍的 ReadSlice一样，返回的 []byte 只是 buffer 中的引用，在下次IO操作后会无效，可见该方法（以及ReadSlice这样的，返回buffer引用的方法）对多 goroutine 是不安全的，也就是在多并发环境下，不能依赖其结果。

我们通过例子来证明一下：

```
    package main

    import (
        "bufio"
        "fmt"
        "strings"
        "time"
    )

    func main() {
        reader := bufio.NewReaderSize(strings.NewReader("http://studygolang.com.\t It is the home of gophers"), 14)
        go Peek(reader)
        go reader.ReadBytes('\t')
        time.Sleep(1e8)
    }

    func Peek(reader *bufio.Reader) {
        line, _ := reader.Peek(14)
        fmt.Printf("%s\n", line)
        // time.Sleep(1)
        fmt.Printf("%s\n", line)
    }
```

输出：

```
    http://studygo
    http://studygo
```

输出结果和预期的一致。然而，这是由于目前的 goroutine 调度方式导致的结果。如果我们将例子中注释掉的 time.Sleep(1) 取消注释（这样调度其他 goroutine 执行），再次运行，得到的结果为：

```
    http://studygo
    ng.com.     It is
```

另外，Reader 的 Peek 方法如果返回的 []byte 长度小于 n，这时返回的 `err != nil` ，用于解释为啥会小于 n。如果 n 大于 reader 的 buffer 长度，err 会是 ErrBufferFull。

### 1.1.4. 1.4.1.4 其他方法

Reader 的其他方法都是实现了 io 包中的接口，它们的使用方法在io包中都有介绍，在此不赘述。

这些方法包括：

```
    func (b *Reader) Read(p []byte) (n int, err error)
    func (b *Reader) ReadByte() (c byte, err error)
    func (b *Reader) ReadRune() (r rune, size int, err error)
    func (b *Reader) UnreadByte() error
    func (b *Reader) UnreadRune() error
    func (b *Reader) WriteTo(w io.Writer) (n int64, err error)
```

你应该知道它们都是哪个接口的方法吧。

## 1.2. 1.4.2 Scanner 类型和方法

对于简单的读取一行，在 Reader 类型中，感觉没有让人特别满意的方法。于是，Go1.1增加了一个类型：Scanner。官方关于**Go1.1**增加该类型的说明如下：

> 在 bufio 包中有多种方式获取文本输入，ReadBytes、ReadString 和独特的 ReadLine，对于简单的目的这些都有些过于复杂了。在 Go 1.1 中，添加了一个新类型，Scanner，以便更容易的处理如按行读取输入序列或空格分隔单词等，这类简单的任务。它终结了如输入一个很长的有问题的行这样的输入错误，并且提供了简单的默认行为：基于行的输入，每行都剔除分隔标识。这里的代码展示一次输入一行：
> 
> ```
>     scanner := bufio.NewScanner(os.Stdin)
>     for scanner.Scan() {
>         fmt.Println(scanner.Text()) // Println will add back the final '\n'
>     }
>     if err := scanner.Err(); err != nil {
>         fmt.Fprintln(os.Stderr, "reading standard input:", err)
>     }
> ```
> 
> 输入的行为可以通过一个函数控制，来控制输入的每个部分（参阅 SplitFunc 的文档），但是对于复杂的问题或持续传递错误的，可能还是需要原有接口。

Scanner 类型和 Reader 类型一样，没有任何导出的字段，同时它也包装了一个 io.Reader 对象，但它没有实现 io.Reader 接口。

Scanner 的结构定义如下：

```
type Scanner struct {
    r            io.Reader // The reader provided by the client.
    split        SplitFunc // The function to split the tokens.
    maxTokenSize int       // Maximum size of a token; modified by tests.
    token        []byte    // Last token returned by split.
    buf          []byte    // Buffer used as argument to split.
    start        int       // First non-processed byte in buf.
    end          int       // End of data in buf.
    err          error     // Sticky error.
}
```

这里 split、maxTokenSize 和 token 需要讲解一下。

然而，在讲解之前，需要先讲解 split 字段的类型 SplitFunc。

### 1.2.1. 1.4.2.1 SplitFunc 类型和实例

**SplitFunc 类型定义**如下：

```
    type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

SplitFunc 定义了 用于对输入进行分词的 split 函数的签名。参数 data 是还未处理的数据，atEOF 标识 Reader 是否还有更多数据（是否到了EOF）。返回值 advance 表示从输入中读取的字节数，token 表示下一个结果数据，err 则代表可能的错误。

举例说明一下这里的 token 代表的意思：

```
有数据 "studygolang\tpolaris\tgolangchina"，通过"\t"进行分词，那么会得到三个token，它们的内容分别是：studygolang、polaris 和 golangchina。而 SplitFunc 的功能是：进行分词，并返回未处理的数据中第一个 token。对于这个数据，就是返回 studygolang。
```

如果 data 中没有一个完整的 token，例如，在扫描行（scanning lines）时没有换行符，SplitFunc 会返回(0,nil,nil)通知 Scanner 读取更多数据到 slice 中，然后在这个更大的 slice 中同样的读取点处，从输入中重试读取。如下面要讲解的 split 函数的源码中有这样的代码：

```
    // Request more data.
    return 0, nil, nil
```

如果 `err != nil`，扫描停止，同时该错误会返回。

如果参数 data 为空的 slice，除非 atEOF 为 true，否则该函数永远不会被调用。如果 atEOF 为 true，这时 data 可以非空，这时的数据是没有处理的。

**bufio 包定义的 split 函数，即 SplitFunc 的实例**

在 bufio 包中预定义了一些 split 函数，也就是说，在 Scanner 结构中的 split 字段，可以通过这些预定义的 split 赋值，同时 Scanner 类型的 Split 方法也可以接收这些预定义函数作为参数。所以，我们可以说，这些预定义 split 函数都是 SplitFunc 类型的实例。这些函数包括：ScanBytes、ScanRunes、ScanWords 和 ScanLines。（由于都是 SplitFunc 的实例，自然这些函数的签名都和 SplitFunc 一样）

**ScanBytes** 返回单个字节作为一个 token。

**ScanRunes** 返回单个 UTF-8 编码的 rune 作为一个 token。返回的 rune 序列（token）和 range string类型 返回的序列是等价的，也就是说，对于无效的 UTF-8 编码会解释为 U+FFFD = "\xef\xbf\xbd"。

**ScanWords** 返回通过“空格”分词的单词。如：study golang，调用会返回study。注意，这里的“空格”是 `unicode.IsSpace()`，即包括：'\t', '\n', '\v', '\f', '\r', ' ', U+0085 (NEL), U+00A0 (NBSP)。

**ScanLines** 返回一行文本，不包括行尾的换行符。这里的换行包括了Windows下的"\r\n"和Unix下的"\n"。

一般地，我们不会单独使用这些函数，而是提供给 Scanner 实例使用。现在我们回到 Scanner 的 split、maxTokenSize 和 token 字段上来。

**split 字段**（SplitFunc 类型实例），很显然，代表了当前 Scanner 使用的分词策略，可以使用上面介绍的预定义 SplitFunc 实例赋值，也可以自定义 SplitFunc 实例。（当然，要给 split 字段赋值，必须调用 Scanner 的 Split 方法）

**maxTokenSize 字段** 表示通过 split 分词后的一个 token 允许的最大长度。在该包中定义了一个常量 MaxScanTokenSize = 64 * 1024，这是允许的最大 token 长度（64k）。

**token 字段** 上文已经解释了这个是什么意思。

### 1.2.2. 1.4.2.2 Scanner 的实例化

Scanner 没有导出任何字段，而它需要有外部的 io.Reader 对象，因此，我们不能直接实例化 Scanner 对象，必须通过 bufio 包提供的实例化函数来实例化。实例化函数签名以及内部实现：

```
    func NewScanner(r io.Reader) *Scanner {
        return &Scanner{
            r:            r,
            split:        ScanLines,
            maxTokenSize: MaxScanTokenSize,
            buf:          make([]byte, 4096), // Plausible starting size; needn't be large.
        }
    }
```

可见，返回的 Scanner 实例默认的 split 函数是 ScanLines。

### 1.2.3. 1.4.2.2 Scanner 的方法

**Split 方法** 前面我们提到过可以通过 Split 方法为 Scanner 实例设置分词行为。由于 Scanner 实例的默认 split 总是 ScanLines，如果我们想要用其他的 split，可以通过 Split 方法做到。

比如，我们想要统计一段英文有多少个单词（不排除重复），我们可以这么做：

```
    const input = "This is The Golang Standard Library.\nWelcome you!"
    scanner := bufio.NewScanner(strings.NewReader(input))
    scanner.Split(bufio.ScanWords)
    count := 0
    for scanner.Scan() {
        count++
    }
    if err := scanner.Err(); err != nil {
        fmt.Fprintln(os.Stderr, "reading input:", err)
    }
    fmt.Println(count)
```

输出：

```
    8
```

我们实例化 Scanner 后，通过调用 scanner.Split(bufio.ScanWords) 来更改 split 函数。注意，我们应该在调用 Scan 方法之前调用 Split 方法。

**Scan 方法** 该方法好比 iterator 中的 Next 方法，它用于将 Scanner 获取下一个 token，以便 Bytes 和 Text 方法可用。当扫描停止时，它返回false，这时候，要么是到了输入的末尾要么是遇到了一个错误。注意，当 Scan 返回 false 时，通过 Err 方法可以获取第一个遇到的错误（但如果错误是 io.EOF，Err 方法会返回 nil）。

**Bytes 和 Text 方法** 这两个方法的行为一致，都是返回最近的 token，无非 Bytes 返回的是 []byte，Text 返回的是 string。该方法应该在 Scan 调用后调用，而且，下次调用 Scan 会覆盖这次的 token。比如：

```
    scanner := bufio.NewScanner(strings.NewReader("http://studygolang.com. \nIt is the home of gophers"))
    if scanner.Scan() {
        scanner.Scan()
        fmt.Printf("%s", scanner.Text())
    }
```

返回的是：`It is the home of gophers` 而不是 `http://studygolang.com.`

**Err 方法** 前面已经提到，通过 Err 方法可以获取第一个遇到的错误（但如果错误是 io.EOF，Err 方法会返回 nil）。

下面，我们通过一个完整的示例来演示 Scanner 类型的使用。

### 1.2.4. 1.4.2.3 Scanner 使用示例

我们经常会有这样的需求：读取文件中的数据，一次读取一行。在学习了 Reader 类型，我们可以使用它的 ReadBytes 或 ReadString来实现，甚至使用 ReadLine 来实现。然而，在 Go1.1 中，我们可以使用 Scanner 来做这件事，而且更简单好用。

```
    file, err := os.Create("scanner.txt")
    if err != nil {
        panic(err)
    }
    defer file.Close()
    file.WriteString("http://studygolang.com.\nIt is the home of gophers.\nIf you are studying golang, welcome you!")
    // 将文件 offset 设置到文件开头
    file.Seek(0, os.SEEK_SET)
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        fmt.Println(scanner.Text())
    }
```

输出结果：

```
    http://studygolang.com.
    It is the home of gophers.
    If you are studying golang, welcome you!
```

## 1.3. 1.4.3 Writer 类型和方法

bufio.Writer 结构包装了一个 io.Writer 对象，提供缓存功能，同时实现了 io.Writer 接口。

Writer 结构没有任何导出的字段，结构定义如下：

```
    type Writer struct {
        err error        // 写过程中遇到的错误
        buf []byte        // 缓存
        n   int            // 当前缓存中的字节数
        wr  io.Writer    // 底层的 io.Writer 对象
    }
```

相比 bufio.Reader, bufio.Writer 结构定义简单很多。

注意：如果在写数据到 Writer 的时候出现了一个错误，不会再允许有数据被写进来了，并且所有随后的写操作都会返回该错误。

### 1.3.1. 1.4.3.1 实例化

和 Reader 类型一样，bufio 包提供了两个实例化 bufio.Writer 对象的函数：NewWriter 和 NewWriterSize。其中，NewWriter 函数是调用 NewWriterSize 函数实现的：

```
    func NewWriter(wr io.Writer) *Writer {
        // 默认缓存大小：defaultBufSize=4096
        return NewWriterSize(wr, defaultBufSize)
    }
```

我们看一下 NewWriterSize 的源码：

```
    func NewWriterSize(wr io.Writer, size int) *Writer {
        // 已经是 bufio.Writer 类型，且缓存大小不小于 size，则直接返回
        b, ok := wr.(*Writer)
        if ok && len(b.buf) >= size {
            return b
        }
        if size <= 0 {
            size = defaultBufSize
        }
        return &Writer{
            buf: make([]byte, size),
            wr:  w,
        }
    }
```

### 1.3.2. 1.4.3.2 Available 和 Buffered 方法

Available 方法获取缓存中还未使用的字节数（缓存大小 - 字段 n 的值）；Buffered 方法获取写入当前缓存中的字节数（字段 n 的值）

### 1.3.3. 1.4.3.3 Flush 方法

该方法将缓存中的所有数据写入底层的 io.Writer 对象中。使用 bufio.Writer 时，在所有的 Write 操作完成之后，应该调用 Flush 方法使得缓存都写入 io.Writer 对象中。

### 1.3.4. 1.4.3.4 其他方法

Writer 类型其他方法是一些实际的写方法：

```
    // 实现了 io.ReaderFrom 接口
    func (b *Writer) ReadFrom(r io.Reader) (n int64, err error)

    // 实现了 io.Writer 接口
    func (b *Writer) Write(p []byte) (nn int, err error)

    // 实现了 io.ByteWriter 接口
    func (b *Writer) WriteByte(c byte) error

    // io 中没有该方法的接口，它用于写入单个 Unicode 码点，返回写入的字节数（码点占用的字节），内部实现会根据当前 rune 的范围调用 WriteByte 或 WriteString
    func (b *Writer) WriteRune(r rune) (size int, err error)

    // 写入字符串，如果返回写入的字节数比 len(s) 小，返回的error会解释原因
    func (b *Writer) WriteString(s string) (int, error)
```

这些写方法在缓存满了时会调用 Flush 方法。另外，这些写方法源码开始处，有这样的代码：

```
    if b.err != nil {
        return b.err
    }
```

也就是说，只要写的过程中遇到了错误，再次调用写操作会直接返回该错误。

## 1.4. 1.4.4 ReadWriter 类型和实例化

ReadWriter 结构存储了 bufio.Reader 和 bufio.Writer 类型的指针（内嵌），它实现了 io.ReadWriter 结构。

```
    type ReadWriter struct {
        *Reader
        *Writer
    }
```

ReadWriter 的实例化可以跟普通结构类型一样，也可以通过调用 bufio.NewReadWriter 函数来实现：只是简单的实例化 ReadWriter

```
    func NewReadWriter(r *Reader, w *Writer) *ReadWriter {
        return &ReadWriter{r, w}
    }
```

# Reference
https://juejin.cn/post/6864886461746855949#heading-0
https://zhuanlan.zhihu.com/p/517126765
http://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter01/01.4.html
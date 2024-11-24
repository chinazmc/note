
# 零拷贝技术第一篇：综述

**零拷贝**(zero copy)在一些语境下指代的意思有所不同,本文讲的零拷贝就是大家常说的，通过这个技术让CPU释放出来不去执行内存中数据拷贝的功能，或者避免不必要的拷贝，所以说零拷贝不是没有数据的拷贝(复制)，而是广义上讲的减少和避免不必要的数据拷贝,可以用来节省CPU使用和内带宽等，比如通过网络高速传输文件、实现网络proxy等等，零拷技术可以极大的提高程序的性能。

本文总结零拷贝的各种技术，下一篇介绍常见的零拷贝技术在Go语言中的应用。

## 零拷贝技术

其实，零拷贝很久以来都被用在提升程序的性能上，比如nginx、kafka等，而且很多文章也详细介绍了零拷贝就要解决的问题，我在这里还是在总结一下，如果你已经了解了零拷贝的计数，不妨回顾一下。

我们来分析一个从网络读取文件的场景。服务器从磁盘读取一个文件，并写入到socket中返回给客户端。我们看看服务端的数据拷贝情况：  
[![](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/readwrite.png)](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/readwrite.png)

程序开始使用系统调用[read](https://man7.org/linux/man-pages/man2/read.2.html)告诉操作系统要从磁盘文件中读取数据，它首先从用户态切换到内核态，这个切换是有花费的，操作系统需要保存用户态的状态，一些寄存器的地址等，等read系统调用完成后返回，程序又需要从内核态切换到用户态，把保存的用户态的状态恢复，所以一次系统调用需要两次的用户态/内核态的切换。同样，把文件的内容写入到socket的时候，程序调用[write](https://man7.org/linux/man-pages/man2/write.2.html)系统调用，又进行了两次用户态/内核态的切换。

从操作的数据来看，这个数据还被拷贝了四次。在read系统调用的时候，DMA方式从磁盘拷贝到内核缓冲区，又通过CPU拷贝从内核缓冲区拷贝到用户的程序缓冲区，这里发生了两次拷贝。在写入socket的时候，数据先从用户程序缓冲区写入到socket缓冲区，又通过DMA方式从socket缓冲区写入到网卡。数据拷贝也发生了四次。

> DMA(Direct Memory Access，直接存储器访问) 是计算机科学中的一种内存访问技术。它允许某些电脑内部的硬件子系统（电脑外设），可以独立地直接读写系统内存，允许不同速度的硬件设备来沟通，而不需要依于中央处理器的大量中断负载。

你可以看到，传统的IO读写方式，包括了四次用户态/内核态的上下文切换，四次数据的拷贝，对性能的影响还是挺大的。广义的零拷贝的技术，就是要尽量减少用户态/内核态的上下文切换，以及数据的拷贝次数，为此操作系统也提供了几种方法。

### mmap + write

通过mmap系统调用，将用户空间的虚拟地址和内核空间的虚拟地址映射成同一个物理地址这样可以减少内核空间和内核空间的数据拷贝。

[![](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/mmap.png)](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/mmap.png)

通过mmap系统调用发起IO读取，DMA将磁盘数据写入到内核缓冲区，此时mmap系统调用就返回了。程序调用write系统调用，CPU将内核缓冲区的数据写入到socket缓冲区，DMA又将数据从socket缓冲区谢瑞到网卡。

可以看到，mmap+write方式有两次系统调用，发生四次用户态/内核态的切换，三次数据拷贝。

相对传统的IO方式，减少了一次数据拷贝，但是应该还有优化的空间。

### sendfile

[sendfile](https://man7.org/linux/man-pages/man2/sendfile.2.html)是Linux2.1内核版本后引入的一个系统调用函数,用来优化数据传输。它可以在文件描述符之间传递数据，因为都是在内核之间传递数据，所以非常高效。  
Linux 2.6.33之前目的文件描述符必须是文件，以后的版本就没有限制了，可以是任意的文件。

但是源文件描述符要求必须是支持[mmap](https://man7.org/linux/man-pages/man2/mmap.2.html)操作的文件描述符，普通的文件可以，但是socket就不行了。所以sendfile适合从文件读取数据写socket场景，所以**sendfile**这个名字还是很贴切的，发送文件。

[![](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/sendfile.png)](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/sendfile.png)

用户调用sendfile系统调用，数据通过DMA拷贝到内核缓冲区，CPU将数据从内核缓冲区再写入到socket缓冲区，DMA将socket缓冲区数据写入到网卡，然后sendfile系统调用返回。

可以看到，这里只有一次系统调用，也就是两次用户态/内核态的切换，三次数据拷贝。

相对来说，这种方式对性能已经有所提升。

linux 2.4之后，又对sendfile做了优化，对于支持 dms scatter/gather功能的网卡，只把关于数据的位置和长度的信息的描述符被追加到了socket缓冲区中。DMA引擎直接把数据从内核缓冲区传输到网卡(protocol engine），从而消除了仅有的一次CPU拷贝。

[![](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/sg-dma.png)](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/sg-dma.png)

### splice、tee、vmsplice

sendfile性能虽好，但是还是有些场景下是不能使用的，比如我们想做一个socket proxy,源和目的都是socket,就不能直接使用sendfile了。这个时候我们可以考虑[splice](https://man7.org/linux/man-pages/man2/splice.2.html)。

Linux 2.6.30版本之前，源和目的只能有一个是管道(pipe), 自2.6.31开始, 源和目的只要保证有一个是就行。

[![](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/splice.png)](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/splice.png)

当然，如果我们处理的源和目的不是管道的话，我们可以先建立一个管道，这样就可以使用splice系统调用来实现零拷贝了。

但是，如果每次都创建一个管道，你会发现每次都会多一次系统调用，也就是两次用户态/内核态的切换，所以你如果频繁的拷贝数据，那么可以建立一个管道池就像潘建给Go的标准库提供的一个补丁一样，利用pipe pool对Go语言中的splice做了优化。

tee系统调用用来在两个管道中拷贝数据。  
vmsplice系统调用pipe指向的内核缓冲区和用户程序的缓冲区之间的数据拷贝。

### MSG_ZEROCOPY

Linux v4.14 版本接受了在TCP send系统调用中实现的支持零拷贝([MSG_ZEROCOPY](https://www.kernel.org/doc/html/v4.17/networking/msg_zerocopy.html))的patch，通过这个patch，用户进程就能够把用户缓冲区的数据通过零拷贝的方式经过内核空间发送到网络套接字中去，在5.0中支持UDP。Willem de Bruijn 在他的论文里给出的压测数据是：采用 netperf 大包发送测试，性能提升 39%，而线上环境的数据发送性能则提升了 5%~8%，官方文档陈述说这个特性通常只在发送 10KB 左右大包的场景下才会有显著的性能提升。一开始这个特性只支持 TCP，到内核 v5.0 版本之后才支持 UDP。这里也有一篇官方文档介绍:[Zero-copy networking](https://lwn.net/Articles/726917/)

首先你需要设置socket选项:

```
if (setsockopt(fd, SOL_SOCKET, SO_ZEROCOPY, &one, sizeof(one)))

        error(1, errno, "setsockopt zerocopy");
```

然后调用send系统调用是传入`MSG_ZEROCOPY`参数:

```
ret = send(fd, buf, sizeof(buf), MSG_ZEROCOPY);
```

这里我们传入了buf,但是啥时候buf可以重用呢？这个内核会通知程序进程。它将完成通知放在socket error队列中，所以你需要读取这个队列，知道拷贝啥时候完成buf可释放或者重用了:

```go
pfd.fd = fd;

pfd.events = 0;

if (poll(&pfd, 1, -1) != 1 || pfd.revents & POLLERR == 0)

        error(1, errno, "poll");

ret = recvmsg(fd, &msg, MSG_ERRQUEUE);

if (ret == -1)

        error(1, errno, "recvmsg");

read_notification(msg);
```

因为它可能异步发送数据，你需要检查buf啥时候释放，增加代码复杂度，以及会导致多次用户态和内核态的上下文切换；

Linux 4.18中也支持的receive MSG_ZEROCOPY机制([Zero-copy TCP receive](https://lwn.net/Articles/752188/)).

字节跳动的同学2021年10曾写过文章，通过修改内核的方式兼容先前的send调用方式。这毕竟是特殊的优化，不适合大众的使用方式，所以这个零拷贝的方式还是只在一些特殊的场景下进行优化：

> 字节跳动框架组和字节跳动内核组合作，由内核组提供了同步的接口：当调用 sendmsg 的时候，内核会监听并拦截内核原先给业务的回调，并且在回调完成后才会让 sendmsg 返回。 这使得我们无需更改原有模型，可以很方便地接入 ZeroCopy send。同时，字节跳动内核组还实现了基于 unix domain socket 的 ZeroCopy，可以使得业务进程与 Mesh sidecar 之间的通信也达到零拷贝。
> 
> [字节跳动在 Go 网络库上的实践](https://www.cloudwego.io/zh/blog/2021/10/09/%E5%AD%97%E8%8A%82%E8%B7%B3%E5%8A%A8%E5%9C%A8-go-%E7%BD%91%E7%BB%9C%E5%BA%93%E4%B8%8A%E7%9A%84%E5%AE%9E%E8%B7%B5/)

### copy_file_range

Linux 4.5 增加了一个新的API: [copy_file_range](https://man7.org/linux/man-pages/man2/copy_file_range.2.html), 它在内核态进行文件的拷贝，不再切换用户空间，所以会比cp少块一些，在一些场景下会提升性能。  
[![](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/copy_file_range.png)](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/copy_file_range.png)

## 其它

[AF_XDP](https://lwn.net/Articles/750845/)是Linux 4.18新增加的功能，以前称为AF_PACKETv4（从未包含在主线内核中），是一个针对高性能数据包处理优化的原始套接字，并允许内核和应用程序之间的零拷贝。由于套接字可用于接收和发送，因此它仅支持用户空间中的高性能网络应用。

当然零拷贝技术和数据拷贝的优化一直是大家追求性能优化的方式之一，相关技术也在不断研究之中，欢迎在原文的评论中写出你的看法。


# 零拷贝技术第二篇：Go语言中的应用

书接上回:[零拷贝技术第一篇：综述](https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/), 我们留了一个小尾巴，还没有介绍Go语言中零拷贝技术的应用，那么本文将带你了解Go标准库中零拷贝技术。

## Go标准库中的零拷贝

在Go标准库中，也广泛使用了零拷贝技术来提高性能。因为零拷贝相关的技术很多都是通过系统调用提供的，所以在Go标准库中，也封装了这些系统调用，相关封装的代码可以在[internal/poll](https://github.com/golang/go/tree/600db8a514600df0d3a11edc220ed7e2f51ca158/src/internal/poll)找到。

我们以Linux为例，毕竟我们大部分的业务都是在Linux运行的。

### sendfile

在`internal/poll/sendfile_linux.go`文件中，封装了`sendfile`系统调用，我删除了一部分的代码，这样更容易看到它是如何封装的:

```go
/ SendFile wraps the sendfile system call.

func SendFile(dstFD *FD, src int, remain int64) (int64, error) {

	...... //写锁

	dst := dstFD.Sysfd

	var written int64

	var err error

	for remain > 0 {

		n := maxSendfileSize

		if int64(n) > remain {

			n = int(remain)

		}

		n, err1 := syscall.Sendfile(dst, src, nil, n)

		if n > 0 {

			written += int64(n)

			remain -= int64(n)

		} else if n == 0 && err1 == nil {

			break

		}

		...... // error处理

	}

	return written, err

}
```

可以看到`SendFile`调用senfile批量写入数据。`sendfile`系统调用一次最多会传输 0x7ffff00(2147479552) 字节的数据。这里Go语言设置maxSendfileSize为 0<<20 (4194304)字节。

`net/sendfile_linux.go`文件中会使用到它:
```go
func sendFile(c *netFD, r io.Reader) (written int64, err error, handled bool) {

	var remain int64 = 1 << 62 // by default, copy until EOF

	lr, ok := r.(*io.LimitedReader)

	......

	f, ok := r.(*os.File)

	if !ok {

		return 0, nil, false

	}

	sc, err := f.SyscallConn()

	if err != nil {

		return 0, nil, false

	}

	var werr error

	err = sc.Read(func(fd uintptr) bool {

		written, werr = poll.SendFile(&c.pfd, int(fd), remain)

		return true

	})

	if err == nil {

		err = werr

	}

	if lr != nil {

		lr.N = remain - written

	}

	return written, wrapSyscallError("sendfile", err), written > 0

}
```

这个函数谁又会调用呢？是**TCPConn**。

```go
func (c *TCPConn) readFrom(r io.Reader) (int64, error) {

	if n, err, handled := splice(c.fd, r); handled {

		return n, err

	}

	if n, err, handled := sendFile(c.fd, r); handled {

		return n, err

	}

	return genericReadFrom(c, r)

}
```

这个方法又会被ReadFrom方法封装。 记住这个**ReadFrom**方法，我们待会再说。

```go
func (c *TCPConn) ReadFrom(r io.Reader) (int64, error) {

	if !c.ok() {

		return 0, syscall.EINVAL

	}

	n, err := c.readFrom(r)

	if err != nil && err != io.EOF {

		err = &OpError{Op: "readfrom", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}

	}

	return n, err

}
```

TCPConn.readFrom方法实现很有意思。它首先检查是否满足使用splice系统调用进行零拷贝优化，在目的是TCP connection, 源是TCP或者是Unix connection才能调用splice。  
否则才尝试使用sendfile。如果要使用sendfile优化，也有限制，要求源是\*os.File文件。  
再否则使用不同的拷贝方式。

ReadFrom又会在什么情况下被调用？实际上你经常会用到，`io.Copy`就会调用`ReadFrom`。也许在不经意之间，当你在将文件写入到socket过程中，就不经意使用到了零拷贝。当然这不是唯一的调用和被使用的方式。

如果我们看一个调用链，就会把脉络弄清楚：`io.Copy` -> `*TCPConn.ReadFrom` -> `*TCPConn.readFrom` -> `net.sendFile` -> `poll.sendFile`。

### splice

上面你也看到了，`*TCPConn.readFrom`初始就是尝试使用splice,使用的场景和限制也提到了。  
`net.splice`函数其实是调用`poll.Splice`:

```go
func Splice(dst, src *FD, remain int64) (written int64, handled bool, sc string, err error) {

	p, sc, err := getPipe()

	if err != nil {

		return 0, false, sc, err

	}

	defer putPipe(p)

	var inPipe, n int

	for err == nil && remain > 0 {

		max := maxSpliceSize

		if int64(max) > remain {

			max = int(remain)

		}

		inPipe, err = spliceDrain(p.wfd, src, max)

		handled = handled || (err != syscall.EINVAL)

		if err != nil || inPipe == 0 {

			break

		}

		p.data += inPipe

		n, err = splicePump(dst, p.rfd, inPipe)

		if n > 0 {

			written += int64(n)

			remain -= int64(n)

			p.data -= n

		}

	}

	if err != nil {

		return written, handled, "splice", err

	}

	return written, true, "", nil

}
```

在上一篇中讲到pipe如果每次都创建其实挺损耗性能的，所以这里使用了pip pool,也提到是潘少优化的。

所以你看到，不经意间你就会用到splice或者sendfile。

### CopyFileRange

[copy_file_range_linux.go](https://github.com/golang/go/blob/600db8a514600df0d3a11edc220ed7e2f51ca158/src/internal/poll/copy_file_range_linux.go)封装了copy_file_range系统调用。因为这个系统调用非常的新，所以封装的时候首先要检查Linux的版本，看看是否支持此系统调用。 版本检查和调用批量拷贝的代码我们略过，具体看是怎么使用这个系统调用的：

```go
func copyFileRange(dst, src *FD, max int) (written int64, err error) {

	if err := dst.writeLock(); err != nil {

		return 0, err

	}

	defer dst.writeUnlock()

	if err := src.readLock(); err != nil {

		return 0, err

	}

	defer src.readUnlock()

	var n int

	for {

		n, err = unix.CopyFileRange(src.Sysfd, nil, dst.Sysfd, nil, max, 0)

		if err != syscall.EINTR {

			break

		}

	}

	return int64(n), err

}
```

哪里会使用到它呢？of.File的读取数据的时候：
```go
var pollCopyFileRange = poll.CopyFileRange

func (f *File) readFrom(r io.Reader) (written int64, handled bool, err error) {

	// copy_file_range(2) does not support destinations opened with

	// O_APPEND, so don't even try.

	if f.appendMode {

		return 0, false, nil

	}

	remain := int64(1 << 62)

	lr, ok := r.(*io.LimitedReader)

	if ok {

		remain, r = lr.N, lr.R

		if remain <= 0 {

			return 0, true, nil

		}

	}

	src, ok := r.(*File)

	if !ok {

		return 0, false, nil

	}

	if src.checkValid("ReadFrom") != nil {

		// Avoid returning the error as we report handled as false,

		// leave further error handling as the responsibility of the caller.

		return 0, false, nil

	}

	written, handled, err = pollCopyFileRange(&f.pfd, &src.pfd, remain)

	if lr != nil {

		lr.N -= written

	}

	return written, handled, NewSyscallError("copy_file_range", err)

}
```

同样的是\*FIle.ReadFrom调用：

```go
func (f *File) ReadFrom(r io.Reader) (n int64, err error) {

	if err := f.checkValid("write"); err != nil {

		return 0, err

	}

	n, handled, e := f.readFrom(r)

	if !handled {

		return genericReadFrom(f, r) // without wrapping

	}

	return n, f.wrapErr("write", e)

}
```

所以这个优化用在文件的拷贝中，一般的调用链路是 `io.Copy` -> `*File.ReadFrom` -> `*File.readFrom` -> `poll.CopyFileRange` -> `poll.copyFileRange`

### 标准库零拷贝的应用

Go标准库将零拷贝技术在底层做了封装，所以很多时候你是不知道的。比如你实现了一个简单的文件服务器：

main

```go
import "net/http"

func main() {

	// 绑定一个handler

	http.Handle("/", http.StripPrefix("/static/", http.FileServer(http.Dir("../root.img"))))

	// 监听服务

	http.ListenAndServe(":8972", nil)

}
```

调用链如左：`http.FileServer` -> `*fileHandler.ServeHTTP` -> `http.serveFile` -> `http.serveContent` -> `io.CopyN` -> `io.Copy` -> 和sendFile的调用链接上了。  
可以看到访问文件的时候是调用了sendFile。

## 第三方库

有几个库提供了sendFile/splice的封装。

-   [https://github.com/acln0/zerocopy](https://github.com/acln0/zerocopy)
-   [https://github.com/hslam/splice](https://github.com/hslam/splice)
-   [https://github.com/hslam/sendfile](https://github.com/hslam/sendfile)

因为直接调用系统调用很方便，所以很多时候我们可以模仿标准库实现我们自己零拷贝的方法。  
所以个人感觉这些传统的方式没有太多锦上添花的东西可做了，要做的就是新的零拷贝系统接口的封装或者自定义开发。

# 参考文章  
以下文章是我整理的关于零拷贝技术一部分文章，如果你想深入了解零拷贝技术，可以阅读这些更多的文章。

1.  [https://www.zhihu.com/question/35093238?utm_id=0](https://www.zhihu.com/question/35093238?utm_id=0)
2.  [https://strikefreedom.top/archives/pipe-pool-for-splice-in-go](https://strikefreedom.top/archives/pipe-pool-for-splice-in-go)
3.  [https://www.modb.pro/db/212924](https://www.modb.pro/db/212924)
4.  [https://blog.lpflpf.cn/passages/golang-zerocopy/](https://blog.lpflpf.cn/passages/golang-zerocopy/)
5.  [https://medium.com/swlh/linux-zero-copy-using-sendfile-75d2eb56b39b](https://medium.com/swlh/linux-zero-copy-using-sendfile-75d2eb56b39b)
6.  [https://www.cloudwego.io/zh/blog/2021/10/09/%E5%AD%97%E8%8A%82%E8%B7%B3%E5%8A%A8%E5%9C%A8-go-%E7%BD%91%E7%BB%9C%E5%BA%93%E4%B8%8A%E7%9A%84%E5%AE%9E%E8%B7%B5/#zerocopy](https://www.cloudwego.io/zh/blog/2021/10/09/%E5%AD%97%E8%8A%82%E8%B7%B3%E5%8A%A8%E5%9C%A8-go-%E7%BD%91%E7%BB%9C%E5%BA%93%E4%B8%8A%E7%9A%84%E5%AE%9E%E8%B7%B5/#zerocopy)
7.  [https://zhuanlan.zhihu.com/p/360343446](https://zhuanlan.zhihu.com/p/360343446)
8.  [https://blog.devgenius.io/linux-zero-copy-d61d712813fe](https://blog.devgenius.io/linux-zero-copy-d61d712813fe)
9.  [https://www.kernel.org/doc/html/v4.18/networking/msg_zerocopy.html](https://www.kernel.org/doc/html/v4.18/networking/msg_zerocopy.html)
10.  [https://lwn.net/Articles/879724/](https://lwn.net/Articles/879724/)
11.  [https://www.phoronix.com/news/Linux-5.20-IO_uring-ZC-Send](https://www.phoronix.com/news/Linux-5.20-IO_uring-ZC-Send)
12.  [https://en.wikipedia.org/wiki/Zero-copy](https://en.wikipedia.org/wiki/Zero-copy)
13.  [https://aijishu.com/a/1060000000149804](https://aijishu.com/a/1060000000149804)
14.  [https://github.com/golang/go/issues/48530](https://github.com/golang/go/issues/48530)
15.  [https://juejin.cn/post/6863264864140935175](https://juejin.cn/post/6863264864140935175)
16.  [https://www.linuxjournal.com/article/6345](https://www.linuxjournal.com/article/6345)
17.  [https://jishuin.proginn.com/p/763bfbd47570](https://jishuin.proginn.com/p/763bfbd47570)


https://colobu.com/2022/11/19/zero-copy-and-how-to-use-it-in-go/
https://colobu.com/2022/11/21/zero-copy-and-how-to-use-it-in-go-2/
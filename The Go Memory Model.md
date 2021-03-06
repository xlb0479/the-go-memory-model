# Go内存模型

[介绍](#介绍)
[忠告](#忠告)
[先行发生原则](#先行发生原则)
[同步](#同步)
&nbsp;&nbsp;&nbsp;&nbsp;[初始化](#初始化)
&nbsp;&nbsp;&nbsp;&nbsp;[Go协程的创建](#Go协程的创建)
&nbsp;&nbsp;&nbsp;&nbsp;[Go协程的销毁](#Go协程的销毁)
&nbsp;&nbsp;&nbsp;&nbsp;[香奈儿通信](#香奈儿通信)
&nbsp;&nbsp;&nbsp;&nbsp;[锁](#锁)
&nbsp;&nbsp;&nbsp;&nbsp;[Once](#Once)
[错误的同步](#错误的同步)

## 介绍

Go内存模型定义了在什么样的条件下能够保证一个Go协程可以观察到另一个Go协程对同一变量的修改。

## 忠告

如果程序要修改可能被多个Go协程访问的数据，那么就必须要对访问做串行化处理。

要保护数据，做串行化访问，可以用香奈儿或者其他的同步原语，比如[sync](#)和[sync/atomic](#sync/atomic)中提供的那些玩意儿。

如果你想通过本文的学习来了解你的程序，那你想多了。

别自作聪明。

（网上查了一下，最后这点翻译的五花八门，我也不确定是不是该这么翻译。）

## 先行发生原则

在一个Go协程内部，读写操作必须要表现得跟代码定义的顺序一致。也就是说编译器和处理器只能在不影响语言规范定义的行为顺序时，才能调整一个Go协程内部的读写操作顺序。因为这种重排序的存在，导致一个Go协程内观察到的执行顺序可能跟另一个Go协程感受到的不一样。比如一个Go协程执行a = 1; b = 2，另一个Go协程可能先观察到b的更新，然后才是a的更新。

为了明确读写行为准则，我们搞了一套*先行发生（happens before）*原则，这是Go程序中关于内存操作执行顺序的部分有序原则。如果事件e<sub>1</sub>先行发生于e<sub>2</sub>，那么e<sub>2</sub>就后行发生于e<sub>1</sub>。并且，如果e<sub>1</sub>对于e<sub>2</sub>既非先行发生也非后行发生，则我们说e<sub>1</sub>和e<sub>2</sub>是并发的。

*在一个Go协程内，先行发生的顺序即程序所表达的顺序。*

当以下条件成立时，对变量v的读操作*r*，*允许*观察到对v的写操作*w*：

1. *r*没有先行发生于*w*。
2. 不存在另一个对v的写操作*w'*，它后行发生于*w*但先行发生于*r*。

要想保证对变量v的读操作*r*观察到一个特定的对v的写操作*w*，就要保证*w*是*r*唯一能够被允许观察到的写操作。也就是说，当以下条件成立时，*r*能够*确保*观察到*w*：

1. *w*先行发生于*r*。
2. 对于共享变量v的其他写操作全都先行发生于*w*或者后行发生于*r*。

这组条件比上一组更严格；它要求不能有其他的写操作和*w*或*r*并发。

在一个Go协程内，不存在并发，此时两种定义是等价的：读操作*r*观察到对变量v最近一次写操作*w*所写入的值。当多个Go协程访问共享变量v，它们必须要基于同步事件来建立起先行发生原则，保证读操作能够观察到所需的写操作。

在内存模型中，对变量v使用对应类型零值进行初始化，被视为一次写操作。

如果读写的值大于一个机器字（word），则以无法预知的顺序进行多次机器字长（machine-word-sized）操作。

## 同步

### 初始化

程序初始化是在单一Go协程内运行的，但这个Go协程也可能创建其他Go协程，它们就并发了。

*如果p包导入了q包，那么q的init函数执行完成先行发生于p的所有函数。*

*main.main函数的执行后行发生于所有init函数结束之后。*

### Go协程的创建

*go语句可以启动一个新的Go协程，它先行发生于Go协程本身的执行。*

比如这样：

```go
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

调用hello就会在之后打印出“hello,world”（也可能在hello返回后才打印）。

### Go协程的销毁

Go协程的退出无法保证先行发生于程序中的任何事件。比如这样：

```go
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

a的赋值没有伴随任何同步事件，因此无法保证被其他Go协程观察到。事实上，一个激进的编译器可能会直接删掉整个Go语句。

如果一个Go协程要使其效果被其他Go协程观察到，那就要使用同步机制，比如锁或者香奈儿通信，来建立一个相对顺序。

### 香奈儿通信

香奈儿通信是Go写成之间主要的同步手段。对香奈儿的每一次发送都会对应发生一次接收，而这通常是分别发生在不同的Go协程中。

*发送数据到香奈儿先行发生于在该香奈儿对应的接收操作完成之前。*

看一下：

```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

这样可以保证一定能够打印出“hello,world”。对a的写入先行发生于对c的发送，而它又先行发生于对应的接受操作完成之前，然后又继续先行发生于print。

*香奈儿的关闭先行发生于关闭导致接收到零值之前。*

上例中，把c<-0改成close(c)可以保证同样的行为顺序。

*从无缓冲香奈儿接收数据先行发生于对应的发送操作完成之前。*

继续栗子（把发送和接收换了一下，并且改用无缓冲香奈儿）：

```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c <- 0
	print(a)
}
```

依然能够保证打印出“hello,world”。对a的写入先行发生于对c的接收，而它又先行发生于对应的发送操作完成之前，然后又继续先行发生于print。

如果香奈儿带缓冲（比如c=make(chan int, 1)）则无法保证打印出“hello,world”。（可能会打印出空字符串，也可能会崩溃，还有可能出现其他情况。）

*对容量为C的香奈儿的第k次读取先行发生于对应的k+C次发送操作完成之前。*

这条规则对上一条带缓冲香奈儿的规则进行了一般化。它允许带缓冲的香奈儿建模出一个计数信号量：香奈儿中元素的数量对应正在使用的元素数量，香奈儿的容量对应最大同时访问数，发送一个数据时要先获取信号量，接收一个数据则释放信号量。这是一种常见的并发限制手段。

下面我们为工作列表中的每个条目启动了一个Go协程，这些Go协程会通过limit香奈儿进行协调，保证同时最多只能有三个Go协程在执行工作函数。

```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

### 锁

sync包提供了两种锁类型，sync.Mutex和sync.RWMutex。

*对任意sync.Mutext或sync.RWMutex类型的变量l，且n<m，对l.Unlock()的n次调用先行发生于对l.Lock()的m次调用返回之前。*

来看下：

```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

它能够确保打印出“hello,world”。第一次调用l.Unlock()（在f里）先行发生于第二次调用l.Lock()（在main里）返回之前，而它又继续先行发生于print。

*对sync.RWMutex类型的变量l的任意次调用l.RLock，都存在一个n，l.RLock的返回后行发生于l.Unlock的n次调用，并且l.RUnlock先行发生于n+1次l.Lock调用。*

### Once

sync包提供了一个Once类型，它可以在多个Go协程出现的时候提供安全的初始化机制。对于特定的f，多个线程都可以执行once.Do(f)，但只有一个可以真正运行f()，其他的调用会被一直阻塞直到f()返回。

*通过once.Do(f)对f进行的一次调用先行发生（返回）于任意once.Do(f)的返回。*

看下面：

```go
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

调用twoprint只会调用一次setup。setup函数会在print调用之前完成。结果就是打印了两遍“hello,world”。

## 错误的同步

值得注意的是，一次读操作r可能会观察到与*r*并发的写操作*w*写入的值。但即便出现这种情况，并不意味之*r*之后发生的读操作会观察到*w*之前发生的写操作。

来看下：

```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

可能出现的情况是g打印出2和0。

这样就打破了一些常用的套路。

双重检查锁定常用于减少同步带来的损耗。比如twoprint可能被错误地写成这样：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

但这样无法保证在doprint中观察到done的写入就一定意味着观察到了a的写入。这个版本可能会（错误地）打印出一个空字符串而非“hello,world”。

另一个错误的用法是繁忙等待（busy waiting），比如：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

同样，无法保证在main中观察到对done的写入时也能观察到对a的写入，所以这个程序可能打印的也是空字符串。更糟糕的是，main可能永远也无法观察到done的写入，因为两个线程之间不存在同步事件。main中的循环可能永不止步。

关于这个事儿，我们再看一个不一样的。

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

即便main观察到了g!=nil并且退出了循环，仍然无法保证它观察到了g.msg的初始值。

对于所有这些问题，解决之道都是一样的：显式同步。
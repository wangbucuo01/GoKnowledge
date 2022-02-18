# Golang并发编程

### 1. 基础理论知识

1. 并发和并行

并发：同一时间段执行多个任务 

并行：同一时刻执行多个任务       一个程序在某一时刻被多个CPU流水线同时处理 

2. 进程 线程和协程

进程Process：应用程序的运行方式，一道程序在一个数据集上的一次运行过程。

每个进程有自己的独立的地址空间，使得进程之间的地址空间相互隔离。

![1644916097267](D:\专业相关资料\研究生资料\Go语言核心编程课程\笔记\img\1.png)

线程：也叫轻量级进程Lightweight Process。是程序执行的最小单元。一个标准的线程：线程ID，当前指令指针PC，寄存器集合，堆栈。一般来说，一个进程可以有多个线程，各个线程之间共享程序的内存空间（包括代码段，数据段，堆等）。

程序的内存空间：5个区

BSS段：用来存放程序中未初始化的全局变量的一块内存区域

数据段：用来存放已初始化的全局变量

代码段：用来存放执行代码的。

堆：存放程序运行过程中（进程运行）动态分配的内存段。大小不固定。当进程调用malloc函数时，新分配的内存被动态添加到堆上。free从堆上删减。

栈：存放局部变量



协程：goroutine 是线程，只不过是轻量级的线程。

**协程为什么是轻量级的？**

**1. go协程调用和切换比线程效率高**

线程是内核对外提供的服务，应用程序通过系统调用让内核启动线程，由内核完成线程的调度和切换。

go协程不依赖操作系统和其提供的线程，用自己实现的GMP模型来实现协程的调度。

go协程也叫用户态线程。协程之间的切换发生在用户态，在用户态没有时钟中断，系统调用等机制，所以效率高。

（用户态线程和系统态线程：系统态也叫内核态。简单来说，内核态运行操作系统程序，操作硬件。用户态运行用户程序。）

（时钟中断：是中断的一种，让CPU停下来，重新调度所有进程。）

2. go协程占用内存少

需要极少的栈内存大约4-5KB。默认线程需要的栈内存1MB。



三者的关系：

进程：拥有自己独立的堆和栈，既不共享堆，也不共享栈，进程由操作系统调度。

线程：拥有自己独立的栈和共享的堆。也有操作系统调度（标准的线程）。

协程：和线程一样，共享堆，不共享栈，**协程由程序员在协程的代码里显示调度**。



### 2. goroutine快速入门

Go中，协程就是goroutine。

用法：在函数前面加go。

有名函数 

func Add(a, b int) int{

​	

}

go Add(1, 2)

匿名函数

go func ()

案例：使用协程输出Hello World

```go
package main

import (
	"fmt"
	"time"
)

func Echo(s string) {
	for i := 0; i < 3; i++ {
		time.Sleep(3)
		fmt.Println(s)
	}
}

func main() {
	go Echo("Hello")
	Echo("Hello World")
}

```



### 3. goroutine要知道的重点

1. go程序在执行过程中，如果主线程结束，不论协程是否执行完毕，都一同结束。

2. 基于1的原则，我们必须知道协程什么时候结束，然后才能让主线程结束，所以引入我们接下来要学习的——管道。

### 4. GMP模型

通常说并发编程，是允许多个任务同时执行，但实际上并不是同一时刻被执行。多线程或多进程是并行的基本条件，但是单线程也可以用协程实现并发。在go中，通过使用go关键字就可以实现并发编程，但是其背后的具体的调度实现比较复杂。这个调度器就是GMP模型。

1. G：协程，goroutine对象

Goroutine，相当于OS的进程控制块，包括执行的函数指令及参数，G保存的任务对象，线程上下文切换，现场保护和现场恢复需要的寄存器等信息。封装在runtime.g。

```go
// Go1.11版本默认stack大小为2KB

_StackMin = 2048
 
// 创建一个g对象,然后放到g队列
// 等待被执行
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
    _g_ := getg()

    _g_.m.locks++
    siz := narg
    siz = (siz + 7) &^ 7

    _p_ := _g_.m.p.ptr()
    newg := gfget(_p_)    
    if newg == nil {        
       // 初始化g stack大小
        newg = malg(_StackMin)
        casgstatus(newg, _Gidle, _Gdead)
        allgadd(newg)
    }    
    // 以下省略}
```

2. M：Machine，线程

程序执行的主线程

3. P：Processor 协程执行的上下文（环境），执行执行的处理器

是一个抽象的概念。当P有任务时，需要创建或者唤醒一个系统线程来执行它队列里的任务。所以P/M进程绑定，构成一个执行单元。

P决定了同时可以并发执行的任务的数量，可通过GOMAXPROCS限制同时执行用户级任务的操作系统线程。

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println(runtime.GOMAXPROCS(-1))    // 8
	fmt.Println(runtime.NumCPU())    // 8
	runtime.GOMAXPROCS(20)
	fmt.Println(runtime.GOMAXPROCS(-1))    // 20
	runtime.GOMAXPROCS(300)
	fmt.Println(runtime.GOMAXPROCS(-1))     // 300
	runtime.GOMAXPROCS(8)
	fmt.Println(runtime.GOMAXPROCS(-1))    // 8
}
```

Go调度器的调度过程

首先创建一个G对象，G对象保存在P的本地队列或全局队列。

P唤醒一个M，P继续执行。

M寻找是否有空闲的P。如果有就将该G对象移动到它本身。

接下来M执行一个调度循环（调用G对象-》执行-》清理线程-》继续寻找新的goroutine执行）

M在执行过程中， 随时会发生上下文切换，当发生上下文切换时，需要对执行现场进行保护，以便下次执行时对现场进行恢复。Go调度器的M的栈保存在G对象上，只需要将M需要的寄存器保存到G对象上就可以实现现场保护。当这些寄存器数据被保存起来，就随时可以做上下文切换了。如果此时G任务还没执行完，M可以将任务重新丢到P的任务队列，等待下一次被调度执行。当再次被调度时，M通过访问G的vdsoSP、vdsoPC寄存器进行现场恢复。



（1）P队列：

本地队列：当前P的队列，没有数据竞争的问题，无需加锁，处理速度快

全局队列：为了保证多个P之间任务的平衡。所有M共享P全局队列，为保证数据竞争问题，需要加锁处理。

（2）上下文切换：协程执行的当前环境

（3）线程清理：goroutine被调度执行必须保证P/M绑定，所以线程清理只需要将P释放就可以实现线程的清理。



![1645108150681](D:\专业相关资料\研究生资料\Go语言核心编程课程\笔记\img\2.png)



### 5. 进程通信的理论知识

进程通信并不是指进程间“传递数据”。

进程共享内存时会发生冲突。

解决方法的核心思想是一致的：当某个进程正在操作共享内存区时，其他进程不得操作共享内存区。

这个思路实现的关键点就是：令其他进程知道有一个进程在操作共享内存区。这个问题就是进程间通信问题。通信的内容是有没有其他进程在操作共享内存区。

1. 锁机制
2. 管道机制

![1645109600521](D:\专业相关资料\研究生资料\Go语言核心编程课程\笔记\img\3.png)



### 6. 管道入门

也叫通道。是一种数据结构，传递数据，用管道来代替共享内存。当一个资源需要在goroutine之间共享，就会在goroutine间架起通道，同步交换数据。同时只能有一个goroutine访问管道进行发送和接收数据。

管道和队列相似，先进先出，保证数据的收发顺序。

管道的特点：

1. 类似于Unix中管道 pipe
2. 先进先出
3. 线程安全的，多个goroutine可以同时访问，不需要加锁
4. channel是有类型的，内部存放的数据只能是对应的类型

```markdown
补充知识：
1. make
是go的内建函数，用来为一些数据类型分配内存和初始化一个对象。只能为slice , map, chan三种类型。
2. make和new的区别
二者都是内存的分配（堆上），make只用于slice，map，chan的初始化（非零值），而new只用于分配内存，并且内存置为0；
make返回引用类型本身；
new返回指向类型的指针。

```

```go
	// 1. 管道的声明
	var chan_name chan int
	// var 管道名 chan 管道内数据的类型

	// 2. 创建管道
	// make
	chan_name = make(chan int)
	fmt.Printf("%T\n", chan_name) // chan int

	chan_name2 := new(chan int)
	fmt.Printf("%T\n", chan_name2) // *chan int

	// 3. 向管道内发送数据
	go func() {
		chan_name <- 1
	}()
	// 发送的数据将持续阻塞直到数据被接收
	// 4. 接收数据
	//data := <- chan_name // 阻塞接收数据
	//fmt.Println(data)
	data, ok := <- chan_name
	fmt.Println(ok)
	fmt.Println(data)
```

1. 当向管道中发送的数据是表达式时，它会先计算出表达式的结果，再送入管道。
2. 管道是没有缓存的，只有在接收方准备好发送才被执行。make(chan int)
3. 管道也可以创建时给他缓存，如果有缓存，并且缓存未满，则发送会被执行。
4. 往一个已经被close的channel中继续发送数据将会报run-time panic。
5. 往空的channel发送数据会一直被阻塞着。

```go
	// 3. 给管道缓存
	chan_name := make(chan int, 3)
	// 表示我们创建了一个带有缓存的管道。
	chan_name <- 1
	data := <- chan_name
	fmt.Println(data) // 1 可以接收到的
```

capacity：容量。代表管道容纳的最多元素的数量， 代表channel的缓存大小。如果设置为0，表示没有缓存，只有sender和receiver同时准备好了才能通信，否则会被阻塞。如果设置了缓存，就可能不发生阻塞，只有buffer满了，发送才会阻塞。

```go
close(chan_name) // 将管道关闭的内建函数
```

### 7. 管道的案例

```go
package main

import "fmt"

func main() {
	// 1. range遍历channel
	var ch chan int
	ch = make(chan int, 10)
	// 循环向管道发送数据
	for i := 0; i < 10; i++ {
		ch <- i
	}
	// 管道关闭
	//close(ch)
	//// 遍历管道 不是从管道接收数据
	//for v := range ch {
	//	fmt.Println(v)
	//}
	// 从管道接收数据
	// data := [10]int{}
	data := []int{} // slice
	for i := 0; i < 10; i++ {
		//data[i] = <- ch
		data = append(data, <- ch)
	}
	fmt.Println(data)
}

```



```go
package main

import "fmt"

func main() {
	// 2. len和cap的使用
	var ch chan int
	ch = make(chan int, 10)
	// 循环向管道发送数据
	for i := 0; i < 8; i++ {
		ch <- i
	}
	// 管道关闭
	close(ch)
	// 看一下管道的长度
	fmt.Println(len(ch)) // 8 len内建函数返回管道内缓存中的元素个数
	fmt.Println(cap(ch)) // 10 cap返回的是缓存区的大小
}


```



```go
package main

import "fmt"

func main() {
	// 3. 现象
	ch := make(chan string)
	go func() {
		// 当程序运行时，协程和主线程一同运行
		fmt.Println("开启一个协程")
		ch <- "hello"
		fmt.Println("退出一个协程")
	}()
	fmt.Println("程序退出")
	// 程序执行完毕，自动结束，但是协程还没有运行完毕，就和主线程一起被销毁了
	// 为了解决上述问题，可以将协程阻塞
	<- ch
	// 执行这行语句，会发生阻塞，直到接收到数据，但数据会被忽略，实际上只是通过管道在goroutine间阻塞收发，从而实现并发同步
}


```



管道的作用：实现协程间的通信和同步

### 8. select

一个控制结构，类似于switch。

```go
package main

import "fmt"

func main()  {
	// switch 选择/条件结构
	// 案例：根据学生成绩输出学生成绩等级
	var score int
	fmt.Scanf("%d", &score)
	switch {
	case score < 60:
		fmt.Println("不及格")
	case score >= 60 && score < 80:
		fmt.Println("及格")
	case score >= 80 && score < 90:
		fmt.Println("良好")
	case score >= 90 && score <= 100:
		fmt.Println("优秀")
	default:
		fmt.Println("输入的成绩有误！")
	}
	
	// switch的第二种用法
	switch score {
	case 100:
		fmt.Println("满分")
	case 90:
		fmt.Println("再接再厉")
	case 80:
		fmt.Println("还要加油啊")
	case 50:
		fmt.Println("罚站1小时")
	}
}
```



select的结构：

```go
select {
    case 通信操作communication clause:
    	语句
   	...
    default:
    	语句
}
```

1. 每一个case必须是通信操作（发送和接收）
2. 有多个case，随机执行一个可运行的case，如果没有case可运行，就会阻塞，直到有case可运行。默认子句总是可运行的。
3. 通信操作里面有表达式即对管道发送的数据是表达式，会先求值
4. 当某个任意的case被执行，其他的就会被忽略。
5. 如果多个case都可以执行的话，select会随机公平的选一个case执行。否则：如果有default，就执行该语句；若没有，select将被阻塞，直到某个通信可以运行。

```go
package main

import "fmt"

func main()  {
	var c1, c2, c3 chan int
	var i1, i2 int
	c1 = make(chan int)
	c2 = make(chan int)
	c3 = make(chan int)
	go func() {
		c1 <- 1
		c3 <- 3
	}()
	i2 = 2
	select {
	case i1 = <- c1:
		fmt.Println(i1, "from c1")
	case c2 <- i2:
		fmt.Println(i2, "to c2")
	case i3, ok := <- c3:
		if ok {
			fmt.Println(i3, "from c3")
		} else {
			fmt.Println("c3通道被关闭了")
		}
	default:
		fmt.Println("no communication")
	}
}
```



select的作用：用于处理异步I/O问题，select默认是阻塞的，只有当监听的通道中发送或接收可以进行时才会运行。

异步IO：相对于同步IO来说的，同步的IO是，当一个IO操作执行时，应用程序必须等待，直到此IO执行完。相反，异步IO操作在后台运行，IO操作和应用程序可以同时执行。

select的一个重要应用就是超时处理。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	c1 := make(chan string, 1)
	go func() {
		time.Sleep(time.Second  * 2)
		c1 <- "result 1"
	}()
	select {
	case res := <-c1 :
		fmt.Println(res)
	case <- time.After(time.Second * 1): // time.After返回一个<-chan Time类型的单向channel
		// 设置select1秒超时
		// 即使当没有c1中读取到数据，也可以从timeout中读取到数据，这样select可以避免永久等待的问题
		fmt.Println("timeout 1")
	}

}
```



### 9. 锁机制 sync包实现并发

#### 9.1 竞态

竞态是指数据争用的现象。

```go
package main

import "fmt"

func getNumber() int {
	var i int
	go func() {
		i = 6
	}()
	return i
}

func main() {
	fmt.Println(getNumber())
}
```

getNumber()开启一个协程设置i的值，主线程返回i的值。现在不知道谁先完成，所以可能出现两种情况：

（1） 协程先完成，i=6

（2） 主线程先完成，协程还未对i进行设置就被销毁，所以i=0

为了避免数据竞态，两种机制：通道阻塞；锁机制（互斥锁）。

#### 9.2 互斥锁

sync.Mutex。适用于读写不确定的场景，即读写次数没有明显的区别，并且只允许有一个读或写的场景。也叫全局锁。

```go
func (m *Mutex) Lock() //Lock方法锁住m，如果m已经加锁，则阻塞直到m解锁。
```

```go
func (m *Mutex) Unlock()
```

Unlock方法解锁m，如果m未加锁会导致运行时错误。锁和线程无关，可以由不同的线程加锁和解锁。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	// 定义锁
	var mutex sync.Mutex
	wait := sync.WaitGroup{} // 是一个结构体，可以等待一组线程的结束

	fmt.Println("Locked")
	mutex.Lock()

	for i := 1; i <= 5; i++ {
		wait.Add(1) // 正数表示阻塞
		go func(i int) {
			fmt.Println("Not Lock", i)
			mutex.Lock()
			fmt.Println("Lock", i)
			time.Sleep(time.Second)
			fmt.Println("Unlock", i)
			mutex.Unlock()
			defer wait.Done()
		}(i)
	}
	// 协程还未全部结束就销毁了
	// WaitGroup用于等待一组线程的结束。
	// 父线程调用Add方法来设定应等待的线程的数量。每个被等待的线程在结束时应调用Done方法。同时，主线程里可以调用Wait方法阻塞至所有线程结束。
	time.Sleep(time.Second)
	fmt.Println("Unlocked")
	mutex.Unlock()
	wait.Wait() // 阻塞主线程，直至所有线程结束
}
```

通过锁机制解决getNumber这个函数的问题：

```go
func main() {
	var i int
	var mutex sync.Mutex
	wait := sync.WaitGroup{}

	mutex.Lock()
	wait.Add(1)
	go func() {
		mutex.Lock()
		i = 6
		mutex.Unlock()
		wait.Done()
	}()
	mutex.Unlock()
	wait.Wait()
	fmt.Println(i)
}
```

通过waitGroup解决：

```go
func main() {
	var i int
	wait := sync.WaitGroup{}
	wait.Add(1)
	go func() {
		i = 6
		wait.Done()
	}()
	wait.Wait()
	fmt.Println(i)
}
```

#### 9.3 sync.WaitGroup

用于等待一组线程的结束。

1. 创建一个waitGroup对象

wait := sync.WaitGroup{}

2. 在开启协程前，为对象内部的计数器+1（即表示要开启一个协程了）.wait.Add(1)
3. 开启协程，协程内部的函数执行完毕前，wait.Done()表示协程执行完毕了，计数器减1.必须要写在协程里面，执行到这一句表示协程执行完毕，可以销毁这个协程了
4. 协程外，主线程中，wait.Wait()函数表示当计数器归0时即所有的协程都执行完毕了，主线程才能结束。用于主线程的阻塞。

#### 9.4 读写互斥锁 sync.RWMutex

是一个控制协程访问的读写锁。该锁可以加多个读锁或一个写锁。其经常用于读次数远远多于写次数的场景。

```go
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
```

可以理解为是Mutex的子类。

写操作：

func (*RWMutex) Lock() / Unlock()

读操作：

func (*RWMutex) RLock() / RUnlock()

原则：

1. 写锁权限远高于读锁，有写锁时优先进行写锁定。
2. 当有读锁时，其他人无法写，但可以读。
3. 有写锁时，其他人既不能写，也不能读。

#### 9.5 sync.Once

Once是只执行一次动作的对象，用于解决一次性初始化的问题。类似于init函数。init函数先于main函数执行，init函数是在文件包首次被加载时才执行。且执行一次。

只执行一次的动作包括：加载一次配置文件，关闭一次通道等。

```go
func (o *Once) Do(f func())
```

Do方法当且仅当第一次被调用时才执行函数f。换句话说，给定变量：

```go
var once Once
```

如果once.Do(f)被多次调用，只有第一次调用会执行f，即使f每次调用Do 提供的f值不同。需要给每个要执行仅一次的函数都建立一个Once类型的实例。

Do用于必须刚好运行一次的初始化。因为f是没有参数的，因此可能需要使用闭包来提供给Do方法调用：

```go
config.once.Do(func() { config.init(filename) })
```

因为只有f返回后Do方法才会返回，f若引起了Do的调用，会导致死锁。



以关闭一次通道为例：

管道如果关闭后在关闭一次，就会造成程序出错。就可以借助sync.Once.Do()方法保证管道在运行过程中只被关闭一次。

```go
package main

import (
	"fmt"
	"sync"
)

var wait sync.WaitGroup // 用于等待一组协程全部结束
var once sync.Once // 用于只执行一次的操作

func func1(ch1 chan int) {
	defer wait.Done()
	for i := 0; i < 10; i++ {
		ch1 <- i
	}
	once.Do(func() {
		close(ch1)
	}) // 闭包的方式
}

func func2(ch1 chan int, ch2 chan int)  {
	defer wait.Done()
	for {
		x, ok := <- ch1
		if !ok {
			break
		}
		ch2 <- 2 * x
	}
	once.Do(func() {
		close(ch2)
	})
}

func main() {
	ch1 := make(chan int, 10)
	ch2 := make(chan int, 10)
	wait.Add(2)
	go func1(ch1)
	go func2(ch1, ch2)
	wait.Wait()
	for i := 0; i < 10; i++ {
		fmt.Println(<-ch2) // 要有数据的接收
	}
}
```



```go
package main

import (
	"fmt"
	"sync"
)

var wait sync.WaitGroup // 用于等待一组协程全部结束
var once sync.Once // 用于只执行一次的操作

func func1(ch1 chan int) {
	defer wait.Done()
	for i := 0; i < 10; i++ {
		ch1 <- i
	}
	once.Do(func() {
		close(ch1)
	}) // 闭包的方式
}

func func2(ch1 chan int, ch2 chan int)  {
	defer wait.Done()
	for {
		x, ok := <- ch1
		if !ok {
			break
		}
		ch2 <- 2 * x
	}
	once.Do(func() {
		close(ch2)
	})
}

func main() {
	ch1 := make(chan int, 10)
	ch2 := make(chan int, 10)
	wait.Add(2)
	go func1(ch1)
	go func2(ch1, ch2)
	wait.Wait()
	//for i := 0; i < 10; i++ {
	//	fmt.Println(<-ch2) // 要有数据的接收
	//}
	close(ch2)
	for i := range ch2 {
		fmt.Println(i)
	}
}
```



#### 9.6 竞态检测器

go run -race main.go










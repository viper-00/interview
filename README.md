# 面试题

# 分类

- [Golang 基础](#Golang基础)

## Golang基础

这里存放一些面试中 Golang 的基础题目。

### = 和 := 的区别？

`=`只有赋值作用，而`:=`除去赋值外，还包含声明的作用。

因此，在变量第一次声明，可以使用`var a = 0`，也可以使用`a := 0`，两者等价（因为是`0`，所以可以省略类型`int`）

在其他赋值的时候，只能使用`=`。

可以近似认为，`:`等价于`var`关键字

### = 指针的作用

记录某个数据存放在哪块内存中。如，`var a int32 = 0`，实际上是申请了一块 4 字节的内存，并置为全空。并使用`&a`来记录这块内存的地址。在任何时候，需要访问这块地址时，就可以使用`a`来获取内容了。

- 对于指针`p`，可以使用`*p`获取该指针对应内存的内容
- 对于变量`a`，可以使用`&a`获取该变量对应的内存地址（指针）

指针对应的可能是变量，也可能是一个函数，或是其他资源。

在实际使用中，可以通过传输一个指针作为变量来实现非引用传参。这样在函数内部可以直接修改外部变量内容（也即实现了修改和访问其他作用域的内容），同时可以避免在函数调用时额外的内存申请与拷贝消耗；但是这种做法可能会在并发执行中引入问题。

### Go 允许多个返回g值吗？

允许，可以使用类似下面的方式返回多个值

```golang
package main

func Sum(a, b int) (c int, isOdd bool) {
    c = a + b
    isOdd = c % 2 == 1
    return 
}

func main() {
    c, isOdd := Sum(a, b)
    fmt.Printf("%d + %d = %d, odd %v", a, b, c, isOdd)
}
```

### Go 有异常类型吗？

Go 没有单独的异常类型，其异常使用`panic()`抛出，可接受的参数为`interface{}`

Go 使用错误和异常两种概念，来对应其他语言中的异常(Exception)，前者使用`error`作为函数的返回值来解决，后者则通过`panic()`、`recover()`、`defer`来实现。

### 什么是协程（Goroutine）

Goroutine 是一种轻量级的线程，其开销很小，通过 goroutine 可以在非常短的时间内切换执行上下文，实现并发执行。在大量 IO 的情况下，这么做可以提升执行效率。

### 如何高效地拼接字符串？

几乎在所有的语言中，字符串拼接都有多种方案，如直接使用`+`，或是使用构造器。

对于大量的字符串拼接，应该使用`strings.Builder`或是`bytes.Buffer`

在 Go 中字符串本身是个不可变的数据结构，具体体现为无法修改指定下标的内容。由于无法修改字符串，下面的代码实际上是无法执行的：

```go
package main

import "fmt"

func main() {
    s := "abc"
    s[1] = 'd'
    fmt.Println(s)
}
```

使用下面的 benchmark 测试可以看到不同拼接方法的具体执行效率

```go
package main

import (
    "bytes"
    "fmt"
    "strings"
    "testing"
)

const (
    sa          = "abcdefg"
    sb          = "1234567"
    concatTimes = 1000
)

func concat1(a, b string) string {
    s := ""
    for i := 0; i < concatTimes; i++ {
        s += a + b
    }
    return s
}

func concat2(a, b string) string {
    s := ""
    for i := 0; i < concatTimes; i++ {
        s = fmt.Sprintf("%s%s%s", s, a, b)
    }
    return s
}

func concat3(a, b string) string {
    var str strings.Builder
    for i := 0; i < concatTimes; i++ {
        str.WriteString(a)
        str.WriteString(b)
    }
    return str.String()
}

func concat4(a, b string) string {
    var str bytes.Buffer
    for i := 0; i < concatTimes; i++ {
        str.WriteString(a)
        str.WriteString(b)
    }
    return str.String()
}
func BenchmarkPlus(b *testing.B) {
    for i := 0; i < b.N; i++ {
        concat1(sa, sb)
    }
}

func BenchmarkPrintf(b *testing.B) {
    for i := 0; i < b.N; i++ {
        concat2(sa, sb)
    }
}

func BenchmarkBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        concat3(sa, sb)
    }
}

func BenchmarkBytes(b *testing.B) {
    for i := 0; i < b.N; i++ {
        concat4(sa, sb)
    }
}
```

```go
$ go test -bench=.
goos: linux
goarch: amd64
BenchmarkPlus-4              465           2520003 ns/op
BenchmarkPrintf-4            397           3015618 ns/op
BenchmarkBuilder-4         36073             31416 ns/op
BenchmarkBytes-4           38354             30013 ns/op
PASS
```

可以看出，拼接速度相差了 100 倍。（如果只是简单的一次 a+b，也即`concatTimes = 1`，那么实际上直接相加更快。）

### 什么是 rune 类型

rune 是 Go 中的字符类型，其等价于 int32（和 C 中的 char 不一样）

具体表现可以看这段代码

```go
package main

import (
    "bytes"
    "fmt"
)

func main() {
    s := "Hello 世界"
    for _, c := range s {
        buf := bytes.NewBuffer([]byte{})
        buf.WriteRune(c)
        fmt.Printf("%T %c %+v %+v\n", c, c, buf.Bytes(), byte(c))
    }
    fmt.Println(len(s), len([]rune(s)))
    fmt.Println([]rune(s))
    fmt.Println([]byte(s))
}
```

其输出如下，可以看出，rune 实际上是一个 int32 类型，对应的是 Unicode。

```go
int32 H [72] 72
int32 e [101] 101
int32 l [108] 108
int32 l [108] 108
int32 o [111] 111
int32   [32] 32
int32 世 [228 184 150] 22
int32 界 [231 149 140] 76
12 8
[72 101 108 108 111 32 19990 30028]
[72 101 108 108 111 32 228 184 150 231 149 140]
```

这里的 19990、30028 是 Unicode 原始字码，非 UTF-8 编码。

对于所有非纯英文操作，如涉及中文，应该先转换为`[]rune`后再处理。

### 如何判断 map 中是否包含某个 key？

直接使用`m[key]`访问对应的内容，共有两个返回值，第一个是对应的 value（如果存在），第二个是是否存在该值

```go
package main

import (
    "fmt"
)

func main() {
    m := map[string]bool{
        "a": true,
        "b": false,
        "c": true,
    }

    _, exists := m["a"]
    fmt.Println(exists)

    _, exists = m["d"]
    fmt.Println(exists)
}
```

### Go 支持默认参数或可选参数吗？

不支持。

虽然可以变相实现，但是并不友好。

```go
package main

import "fmt"

// (a + b) * k
func run(a, b int, args ...map[string]int) int {
    arg := map[string]int{
        "mul": 2,
    }
    if len(args) != 0 {
        for k, v := range args[0] {
            arg[k] = v
        }
    }
    return (a + b) * arg["mul"]
}

func main() {
    fmt.Println(run(2, 3))
    fmt.Println(run(2, 3, map[string]int{"mul": 100}))
}
```

### defer 的执行顺序

当前函数退出时（包括发生异常等各种情况），在 return 语句后执行，因此可以在这里统一处理并修改返回值（比如对 error 进行二次包裹）

如果包含多个`defer`，则会逆序执行，先`defer`的后执行。

### 如何交换 2 个变量的值？

```go
package main

import "fmt"

func main() {
	a := 1
	b := 2
	a, b = b, a
	fmt.Println(a, b)
}
```

### Go 语言 tag 的用处？

标记结构体，如声明该结构体在 json 中应该是哪个字段，在 MySQL 中应该是哪个字段。

### 如何判断 2 个字符串切片（slice) 是相等的？

无脑`reflect.DeepEqual()`即可

要比较两个切片是否相等，有如下几种方法：

1. 使用`reflect.DeepEqual`，从底层比较
2. 使用`for`循环逐个比较
3. 使用`%+v`将其输出成字符串比较

从实现上来说，1 和 3 只需要简单的一行代码，而 2 则需要一段不算太长的代码。

而性能上，可以使用 benchmark 进行比较

```go
package main

import (
	"fmt"
	"math/rand"
	"reflect"
	"testing"
	"time"
)

var (
	l  = 100000
	s1 = generate(l)
	s2 = generate(l)
	s3 = generate(l * 2)
)

var letterRunes = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

func randStringRunes(n int) string {
	rand.Seed(time.Now().UnixNano())
	b := make([]rune, n)
	for i := range b {
		b[i] = letterRunes[rand.Intn(len(letterRunes))]
	}
	return string(b)
}

func generate(n int) []string {
	s := make([]string, n)
	for i := 0; i < n; i++ {
		s[i] = randStringRunes(10)
	}
	return s
}

func cmp1(a, b []string) bool {
	return reflect.DeepEqual(a, b)
}

func cmp2(a, b []string) bool {
	la := len(a)
	lb := len(b)
	if la != lb {
		return false
	}
	for i := 0; i < la; i++ {
		if a[i] != b[i] {
			return false
		}
	}
	return true
}

func cmp3(a, b []string) bool {
	return fmt.Sprintf("%+v", a) == fmt.Sprintf("%+v", b)
}

func BenchmarkReflect(b *testing.B) {
	for i := 0; i < b.N; i++ {
		cmp1(s1, s2)
		cmp1(s1, s3)
		cmp1(s1, s1)
	}
}

func BenchmarkFor(b *testing.B) {
	for i := 0; i < b.N; i++ {
		cmp2(s1, s2)
		cmp2(s1, s3)
		cmp2(s1, s1)
	}
}

func BenchmarkString(b *testing.B) {
	for i := 0; i < b.N; i++ {
		cmp3(s1, s2)
		cmp3(s1, s3)
		cmp3(s1, s1)
	}
}
```

结果如下，可以看出，反射性能远高于另两种

```go
$ go test -bench=.
goos: linux
goarch: amd64
BenchmarkReflect-4        942232              1463 ns/op
BenchmarkFor-4              3004            417853 ns/op
BenchmarkString-4              9         112793444 ns/op
PASS
```

但这个结果是切片很长，而切片中的元素长的情况，如果切片本身不长，但是元素较长呢？

将上面代码修改为`l=100`，`randStringRunes(1000000)`，则新的结果为

```go
go test -bench=.
goos: linux
goarch: amd64
BenchmarkReflect-4        965967              1172 ns/op
BenchmarkFor-4           3006084               434 ns/op
BenchmarkString-4              1        1144034000 ns/op
PASS
```

可以看出，无论如何，使用`%+v`的方案完全可以舍弃，其与另两种方案差距非常之大。

对于切片长度较长，而字符串本身并不长的情况，使用反射可以达到很好的效果。

对于切片长度本身补偿，而字符串本身很长的情况，应该使用`for`循环来进行判断。

不过考虑到性能差距，无脑用反射即可，与目前网络上生成的反射效率低下似乎并不太一致。

### 字符串打印时，%v 和 %+v 的区别

- `%v`: 输出结构体各项的值
- `%+v`: 输出结构体字段和各项值

各个格式化输出占位符含义

| 占位符 | 含义 | # | # 含义 | + | + 含义 |
| --- | --- | --- | --- | --- | --- |
| %v | 输出内容默认格式 | %#v | 输出类型及详细格式 | %+v | 输出内容详细格式（输出结构体字段名） |
| %T | 输出类型 |
| %% | 输出% |
| %t | 输出布尔类型 |
| %b | 输出二进制数字 | %#b | 输出十进制数字（带前导 0b）| %+b | 输出十进制数字（带正负号）|
| %d | 输出十进制数字 | | | %+d | 输出十进制数字（带正负号）|
| %o | 输出八进制数字 | %#o | 输出八进制数字（带前导 0）| %+o | 输出八进制数字（带正负号）|
| %x | 输出十六进制数字（或十六进制表示字符串）| %#x | 输出十六进制数字（带前导 0x）| %+x | 输出十六进制数字（带正负号）|
| %X | 输出大写十六进制数字（或十六进制表示字符串）| %#X | 输出大写十六进制数字（带前导 0X）| %+X | 输出大写十六进制数字（带正负号）|
| %U | 输出 Unicode 编码（格式为U+0000）| %#U | 输出 Unicode 编码及对应字符（带单引号）	|
| %e | 按照科学计数法输出数字 | | | %+e | 按照科学计数法输出（带正负号）|
| %E | 按照科学计数法输出数字（大写 e）| | | %+e | 按照科学计数法输出（大写 E，带正负号）|
| %f | 输出浮点数 | | | %+f | 输出浮点数（带正负号）|
| %g | 根据合适输出%e或%f | | | %+g | 自动选择合适的输出（带正负号）|
| %G | 根据合适输出%E或%f | | | %+G | 自动选择合适的输出（带正负号）|
| %c | 输出数字对应的 Unicode 字符 |
| %s | 输出字符串 |
| %q | 输出字符串（使用引号包裹并转义）| %#q | 输出字符串（使用反引号包裹，需要转义时，按%q输出）| %+q | 输出 Unicode 编码（格式为\u0000）|
| %p | 输出的地址指针（前导为 0x）| %#p | 输出地址指针（不带前导）|

### Go 语言中如何表示枚举值(enums)？

Go 没有默认的枚举值，但可以通过定义结构体实现。

```go
package weekday

//go:generate stringer -type=Weekday

// Weekday 星期
type Weekday byte

const (
	// Monday 星期一
	Monday Weekday = iota + 1
	// Tuesday 星期二
	Tuesday
	// Wedesday 星期三
	Wedesday
	// Thursday 星期四
	Thursday
	// Friday 星期五
	Friday
	// Saturday 星期六
	Saturday
	// Sunday 星期日
	Sunday
)
```

### 空 struct{} 的用途？

空`struct{}`不占据任何空间，因此可以用在所有不需要体现数值本身，但是需要体现存在这个数据的情况。

如：

- 使用`map[string]struct{}`实现集合，这时值本身不会浪费任何空间。
- 在 channel 只需要传输数据，对数据本身并不关心时，可以使用`chan struct{}`，并传输`c <- struct{}{}`。


### init() 函数是什么时候执行的？

当一个编译好的 Go 程序运行时，首先会构建依赖关系，没有任何依赖的包优先处理，按照 常量 → 变量 → 各个`init()`的顺序进行初始化，接着会继续对其他包进行初始化。当所有初始化完成后，执行`main()`

同一个文件内，可以包含多个`init()`，但运行顺序不做保证。

### Go 语言的局部变量分配在栈上还是堆上？

如果编译器检查到该变量可能会在外部使用（如返回了指针），则会分配在堆上，否则分配在栈上。

### 2 个 interface 可以比较吗 ？

可以比较，且在如下情况时两者相等：

两者类型相同，且数值相同（类型相同指类型一致，或为别名）

### 2 个 nil 可能不相等吗？

有可能，要比较两个变量是否相同，实际上如同前面的比较`interface{}`一样，先比较类型再比较内容。

因此，两个 nil 如果是不同类型指针的 nil，那么两者的比较结果是不同的。

不同类型的数据是无法比较的，因此需要使用`interface{}`进行比较（直接给`interface{}`赋值`nil`，是无类型`nil`，与其他有类型的`nil`也不同）

```go
package main

import (
	"fmt"
)

func main() {
	var a, b interface{}
	var c *int = nil
	a = c
	b = c
	fmt.Println(a == b, a == nil, b == nil)
	fmt.Printf("%#v %#v %#v %#v\n", a, b, c, nil)

}
```

上面的代码输出如下：

```go
true false false
(*int)(nil) (*int)(nil) (*int)(nil) <nil>
```

### 函数返回局部变量的指针是否安全?

安全，Go 会对局部变量进行逃逸分析，如果变量作用域超过该函数，则会将其分配在堆上。

### 非接口的任意类型 T() 都能够调用 *T 的方法吗？反过来呢?

`*T`必然可以调用所有`T`和`*T的方法，但是`T`未必可以调用`*T`的方法，只有`T`可寻址时，才可行。

由于将`T`变成`*T`，本质上引入了修改其内容的可能性，因此所有的常量、包级别的函数、`map`元素、字符串中的内容这些“不可变”内容都无法寻址。

### 无缓冲的 channel 和有缓冲的 channel 的区别？

缓冲区可以允许通道内存储缓冲区个数个数据。

如无缓冲的 channel（等价于`make(chan T,0)`），每当发送方发送一条数据时，发送方将会被阻塞，直到接收方取出数据后才会继续运行

而有缓冲 channel 则允许存储多条数据不阻塞发送方。

可以参考下面的代码，修改通道的缓冲长度，查看被阻塞的主函数

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println(1)
	c := make(chan byte, 1)
	fmt.Println(2)
	go func() {
		for {
			time.Sleep(time.Second * 5)
			select {
			case <-c:
				fmt.Println("recv")
			}
		}
	}()
	c <- 0x1
	fmt.Println(3)
	c <- 0x2
	fmt.Println(4)
}
```

### 什么是协程泄露(Goroutine Leak)？

协程大量创建却未释放，导致内存耗尽，程序崩溃，就是协程泄露

除去代码本身的问题外，如果未正确处理通道，也可能导致该问题。如大量协程等待写入通道或等待从通道读出、竞争资源导致死锁、无限循环……

### Go 可以限制运行时操作系统线程的数量吗？

使用`GOMAXPROCS`或`runtime.GOMAXPROCS(num int)`可以限制线程数目

该数值默认为 CPU 的逻辑核心数，同一时间一个核心只能绑定一个线程，运行被调度的的协程。

在 CPU 密集型任务中，若该值过大，则会增加线程切换的开销；

在 I/O 密集型任务中，调大该值则可以提高 I/O 吞吐率。
---
date: 2015-09-18T12:55:08
title: Golang新手开发需要注意的七个细节
slug: 7_things-you-may-not-pay-attation-to-in-go
topics:
    - 编程语言与开发
tags:
    - 技巧
    - 优化
    - Go
disqus_identifier: 100020
---
 
Go以简洁著称，但简洁中不乏值得玩味的细节。这些小细节不如goroutine、interface和channel那样"高大上"，"屌丝"得可能不经常被人注意到，但它们却对理解Go语言有着重要的作用。这里大家一起通过详实的例子来逐一展开和理解一些细节内容。 

<!--more-->


### 一、源文件字符集和字符集编码

Go源码文件默认采用Unicode字符集，Unicode码点(code point)和内存中字节序列（byte sequence）的变换实现使用了UTF-8：一种变长多字节编码，同时也是一种事实字符集编码标准，为Linux、MacOSX 上的默认字符集编码，因此使用Linux或MacOSX进行Go程序开发，你会省去很多字符集转换方面的烦恼。但如果你是在Windows上使用 默认编辑器编辑Go源码文本，当你编译以下代码时会遇到编译错误：

```Go
package main

import "fmt"

func main() {
    fmt.Println("中国人")
}
```
```Shell
$ go build hello.go
# command-line-arguments
hello.go:6 illegal UTF-8 sequence d6 d0
hello.go:6 illegal UTF-8 sequence b9
hello.go:6 illegal UTF-8 sequence fa c8
hello.go:6 illegal UTF-8 sequence cb 22
hello.go:6 newline in string
hello.go:7 syntax error: unexpected }, expected )
```

这是因为Windows默认采用的是CP936字符集编码，也就是GBK编码，“中国人”三个字的内存字节序列为：

```
“d0d6    fab9    cbc8    000a” （通过iconv转换，然后用od -x查看）
```

这个字节序列并__非utf-8__字节序列，Go编译器因此无法识别。要想通过编译，需要将该源文件转换为UTF-8编码格式。

字符集编码对字符和字符串字面值(Literal)影响最大，在Go中对于字符串我们可以有三种写法：

1) 字面值

```Go
var s = "中国人"
```

2) 码点表示法

```Go
var s1 = "\u4e2d\u56fd\u4eba"

//or 

var s2 = "\U00004e2d\U000056fd\U00004eba"
```

3) 字节序列表示法（二进制表示法）
```Go
var s3 = "\xe4\xb8\xad\xe5\x9b\xbd\xe4\xba\xba"
```

这三种表示法中，除字面值转换为字节序列存储时根据编辑器保存的源码文件编码格式之外，其他两种均不受编码格式影响。我们可以通过逐字节输出来查 看字节序列的内容：

```Go
    fmt.Println("s byte sequence:")
    for i := 0; i < len(s); i++ {
        fmt.Printf("0x%x ", s[i])
    }
    fmt.Println("")
```

### 二、续行

良好的代码style一般会要求代码中不能有太long的代码行，否则会影响代码阅读者的体验。在C中有续行符"\"专门用于代码续行处理；但在 Go中没有专属续行符，如何续行需要依据Go的语法规则（参见Go spec）。

Go与C一样，都是以分号(`;`)作为语句结束的标识。不过大多数情况下，分号无需程序员手工输入，而是由编译器自动识别语句结束位置，并插入 分号。因此续行要选择合法的位置。下面代码展示了一些合法的续行位置：(别嫌太丑，这里仅仅是展示合法位置的demo)

```Go
//details-in-go/2/newline.go
… …
var (
    s = "This is an example about code newline," +
        "for string as right value"
    d = 5 + 4 + 7 +
        4
    a = [...]int{5, 6, 7,
        8}
    m = make(map[string]int,
        100)
    c struct {
        m1     string
        m2, m3 int
        m4     *float64
    }

    f func(int,
        float32) (int,
        error)
)

func foo(int, int) (string, error) {
    return "",
        nil
}

func main() {
    if i := d; i >
        100 {
    }

    var sum int
    for i := 0; i < 100; i = i +
        1 {
        sum += i
    }

    foo(1,
        6)

    var i int
    fmt.Printf("%s, %d\n",
        "this is a demo"+
            " of fmt Printf",
        i)
}
```

实际编码中，我们可能经常遇到的是fmt.Printf系列方法中format string太长的情况，但由于Go不支持相邻字符串自动连接(concatenate)，只能通过+来连接fmt字符串，且+必须放在前一行末尾。另外Gofmt工具会自动调整一些不合理的续行处理，主要针对 for, if等控制语句。

### 三、Method Set

Method Set是Go语法中一个重要的隐式概念，在为interface变量做动态类型赋值、embeding struct/interface、type alias、method expression时都会用到Method Set这个重要概念。

1、interface的Method Set

根据Go spec，interface类型的Method Set就是其interface（An interface type specifies a method set called its interface）。
```Go
type I interface {
    Method1()
    Method2()
}
```
I的Method Set包含的就是其literal中的两个方法：Method1和Method2。我们可以通过reflect来获取interface类型的 Method Set：
```Go
//details-in-go/3/interfacemethodset.go
package main

import (
    "fmt"
    "reflect"
)

type I interface {
    Method1()
    Method2()
}

func main() {
    var i *I
    elemType := reflect.TypeOf(i).Elem()
    n := elemType.NumMethod()
    for i := 0; i < n; i++ {
        fmt.Println(elemType.Method(i).Name)
    }
}
```
运行结果：
```Shell
$go run interfacemethodset.go
Method1
Method2
```

2、除interface type外的类型的Method Set

对于非interface type的类型T，其Method Set为所有receiver为T类型的方法组成；而类型*T的Method Set则包含所有receiver为T和*T类型的方法。
```Go
// details-in-go/3/othertypemethodset.go
package main

import "./utils"

type T struct {
}

func (t T) Method1() {
}

func (t *T) Method2() {
}

func (t *T) Method3() {
}

func main() {
    var t T
    utils.DumpMethodSet(&t)

    var pt *T
    utils.DumpMethodSet(&pt)
}
```
我们要dump出T和*T各自的Method Set，运行结果如下：
```Shell
$go run othertypemethodset.go
main.T's method sets:
     Method1

*main.T's method sets:
     Method1
     Method2
     Method3
```

可以看出类型T的Method set仅包含一个receiver类型为T的方法：Method1，而*T的Method Set则包含了T的Method Set以及所有receiver类型为*T的Method。

如果此时我们有一个interface type如下：

```Go
type I interface {
    Method1()
    Method2()
}
```

那下面哪个赋值语句合法呢？合不合法完全依赖于右值类型是否实现了interface type I的所有方法，即右值类型的Method Set是否包含了I的 所有方法。
```Go
var t T
var pt *T

var i I = t

or

var i I = pt
```

编译错误告诉我们：
```Go
     var i I = t // cannot use t (type T) as type I in assignment:
                  T does not implement I (Method2 method has pointer receiver)
```

T的Method Set中只有Method1一个方法，没有实现I接口中的 Method2，因此不能用t赋值给i；而*T实现了I的所有接口，赋值合 法。不过Method set校验仅限于在赋值给interface变量时进行，无论是T还是*T类型的方法集中的方法，对于T或*T类型变量都是可见且可以调用的，如下面代码 都是合法的：
```Go
    pt.Method1()
    t.Method3()
```

因为Go编译器会自动为你的代码做receiver转换：
```Go
    pt.Method1() <=> (*pt).Method1()
    t.Method3() <=> (&t).Method3()
```

很多人纠结于method定义时receiver的类型（T or *T），个人觉得有两点考虑：

+ 效率
    Go方法调用receiver是以传值的形式传入方法中的。如果类型size较大，以value形式传入消耗较大，这时指针类型就是首选。

+ 是否赋值给interface变量、以什么形式赋值
    就像本节所描述的，由于T和*T的Method Set可能不同，我们在设计Method receiver type时需要考虑在interface赋值时通过对Method set的校验。

3、embeding type的Method Set

【interface embeding】

我们先来看看interface类型embeding。例子如下：
```Go
//details-in-go/3/embedinginterface.go
package main

import "./utils"

type I1 interface {
    I1Method1()
    I1Method2()
}
type I2 interface {
    I2Method()
}

type I3 interface {
    I1
    I2
}

func main() {
    utils.DumpMethodSet((*I1)(nil))
    utils.DumpMethodSet((*I2)(nil))
    utils.DumpMethodSet((*I3)(nil))
}
```
```Shell
$go run embedinginterface.go
main.I1's method sets:
     I1Method1
     I1Method2

main.I2's method sets:
     I2Method

main.I3's method sets:
     I1Method1
     I1Method2
     I2Method
```

可以看出嵌入interface type的interface type I3的Method Set包含了被嵌入的interface type：I1和I2的Method Set。很多情况下，我们Go的interface type中仅包含有少量方法，常常仅是一个Method，通过interface type embeding来定义一个新interface，这是Go的一个惯用法，比如我们常用的io包中的Reader, Writer以及ReadWriter接口：
```Go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}
```

【struct embeding interface】

在struct中嵌入interface type后，struct的Method Set中将包含interface的Method Set：
```Go
type T struct {
    I1
}

func (T) Method1() {

}

… …
func main() {
    … …
    var t T
    utils.DumpMethodSet(&t)
    var pt = &T{
        I1: I1Impl{},
    }
    utils.DumpMethodSet(&pt)
}
```

输出结果与预期一致：
```Go
main.T's method sets:
     I1Method1
     I1Method2
     Method1

*main.T's method sets:
     I1Method1
     I1Method2
     Method1
```

【struct embeding struct】

在struct中embeding struct提供了一种“继承”的手段，外部的Struct可以“继承”嵌入struct的所有方法（无论receiver是T还是*T类型）实现，但 Method Set可能会略有不同。看下面例子：
```Go
//details-in-go/3/embedingstructinstruct.go
package main

import "./utils"

type T struct {
}

func (T) InstMethod1OfT() {

}

func (T) InstMethod2OfT() {

}

func (*T) PtrMethodOfT() {

}

type S struct {
}

func (S) InstMethodOfS() {

}

func (*S) PtrMethodOfS() {
}

type C struct {
    T
    *S
}

func main() {
    var c = C{S: &S{}}
    utils.DumpMethodSet(&c)
    var pc = &C{S: &S{}}
    utils.DumpMethodSet(&pc)

    c.InstMethod1OfT()
    c.PtrMethodOfT()
    c.InstMethodOfS()
    c.PtrMethodOfS()
    pc.InstMethod1OfT()
    pc.PtrMethodOfT()
    pc.InstMethodOfS()
    pc.PtrMethodOfS()
}
```
```Shell
$go run embedingstructinstruct.go
main.C's method sets:
     InstMethod1OfT
     InstMethod2OfT
     InstMethodOfS
     PtrMethodOfS

*main.C's method sets:
     InstMethod1OfT
     InstMethod2OfT
     InstMethodOfS
     PtrMethodOfS
     PtrMethodOfT
```

可以看出：
类型C的Method Set = T的Method Set + *S的Method Set
类型*C的Method Set = *T的Method Set + *S的Method Set

同时通过例子可以看出，无论是T还是*S的方法，C或*C类型变量均可调用（编译器甜头），不会被局限在Method Set中。

4、alias type的Method Set

Go支持为已有类型定义alias type，如：
```Go
type MyInterface I
type Mystruct T
```

对于alias type, Method Set是如何定义的呢？我们看下面例子：
```Go
//details-in-go/3/aliastypemethodset.go
package main

import "./utils"

type I interface {
    IMethod1()
    IMethod2()
}

type T struct {
}

func (T) InstMethod() {

}
func (*T) PtrMethod() {

}

type MyInterface I
type MyStruct T

func main() {
    utils.DumpMethodSet((*I)(nil))

    var t T
    utils.DumpMethodSet(&t)
    var pt = &T{}
    utils.DumpMethodSet(&pt)

    utils.DumpMethodSet((*MyInterface)(nil))

    var m MyStruct
    utils.DumpMethodSet(&m)
    var pm = &MyStruct{}
    utils.DumpMethodSet(&pm)
}
```

```shell
$go run aliastypemethodset.go
main.I's method sets:
     IMethod1
     IMethod2

main.T's method sets:
     InstMethod

*main.T's method sets:
     InstMethod
     PtrMethod

main.MyInterface's method sets:
     IMethod1
     IMethod2

main.MyStruct's method set is empty!
*main.MyStruct's method set is empty!
```

从例子的结果上来看，Go对于interface和struct的alias type给出了“不一致”的结果：

MyInterface的Method Set与接口类型I Method Set一致；
而MyStruct并未得到T的哪怕一个Method，MyStruct的Method Set为空。

### 四、Method Type、Method Expression、Method Value

Go中没有class，方法与对象通过receiver联系在一起，我们可以为任何非builtin类型定义method：
```Go
type T struct {
    a int
}

func (t T) Get() int       { return t.a }
func (t *T) Set(a int) int { t.a = a; return t.a }
```

在C++等OO语言中，对象在调用方法时，编译器会自动在方法的第一个参数中传入this/self指针，而对于Go来 说，receiver也是同样道理，将T的method转换为普通function定义：
```Go
func Get(t T) int       { return t.a }
func Set(t *T, a int) int { t.a = a; return t.a }
```

这种function形式被称为Method Type，也可以称为Method的signature。

Method的一般使用方式如下：
```Go
var t T
t.Get()
t.Set(1)
```

不过我们也可以像普通function那样使用它，根据上面的Method Type定义：
```Go
var t T
T.Get(t)
(*T).Set(&t, 1)
```

这种以直接以类型名T调用方法M的表达方法称为Method Expression。类型T只能调用T的Method Set中的方法；同理*T只能调用*T的Method Set中的方法。上述例子中T的Method Set中只有Get，因此T.Get是合法的。但T.Set则不合法：
```Go
    T.Set(2) //invalid method expression T.Set (needs pointer receiver: (*T).Set)
```
我们只能使用(*T).Set(&t, 11)。

这样看来Method Expression有些类似于C++中的static方法(以该类的某个对象实例作为第一个参数)。

另外Method express自身类型就是一个普通function，可以作为右值赋值给一个函数类型的变量：
```Go
    f1 := (*T).Set //函数类型：func (t *T, int)int
    f2 := T.Get //函数类型：func(t T)int
    f1(&t, 3)
    fmt.Println(f2(t))
```

Go中还定义了一种与Method有关的语法：如果一个表达式t具有静态类型T，M是T的Method Set中的一个方法，那么t.M即为Method Value。注意这里是t.M而不是T.M。
```Go
    f3 := (&t).Set //函数类型：func(int)int
    f3(4)
    f4 := t.Get//函数类型：func()int   
    fmt.Println(f4())
```

可以看出，Method value与Method Expression不同之处在于，Method value绑定了T对象实例，它的函数原型并不包含Method Expression函数原型中的第一个参数。完整例子参见：details-in-go/4/methodexpressionandmethodvalue.go。

### 五、for range“坑”大阅兵

for range的引入提升了Go的表达能力，但for range显然不是”免费的午餐“，在享用这个美味前，需要搞清楚for range的一些坑。

1、iteration variable重用

for range的idiomatic的使用方式是使用short variable declaration（:=）形式在for expression中声明iteration variable，但需要注意的是这些variable在每次循环体中都会被重用，而不是重新声明。
```Go
//details-in-go/5/iterationvariable.go
… …
    var m = [...]int{1, 2, 3, 4, 5}

    for i, v := range m {
        go func() {
            time.Sleep(time.Second * 3)
            fmt.Println(i, v)
        }()
    }

    time.Sleep(time.Second * 10)
… …
```


在我的Mac上，输出结果如下：
```Shell
$go run iterationvariable.go
4 5
4 5
4 5
4 5
4 5
```

各个goroutine中输出的i,v值都是for range循环结束后的i, v最终值，而不是各个goroutine启动时的i, v值。一个可行的fix方法：
```Go
    for i, v := range m {
        go func(i, v int) {
            time.Sleep(time.Second * 3)
            fmt.Println(i, v)
        }(i, v)
    }
```

2、range expression副本参与iteration

range后面接受的表达式的类型包括：array, pointer to array, slice, string, map和channel(有读权限的)。我们以array为例来看一个简单的例子：
```Go
//details-in-go/5/arrayrangeexpression.go
func arrayRangeExpression() {
    var a = [5]int{1, 2, 3, 4, 5}
    var r [5]int

    fmt.Println("a = ", a)

    for i, v := range a {
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }
        r[i] = v
    }

    fmt.Println("r = ", r)
}
```
我们期待输出结果：
```shell
a =  [1 2 3 4 5]
r =  [1 12 13 4 5]
a =  [1 12 13 4 5]
```

但实际输出结果却是：
```shell
a =  [1 2 3 4 5]
r =  [1 2 3 4 5]
a =  [1 12 13 4 5]
```

我们原以为在第一次iteration，也就是i = 0时，我们对a的修改(a[1] = 12，a[2] = 13)会在第二次、第三次循环中被v取出，但结果却是v取出的依旧是a被修改前的值：2和3。这就是for range的一个不大不小的坑：range expression副本参与循环。也就是说在上面这个例子里，真正参与循环的是a的副本，而不是真正的a，伪代码如 下：
```Go
    for i, v := range a' {//a' is copy from a
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }
        r[i] = v
    }
```

Go中的数组在内部表示为连续的字节序列，虽然长度是Go数组类型的一部分，但长度并不包含的数组的内部表示中，而是由编译器在编译期计算出 来。这个例子中，对range表达式的拷贝，即对一个数组的拷贝，a'则是Go临时分配的连续字节序列，与a完全不是一块内存。因此无论a被 如何修改，其副本a'依旧保持原值，并且参与循环的是a'，因此v从a'中取出的仍旧是a的原值，而非修改后的值。

我们再来试试pointer to array：
```Go
func pointerToArrayRangeExpression() {
    var a = [5]int{1, 2, 3, 4, 5}
    var r [5]int

    fmt.Println("pointerToArrayRangeExpression result:")
    fmt.Println("a = ", a)

    for i, v := range &a {
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }

        r[i] = v
    }

    fmt.Println("r = ", r)
    fmt.Println("a = ", a)
    fmt.Println("")
}
```
这回的输出结果如下：
```shell
pointerToArrayRangeExpression result:
a =  [1 2 3 4 5]
r =  [1 12 13 4 5]
a =  [1 12 13 4 5]
```

我们看到这次r数组的值与最终a被修改后的值一致了。这个例子中我们使用了*[5]int作为range表达式，其副本依旧是一个指向原数组 a的指针，因此后续所有循环中均是&a指向的原数组亲自参与的，因此v能从&a指向的原数组中取出a修改后的值。

idiomatic go建议我们尽可能的用slice替换掉array的使用，这里用slice能否实现预期的目标呢？我们来试试：
```Go
func sliceRangeExpression() {
    var a = [5]int{1, 2, 3, 4, 5}
    var r [5]int

    fmt.Println("sliceRangeExpression result:")
    fmt.Println("a = ", a)

    for i, v := range a[:] {
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }

        r[i] = v
    }

    fmt.Println("r = ", r)
    fmt.Println("a = ", a)
    fmt.Println("")
}

pointerToArrayRangeExpression result:
a =  [1 2 3 4 5]
r =  [1 12 13 4 5]
a =  [1 12 13 4 5]
```

显然用slice也能实现预期要求。我们可以分析一下slice是如何做到的。slice在go的内部表示为一个struct，由(*T, len, cap)组成，其中*T指向slice对应的underlying array的指针，len是slice当前长度，cap为slice的最大容量。当range进行expression复制时，它实际上复制的是一个 slice，也就是那个struct。副本struct中的*T依旧指向原slice对应的array，为此对slice的修改都反映到 underlying array a上去了，v从副本struct中*T指向的underlying array中获取数组元素，也就得到了被修改后的元素值。

slice与array还有一个不同点，就是其len在运行时可以被改变，而array的len是一个常量，不可改变。那么len变化的 slice对for range有何影响呢？我们继续看一个例子：
```Go
func sliceLenChangeRangeExpression() {
    var a = []int{1, 2, 3, 4, 5}
    var r = make([]int, 0)

    fmt.Println("sliceLenChangeRangeExpression result:")
    fmt.Println("a = ", a)

    for i, v := range a {
        if i == 0 {
            a = append(a, 6, 7)
        }

        r = append(r, v)
    }

    fmt.Println("r = ", r)
    fmt.Println("a = ", a)
}

输出结果：

a =  [1 2 3 4 5]
r =  [1 2 3 4 5]
a =  [1 2 3 4 5 6 7]
```Go

在这个例子中，原slice a在for range过程中被附加了两个元素6和7，其len由5增加到7，但这对于r却没有产生影响。这里的原因就在于a的副本a'的内部表示struct中的 len字段并没有改变，依旧是5，因此for range只会循环5次，也就只获取a对应的underlying数组的前5个元素。

range的副本行为会带来一些性能上的消耗，尤其是当range expression的类型为数组时，range需要复制整个数组；而当range expression类型为pointer to array或slice时，这个消耗将小得多，仅仅需要复制一个指针或一个slice的内部表示（一个struct）即可。我们可以通过 benchmark test来看一下三种情况的消耗情况对比：

对于元素个数为100的int数组或slice，测试结果如下：
```shell
//details-in-go/5/arraybenchmark
go test -bench=.
testing: warning: no tests to run
PASS
BenchmarkArrayRangeLoop-4             20000000           116 ns/op
BenchmarkPointerToArrayRangeLoop-4    20000000            64.5 ns/op
BenchmarkSliceRangeLoop-4             20000000            70.9 ns/op
```

可以看到range expression类型为slice或pointer to array的性能相近，消耗都近乎是数组类型的1/2。

3、其他range expression类型

对于range后面的其他表达式类型，比如string, map, channel，for range依旧会制作副本。

【string】
对string来说，由于string的内部表示为struct {*byte, len)，并且string本身是immutable的，因此其行为和消耗和slice expression类似。不过for range对于string来说，每次循环的单位是rune(code point的值)，而不是byte，index为迭代字符码点的第一个字节的position：
```Go
    var s = "中国人"

    for i, v := range s {
        fmt.Printf("%d %s 0x%x\n", i, string(v), v)
    }
```

输出结果：
0 中 0x4e2d
3 国 0x56fd
6 人 0x4eba

如果s中存在非法utf8字节序列，那么v将返回0xFFFD这个特殊值，并且在接下来一轮循环中，v将仅前进一个字节：
```Go
//byte sequence of s: 0xe4 0xb8 0xad 0xe5 0x9b 0xbd 0xe4 0xba 0xba
    var sl = []byte{0xe4, 0xb8, 0xad, 0xe5, 0x9b, 0xbd, 0xe4, 0xba, 0xba}
    for _, v := range sl {
        fmt.Printf("0x%x ", v)
    }
    fmt.Println("\n")

    sl[3] = 0xd0
    sl[4] = 0xd6
    sl[5] = 0xb9

    for i, v := range string(sl) {
        fmt.Printf("%d %x\n", i, v)
    }

输出结果：

0xe4 0xb8 0xad 0xe5 0x9b 0xbd 0xe4 0xba 0xba

0 4e2d
3 fffd
4 5b9
6 4eba
```

以上例子源码在details-in-go/5/stringrangeexpression.go中可以找到。

【map】

对于map来说，map内部表示为一个指针，指针副本也指向真实map，因此for range操作均操作的是源map。

for range不保证每次迭代的元素次序，对于下面代码：
```Go
 var m = map[string]int{
        "tony": 21,
        "tom":  22,
        "jim":  23,
    }

    for k, v := range m {
        fmt.Println(k, v)
    }
```

输出结果可能是：

tom 22
jim 23
tony 21

也可能是：

tony 21
tom 22
jim 23

或其他可能。


如果map中的某项在循环到达前被在循环体中删除了，那么它将不会被iteration variable获取到。
```Go
    counter := 0
    for k, v := range m {
        if counter == 0 {
            delete(m, "tony")
        }
        counter++
        fmt.Println(k, v)
    }
    fmt.Println("counter is ", counter)
```
反复运行多次，我们得到的两个结果：

tony 21
tom 22
jim 23
counter is  3

tom 22
jim 23
counter is  2

如果在循环体中新创建一个map元素项，那该项元素可能出现在后续循环中，也可能不出现：
```Go
    m["tony"] = 21
    counter = 0

    for k, v := range m {
        if counter == 0 {
            m["lucy"] = 24
        }
        counter++
        fmt.Println(k, v)
    }
    fmt.Println("counter is ", counter)
```
执行结果：

tony 21
tom 22
jim 23
lucy 24
counter is  4

or

tony 21
tom 22
jim 23
counter is  3

以上代码可以在details-in-go/5/maprangeexpression.go中可以找到。

【channel】

对于channel来说，channel内部表示为一个指针，channel的指针副本也指向真实channel。

for range最终以阻塞读的方式阻塞在channel expression上（即便是buffered channel，当channel中无数据时，for range也会阻塞在channel上），直到channel关闭：
```Go
//details-in-go/5/channelrangeexpression.go
func main() {
    var c = make(chan int)

    go func() {
        time.Sleep(time.Second * 3)
        c <- 1
        c <- 2
        c <- 3
        close(c)
    }()

    for v := range c {
        fmt.Println(v)
    }
}
```
运行结果：

1
2
3

如果channel变量为nil，则for range将永远阻塞。

六、select求值 

golang引入的select为我们提供了一种在多个channel间实现“多路复用”的一种机制。select的运行机制这里不赘述，但select的case expression的求值顺序我们倒是要通过一个例子来了解一下：
```Go
// details-in-go/6/select.go

func takeARecvChannel() chan int {
    fmt.Println("invoke takeARecvChannel")
    c := make(chan int)

    go func() {
        time.Sleep(3 * time.Second)
        c <- 1
    }()

    return c
}

func getAStorageArr() *[5]int {
    fmt.Println("invoke getAStorageArr")
    var a [5]int
    return &a
}

func takeASendChannel() chan int {
    fmt.Println("invoke takeASendChannel")
    return make(chan int)
}

func getANumToChannel() int {
    fmt.Println("invoke getANumToChannel")
    return 2
}

func main() {
    select {
    //recv channels
    case (getAStorageArr())[0] = <-takeARecvChannel():
        fmt.Println("recv something from a recv channel")

        //send channels
    case takeASendChannel() <- getANumToChannel():
        fmt.Println("send something to a send channel")
    }
}
```
运行结果：

$go run select.go
invoke takeARecvChannel
invoke takeASendChannel
invoke getANumToChannel

invoke getAStorageArr
recv something from a recv channel

通过例子我们可以看出：
1) select执行开始时，首先所有case expression的表达式都会被求值一遍，按语法先后次序。

invoke takeARecvChannel
invoke takeASendChannel
invoke getANumToChannel

例外的是recv channel的位于赋值等号左边的表达式（这里是：(getAStorageArr())[0]）不会被求值。

2) 如果选择要执行的case是一个recv channel，那么它的赋值等号左边的表达式会被求值：如例子中当goroutine 3s后向recvchan写入一个int值后，select选择了recv channel执行，此时对=左侧的表达式 (getAStorageArr())[0] 开始求值，输出“invoke getAStorageArr”。

### 七、panic的recover过程

Go没有提供“try-catch-finally”这样的异常处理设施，而仅仅提供了panic和recover，其中recover还要结合 defer使用。最初这也是被一些人诟病的点。但和错误码返回值一样，渐渐的大家似乎适应了这些，征讨之声渐稀，即便有也是排在“缺少generics” 之后了。

【panicking】

在没有recover的时候，一旦panic发生，panic会按既定顺序结束当前进程，这一过程成为panicking。下面的例子模拟了这一过程：
```Go
//details-in-go/7/panicking.go
… …
func foo() {
    defer func() {
        fmt.Println("foo defer func invoked")
    }()
    fmt.Println("foo invoked")

    bar()
    fmt.Println("do something after bar in foo")
}

func bar() {
    defer func() {
        fmt.Println("bar defer func invoked")
    }()
    fmt.Println("bar invoked")

    zoo()
    fmt.Println("do something after zoo in bar")
}

func zoo() {
    defer func() {
        fmt.Println("zoo defer func invoked")
    }()

    fmt.Println("zoo invoked")
    panic("runtime exception")
}

func main() {
    foo()
}
```

执行结果：
```shell
$go run panicking.go
foo invoked
bar invoked
zoo invoked
zoo defer func invoked
bar defer func invoked
foo defer func invoked
panic: runtime exception

goroutine 1 [running]:
… …
exit status 2
```

从结果可以看出：
    panic在zoo中发生，在zoo真正退出前，zoo中注册的defer函数会被逐一执行(FILO)，由于zoo defer中没有捕捉panic，因此panic被抛向其caller：bar。
    这时对于bar而言，其函数体中的zoo的调用就好像变成了panic调用似的，zoo有些类似于“黑客帝国3”中里奥被史密斯(panic)感 染似的，也变成了史密斯(panic)。panic在bar中扩展开来，bar中的defer也没有捕捉和recover panic，因此在bar中的defer func执行完毕后，panic继续抛给bar的caller: foo；
    这时对于foo而言，bar就变成了panic，同理，最终foo将panic抛给了main
    main与上述函数一样，没有recover，直接异常返回，导致进程异常退出。
  
【recover】

recover只有在defer函数中调用才能起到recover的作用，这样recover就和defer函数有了紧密联系。我们在zoo的defer函数中捕捉并recover这个panic：
```Go
//details-in-go/7/recover.go
… …
func zoo() {
    defer func() {
        fmt.Println("zoo defer func1 invoked")
    }()

    defer func() {
        if x := recover(); x != nil {
            log.Printf("recover panic: %v in zoo recover defer func", x)
        }
    }()

    defer func() {
        fmt.Println("zoo defer func2 invoked")
    }()

    fmt.Println("zoo invoked")
    panic("zoo runtime exception")
}
```

这回的执行结果如下：
```shell
$go run recover.go
foo invoked
bar invoked
zoo invoked
zoo defer func2 invoked
2015/09/17 16:28:00 recover panic: zoo runtime exception in zoo recover defer func
zoo defer func1 invoked
do something after zoo in bar
bar defer func invoked
do something after bar in foo
foo defer func invoked
```
由于zoo在defer里恢复了panic，这样在zoo返回后，bar不会感知到任何异常，将按正常逻辑输出函数执行内容，比如：“do something after zoo in bar”,以此类推。

但若如果在zoo defer func中recover panic后，又raise another panic，那么zoo对于bar来说就又会变成panic了。

### Last、参考资料

1、The Go Programming Language Specification (Version of August 5, 2015，Go 1.5)；
2、Effective Go (Go 1.5)；
3、Rob Pike： Go Course Day 1~3。

本文实验环境：Go 1.5 darwin_amd64。

本文作者博文地址：http://tonybai.com/2015/09/17/7-things-you-may-not-pay-attation-to-in-go/

 
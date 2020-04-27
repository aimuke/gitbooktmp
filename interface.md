# interface

## 引言

我们先从一个例子开始

```go
func fun1() interface{} { 
    var x *interface{} = nil
    return x
}

func main() { 
    x := fun1()
    
    if x == nil {
        fmt.Println("is nil")
    } else {
        fmt.Println("is not nil")
    }
}
```

上面的例子中函数 `fun1` 返回值为 `nil` 的变量，然后在 `main`函数中判断，返回值是否等于 `nil`，输出为 `is not nil.` 这是什么原因呢，我们一起看看 interface的底层结构`。`

## interface 底层结构

根据 `interface` 是否包含有 `method`，底层实现上用两种 `struct` 来表示：`iface`和 `eface`。`eface` 表示不含 method 的 interface 结构，或者叫 `empty interface`。对于 Golang 中的大部分数据类型都可以抽象出来 `_type` 结构，同时针对不同的类型还会有一些其他信息。

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
    size       uintptr // type size
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32  // hash of type; avoids computation in hash tables
    tflag      tflag   // extra type information flags
    align      uint8   // alignment of variable with this type
    fieldalign uint8   // alignment of struct field with this type
    kind       uint8   // enumeration for C
    alg        *typeAlg  // algorithm table
    gcdata    *byte    // garbage collection data
    str       nameOff  // string form
    ptrToThis typeOff  // type for pointer to this type, may be zero
}
```

`iface` 表示 `non-empty interface` 的底层实现。相比于 empty interface，non-empty 要包含一些 `method`。method 的具体实现存放在 `itab.fun` 变量里。

你可能会觉得奇怪，为什么 `fun` 数组的大小为 `1`，要是接口定义了多个方法可怎么办？实际上，这里存储的是第一个方法的函数指针，如果有更多的方法，在它之后的内存空间里继续存储。从汇编角度来看，通过增加地址就能获取到这些函数指针，没什么影响。顺便提一句，这些方法是按照函数名称的字典序进行排列的。

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    bad    int32
    inhash int32      // has this itab been added to hash?
    fun    [1]uintptr // variable sized
}
```

```go
package main

import (
    "fmt"
)

type MyInterface interface {
    Print()
}

type MyStruct struct{}
func (ms MyStruct) Print() {}

func main() {
    x := 1
    var y interface{} = x
    var s MyStruct
    var t MyInterface = s
    fmt.Println(y, z)
}
```

查看汇编代码

```bash
$ go build -gcflags '-l' -o interface11 interface11.go
$ go tool objdump -s "main\.main" interface11
TEXT main.main(SB) /Users/kltao/code/go/examples/interface11.go
    interface11.go:15   0x10870f0   65488b0c25a0080000  GS MOVQ GS:0x8a0, CX
    interface11.go:15   0x10870f9   483b6110        CMPQ 0x10(CX), SP
    interface11.go:15   0x10870fd   0f86de000000        JBE 0x10871e1
    interface11.go:15   0x1087103   4883ec70        SUBQ $0x70, SP
    interface11.go:15   0x1087107   48896c2468      MOVQ BP, 0x68(SP)
    interface11.go:15   0x108710c   488d6c2468      LEAQ 0x68(SP), BP
    interface11.go:17   0x1087111   48c744243001000000  MOVQ $0x1, 0x30(SP)
    interface11.go:17   0x108711a   488d057fde0000      LEAQ 0xde7f(IP), AX
    interface11.go:17   0x1087121   48890424        MOVQ AX, 0(SP)
    interface11.go:17   0x1087125   488d442430      LEAQ 0x30(SP), AX
    interface11.go:17   0x108712a   4889442408      MOVQ AX, 0x8(SP)
    interface11.go:17   0x108712f   e87c45f8ff      CALL runtime.convT2E(SB)
    interface11.go:17   0x1087134   488b442410      MOVQ 0x10(SP), AX
    interface11.go:17   0x1087139   4889442438      MOVQ AX, 0x38(SP)
    interface11.go:17   0x108713e   488b4c2418      MOVQ 0x18(SP), CX
    interface11.go:17   0x1087143   48894c2440      MOVQ CX, 0x40(SP)
    interface11.go:19   0x1087148   488d15b1000800      LEAQ 0x800b1(IP), DX
    interface11.go:19   0x108714f   48891424        MOVQ DX, 0(SP)
    interface11.go:19   0x1087153   488d542430      LEAQ 0x30(SP), DX
    interface11.go:19   0x1087158   4889542408      MOVQ DX, 0x8(SP)
    interface11.go:19   0x108715d   e8fe45f8ff      CALL runtime.convT2I(SB)

```

代码 17 行 `var y interface{} = x` 调用了函数 `runtime.convT2E` ，将 `int` 类型的 `x` 转换成 `empty interface`。代码 19 行 `var t MyInterface = s` 将 `MyStruct` 类型转换成 `non-empty interface: MyInterface`。

```go
func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
    ...
  
    x := newobject(t)
    typedmemmove(t, x, elem)
    e._type = t
    e.data = x
    return
}

func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    
    ...
  
    x := newobject(t)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}
```

看上面的函数原型，可以看出中间过程编译器将根据我们的转换目标类型的 `empty interface` 还是 `non-empty interface`，来对原数据类型进行转换（转换成 &lt;_\_type, unsafe.Pointer&gt; 或者 &lt;_itab, unsafe.Pointer&gt;）。这里对于 struct 满不满足 interface 的类型要求（也就是 struct 是否实现了 interface 的所有 method），是由编译器来检测的。

## itab

`iface` 结构中最重要的是 `itab` 结构。`itab` 可以理解为 `pair` 。当然 `itab` 里面还包含一些其他信息，比如 interface 里面包含的 `method` 的具体实现。下面细说。`itab` 的结构如下。

```go
type itab struct {
    inter  *interfacetype  // 接口自身信息
    _type  *_type           // concrete type 对象实际的类型
    link   *itab
    bad    int32
    inhash int32      // has this itab been added to hash?
    fun    [1]uintptr // variable sized
}

type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}

type imethod struct {   //这里的 method 只是一种函数声明的抽象，比如  func Print() error
    name nameOff
    ityp typeOff
}
```

`fun` 表示的 interface 里面的 method 的具体实现。比如 interface type 包含了 method A, B，则通过 `fun` 就可以找到这两个 `method` 的具体实现。这里有个问题 `fun` 是长度为 `1` 的 `uintptr` 数组，那么怎么表示多个 method 呢？看一下测试程序。

```go
package main

type MyInterface interface {
    Print()
    Hello()
    World()
    AWK()
}

func Foo(me MyInterface) {
    me.Print()
    me.Hello()
    me.World()
    me.AWK()
}

type MyStruct struct {}

func (me MyStruct) Print() {}
func (me MyStruct) Hello() {}
func (me MyStruct) World() {}
func (me MyStruct) AWK() {}

func main() {
    var me MyStruct
    Foo(me)
}
```

看一下函数调用对应的汇编代码。

```bash
go build -gcflags '-l' -o interface8 interface8.go
$ go tool objdump -s "main\.Foo" interface8
TEXT main.Foo(SB) /Users/kltao/code/go/examples/interface8.go
    interface8.go:10    0x104c060   65488b0c25a0080000  GS MOVQ GS:0x8a0, CX
    interface8.go:10    0x104c069   483b6110        CMPQ 0x10(CX), SP
    interface8.go:10    0x104c06d   7668            JBE 0x104c0d7
    interface8.go:10    0x104c06f   4883ec10        SUBQ $0x10, SP
    interface8.go:10    0x104c073   48896c2408      MOVQ BP, 0x8(SP)
    interface8.go:10    0x104c078   488d6c2408      LEAQ 0x8(SP), BP
    interface8.go:11    0x104c07d   488b442418      MOVQ 0x18(SP), AX
    interface8.go:11    0x104c082   488b4830        MOVQ 0x30(AX), CX //取得 Print 函数地址
    interface8.go:11    0x104c086   488b542420      MOVQ 0x20(SP), DX
    interface8.go:11    0x104c08b   48891424        MOVQ DX, 0(SP)
    interface8.go:11    0x104c08f   ffd1            CALL CX     // 调用 Print()
    interface8.go:12    0x104c091   488b442418      MOVQ 0x18(SP), AX
    interface8.go:12    0x104c096   488b4828        MOVQ 0x28(AX), CX //取得 Hello 函数地址
    interface8.go:12    0x104c09a   488b542420      MOVQ 0x20(SP), DX
    interface8.go:12    0x104c09f   48891424        MOVQ DX, 0(SP)
    interface8.go:12    0x104c0a3   ffd1            CALL CX           //调用 Hello()
    interface8.go:13    0x104c0a5   488b442418      MOVQ 0x18(SP), AX
    interface8.go:13    0x104c0aa   488b4838        MOVQ 0x38(AX), CX //取得 World 函数地址
    interface8.go:13    0x104c0ae   488b542420      MOVQ 0x20(SP), DX 
    interface8.go:13    0x104c0b3   48891424        MOVQ DX, 0(SP)
    interface8.go:13    0x104c0b7   ffd1            CALL CX           //调用 World()
    interface8.go:14    0x104c0b9   488b442418      MOVQ 0x18(SP), AX
    interface8.go:14    0x104c0be   488b4020        MOVQ 0x20(AX), AX //取得 AWK 函数地址
    interface8.go:14    0x104c0c2   488b4c2420      MOVQ 0x20(SP), CX
    interface8.go:14    0x104c0c7   48890c24        MOVQ CX, 0(SP)
    interface8.go:14    0x104c0cb   ffd0            CALL AX           //调用 AWK()
    interface8.go:15    0x104c0cd   488b6c2408      MOVQ 0x8(SP), BP
    interface8.go:15    0x104c0d2   4883c410        ADDQ $0x10, SP
    interface8.go:15    0x104c0d6   c3          RET
    interface8.go:10    0x104c0d7   e8f48bffff      CALL runtime.morestack_noctxt(SB)
    interface8.go:10    0x104c0dc   eb82            JMP main.Foo(SB)
```

其中 `0x18(SP)` 对应的 `itab` 的值。`fun` 在 `x86-64` 机器上对应 `itab` 内的地址偏移为 `8+8+8+4+4 = 32 = 0x20`，也就是 `0x20(AX)` 对应的 `fun` 的值，此时存放的 `AWK` 函数地址。然后 `0x28(AX) = &Hello`，`0x30(AX) = &Print`，`0x38(AX) = &World`。每次函数是按字典序排序存放的。

我们再来看一下函数地址究竟是怎么写入的？首先 Golang 中的 uintptr 一般用来存放指针的值，这里对应的就是函数指针的值（也就是函数的调用地址）。但是这里的 fun 是一个长度为 1 的 uintptr 数组。我们看一下 runtime 包的 additab 函数。

```go
func additab(m *itab, locked, canfail bool) {
    ...
    *(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
    ...
}
```

上面的代码的意思是在 `fun[0]` 的地址后面依次写入其他 method 对应的函数指针。熟悉 C++ 的同学可以类比 C++ 的虚函数表指针来看。

剩下的还有 bad，link，inhash。其中 bad 是一个表征 itab 状态的变量。而这里的 link 是 \*itab 类型，是不是表示 interface 的嵌套呢？并不是，interface 的嵌套也是把 method 平铺而已。link 要和 inhash 一起来说。在 runtime 包里面有一个 hash 表，通过 `hash[hashitab(interface_type, concrete_type)]` 可以取得 itab，这是出于性能方面的考虑。主要代码如下，这里就不再赘述了。

```go
const (
    hashSize = 1009
)

var (
    ifaceLock mutex // lock for accessing hash
    hash      [hashSize]*itab
)

func itabhash(inter *interfacetype, typ *_type) uint32 {
    // compiler has provided some good hash codes for us.
    h := inter.typ.hash
    h += 17 * typ.hash
    // TODO(rsc): h += 23 * x.mhash ?
    return h % hashSize
}

func additab(...) {
    ...
    h := itabhash(inter, typ)
    m.link = hash[h]
    m.inhash = 1
    atomicstorep(unsafe.Pointer(&hash[h]), unsafe.Pointer(m))
}
```

## Type Assertion

 我们知道使用 interface type assertion \(中文一般叫断言\) 的时候需要注意，不然很容易引入 `panic`。

```go
func do(v interface{}) {
    n := v.(int)    // might panic
}

func do(v interface{}) {
    n, ok := v.(int)
    if !ok {
        // 断言失败处理
    }
}
```

这个过程体现在下面的几个函数上。

```go
// The assertXXX functions may fail (either panicking or returning false,
// depending on whether they are 1-result or 2-result).
func assertI2I(inter *interfacetype, i iface) (r iface) {
    tab := i.tab
    if tab == nil {
        // explicit conversions require non-nil interface value.
        panic(&TypeAssertionError{"", "", inter.typ.string(), ""})
    }
    if tab.inter == inter {
        r.tab = tab
        r.data = i.data
        return
    }
    r.tab = getitab(inter, tab._type, false)
    r.data = i.data
    return
}
func assertI2I2(inter *interfacetype, i iface) (r iface, b bool) {
    tab := i.tab
    if tab == nil {
        return
    }
    if tab.inter != inter {
        tab = getitab(inter, tab._type, true)
        if tab == nil {
            return
        }
    }
    r.tab = tab
    r.data = i.data
    b = true
    return
}

// 类似
func assertE2I(inter *interfacetype, e eface) (r iface)
func assertE2I2(inter *interfacetype, e eface) (r iface, b bool)
```

{% hint style="info" %}


 Super-powers are granted randomly so please submit an issue if you're not happy with yours.
{% endhint %}

Once you're strong enough, save the world:

{% code title="hello.sh" %}
```bash
# Ain't no code for that yet, sorry
echo 'You got to trust me on this, I saved the world'
```
{% endcode %}

## References

* [Go Interface 源码剖析](http://legendtkl.com/2017/07/01/golang-interface-implement/)




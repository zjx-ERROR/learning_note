<!--
 * @Author: zhangjiaxi
 * @Date: 2021-02-25 10:17:38
 * @LastEditors: zhangjiaxi
 * @LastEditTime: 2021-03-01 17:54:20
 * @FilePath: /learning_note/goProgrammingMode.md
 * @Description: 
-->
# 切片，接口，时间和性能

## slice
切片是一个结构体
```go
type slice struct{
    array unsafe.Pointer   //指向存放数据的数组指针
    len   int              //长度
    cap   int              //容量
}
```
结构体里的数组，数据会发生共享
```go
foo := make([]int, 5)
foo[3] = 42
foo[4] = 100
bar := foo[1:4]
bar[1] = 99
```
对于上面这段代码
- 首先先创建一个foo的slice，其中的长度和容量都是5
- 然后开始对foo所指向的数组中的索引为3和4的元素进行赋值
- 对foo作切片后赋值给bar，再修改bar[1]

![1](img/goProgrammingMode/1.png)

通过上图可以看出，因为foo和bar的内存是共享的，所以，foo和bar的对数组内容的修改都会影响到对方。

接下来，我们再来看一个数据操作 append() 的示例
```go
a := make([]int, 32)
b := a[1:16]
a = append(a, 1)
a[2] = 42
```
上面这段代码中，把a[1:16]的切片赋值给到了b，此时，a和b的内存空间是共享的，然后，对a做了一个append()的操作，这个操作会让a重新分配内存，导致a和b不再共享，如下图所示

![2](img/goProgrammingMode/2.png)

从上图可以看到append()操作让a的容量变成了64，而长度是33。这里需要注意以下--append()函数在cap不够用的时候就会重新分配内存以扩大容量，而如果够用的时候不会重新分配内存。

## 接口完整性检查
go语言的编译器并没有严格检查一个对象是否实现了某接口所有的接口方法，如下示例
```go
type Shape interface {
    Sides() int
    Area() int
}
type Square struct {
    len int
}
func (s* Square) Sides() int {
    return 4
}
func main() {
    s := Square{len: 5}
    fmt.Printf("%d\n",s.Sides())
}
```
可以看到Square并没有实现Shape接口的所有方法，程序虽然可以跑通，但如果需要强制实现接口的所有方法，可以这样做
```go
var _ Shape = (*Square)(nil)
或
var _ Shape = new(Square)
```
声明一个_变量（没人用），其会把一个nil的空指针，从Square转成Shape，这样如果没有实现相关接口方法，编译器就会报错。

## 性能提示
- 如果需要把数字转字符串，使用strconv.Itoa()会比fmt.Sprintf()要快一倍左右
- 尽可能地避免把String转成[]Byte。这个转换会导致性能下降
- 如果在for-loop里对某个slice使用append()请把slice的容量扩充到位，这样可以避免内存重新分配以及系统自动按2的N次方幂进行扩展又用不到，从而浪费内存
- 使用StringBuffer或是StringBuild来拼接字符串，会比使用+或+=性能高3到4个数量级
- 尽可能使用并发的goroutine，然后使用sync.WaiGroup来同步分片操作
- 避免在热代码中进行内存分配，这样会导致gc很忙。尽可能的使用sync.Pool来重用对象
- 使用lock-free的操作，避免使用mutex，尽可能使用sync/Atomic包
- 使用I/O缓冲，I/O是个非常非常慢的操作，使用bufio.NewWrite()和bufio.NewReader()可以带来更高的性能
- 对于在for-loop里的固定的正则表达式，一定使用regexp.Compile()编译正则表达式。性能提升两个数量级
- 如果需要更高性能的协议，考虑使用protobuf或msgp而不是json，因为json的序列化和反序列化使用了反射
- 在使用map的时候，使用整型的key会比字符串的要快，因为整型比较字符串比较快

# 配置选项问题
## Functional Options
首先，定义一个函数类型：
```go
type Option func(*Server)
```
然后，我门可以使用函数式的方式定义一组如下的函数：
```go
func Protocol(p string) Option{
    return func(s *Server){
        s.Protocol = p
    }
}
func Timeout(timeout time.Duration) Option{
    return func(s *Server){
        s.Timeout = timeout
    }
}
func MaxConns(maxconns int) Option{
    return func(s *Server){
        s.MaxConns = maxconns
    }
}
func TLS(tls *tls.Config) Option{
    return func(s *Server){
        s.TLS = tls
    }
}
```
上面这组代码传入一个参数，然后返回一个函数，返回的这个函数会设置自己的Server参数。
```go
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {
  srv := Server{
    Addr:     addr,
    Port:     port,
    Protocol: "tcp",
    Timeout:  30 * time.Second,
    MaxConns: 1000,
    TLS:      nil,
  }
  for _, option := range options {
    option(&srv)
  }
  //...
  return &srv, nil
}
```
于是，在创建Server对象的时候，可以这样：
```go
s1, _ := NewServer("localhost", 1024)
s2, _ := NewServer("localhost", 2048, Protocol("udp"))
s3, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second), MaxConns(1000))
```

# 委托和控制反转
反转控制IoC-inversion of Control是一种软件设计的方法，其主要的思想是把控制逻辑与业务逻辑分离，不要在业务逻辑里写控制逻辑，这样会让溶质逻辑依赖业务逻辑。这样的编程方式可以有效的降低程序复杂度，并提升代码重用。
## 嵌入和委托
在Go语言中，可以很方便的把一个结构体嵌入到另一个结构体中：
```go
type Widget struct{
    X,Y int
}
type Label struct{
    Widget
    Text string
}

label := Label{Widget{10,10},"State:"}
label.X = 11
label.Y = 12
```
有了这样的嵌入，就可以像UI组件一样在结构体的设计上进行层层分解。比如，可以新建两个结构体Button和ListBox：
```go
type Button struct {
    Label // Embedding (delegation)
}
type ListBox struct {
    Widget          // Embedding (delegation)
    Texts  []string // Aggregation
    Index  int      // Aggregation
}
```
- 对于Lable来说，只有Painter，没有Cliker
- 对于Button和ListBox来说，Painter和Cliker都有
下面是一些实现：
```go 
func (label Label) Paint() {
  fmt.Printf("%p:Label.Paint(%q)\n", &label, label.Text)
}
//因为这个接口可以通过 Label 的嵌入带到新的结构体，
//所以，可以在 Button 中可以重载这个接口方法以
func (button Button) Paint() { // Override
    fmt.Printf("Button.Paint(%s)\n", button.Text)
}
func (button Button) Click() {
    fmt.Printf("Button.Click(%s)\n", button.Text)
}
func (listBox ListBox) Paint() {
    fmt.Printf("ListBox.Paint(%q)\n", listBox.Texts)
}
func (listBox ListBox) Click() {
    fmt.Printf("ListBox.Click(%q)\n", listBox.Texts)
}
```
Button.Paint()接口可以通过Label的嵌入带到新的结构体，如果Button.Paint()不实现的话，会调用Label.Paint()，所以，在Button中声明Paint()方法相当于Oerride。

## 反转控制
有一个存放整数的数据结构，如下所示：
```go
type IntSet struct{
    data map[int]bool
}
func NewInSet() IntSet{
    return IntSet{make(map[int]bool)}
}
func (set *IntSet) Add(x int){
    set.data[x] = true
}
func (set *IntSet) Delete(x int){
    delete(set.data,x)
}
func (set *IntSet) Contains(x int) bool{
    return set.data[x]
}
```
现在实现一个Undo的功能，我们可以把IntSet再包装一下变成UndoableIntSet：
```go
type UndoableIntSet struct{
    IntSet
    functions []func()
}
func NewUndoableIntSet() UndoableIntSet{
    return UndoableIntSet{NewIntSet(),nil}
}
func (set *UndoableIntSet) Add(x int){
    if !set.Contains(x){
        set.data[x] = true
        set.functions = append(set.functions,func(){set.Delete(x)})
    }else{
        set.functions = append(set.functions,nil)
    }
}

func (set *UndoableIntSet) Delete(x int){
    if set.Contains(x){
        delete(set.data,x)
        set.functions = append(set.functions,func(){set.Add(x)})
    }else{
        set.functions = append(set.functions,nil)
    }
}

func (set *UndoableIntSet) Undo()error{
    if len(set.functions) == 0{
        return errors.New("No functions to undo")
    }
    index := len(set.functions) - 1
    if functino := set.functions[index];function != nil{
        function()
        set.functions[index] = nil
    }
    set.functions = set.functions[:index]
    return nil
}
```
- 我们在UndoableIntSet中嵌入了IntSet，然后Override了它的Add()和Delete()方法
- Contains()方法没有Override，所以被带到UndoableIntSet中
- 在Override的Add()中，记录Delete操作
- 在Override的Delete()中，记录Add操作
- 在新加入Undo()中进行Undo操作

通过这样的方式来为已有的代码扩展新的功能是一个很好的选择，这样，在重用原有代码功能和重新新的功能中达到一个平衡。但是，这种方式最大的问题是，Undo这个功能上是有问题的。因为加入大量跟IntSet相关的业务逻辑。

## 反转依赖
```go
type Undo []func()
func (undo *Undo) Add(function func()){
    *undo = append(*undo,function)
}

func (undo *Undo) Undo()error{
    functions := *undo
    if len(functions) == 0{
        return errors.New("No functions to undo")
    }
    index := len(functions) - 1
    if function := functions[index];function != nil{
        function()
        functions[index] = nil
    }
    *undo = funcions[:index]
    return nil
}

type IntSet struct{
    data map[int]bool
    undo Undo
}
func NewInSet() IntSet{
    return IntSet{make(map[int]bool)}
}
func (set *IntSet) Undo()error{
    return set.undo.Undo()
}
func (set *IntSet) Contains(x int) bool{
    return set.data[x]
}
func (set *IntSet) Add(x int){
    if set.Contains(x){
        set.data[x] = true
        set.undo.Add(func(){set.Delete(x)})
    }else{
        set.undo.Add(nil)
    }
    
}
func (set *IntSet) Delete(x int){
    if set.Contains(x){
        delete(set.data,x)
        set.undo.Add(func(){set.Add(x)})
    }else{
        set.undo.Add(nil)
    }
}
这个就是控制反转，不再由控制逻辑Undo来依赖业务逻辑IntSet，而是由业务逻辑Intet来依赖Undo。
```

# Map-Reduce

```go
func MapStrToStr(arr []string,fn func(s string) string) []string{
    var newArray = []string{}
    for _,it := range arr{
        newArray = append(newArray,fn(it))
    }
    return newArray
}
func MapStrToInt(arr []string,fn func(s string) int) []int{
    var newArray = []int{}
    for _,it := range arr {
        newArray = append(newArray,fn(it))
    }
    return newArray
}
```

整个Map函数运行逻辑都很相似，函数体都是在遍历第一个参数的数组，然后调用第二个参数的函数，然后把其值组合成另一个数组返回。
再来看看Reduce和Filter的函数是怎么样的：
```go
func Reduce(arr []string,fn func(s string) int) int{
    sum := 0
    for _,it := range arr{
        sum += fn(it)
    }
    return sum
}
var list = []string{"Hao","Chen","MegaEase"}

x := Reduce(list,func(s string) int{
    return len(s)
})
fmt.Println("%v\n",x)

func Filter(arr []int,fn func(n int) bool) []int{
    var newArray = []int{}
    for _,it := range arr{
        if fn(it){
            newArray = append(newArray,it)
        }
    }
    return newArray
}
var intset = []int{1,2,3,4,5,6,7,8,9}
out := Filter(intset,func(n int)bool{
    return n%2 == 1
})
fmt.Println("%v\n",out)
out = Filter(intset,func(n int)bool{
    return n > 5
})
fmt.Println("%v\n",out)
```
通过上面的示例，你可能有一些明白，Map/Reduce/Filter只是一种控制逻辑，真正的业务逻辑是在传给他们的数据和那个函数来定义的。这是一个很经典的业务逻辑和控制逻辑分离解耦的编程模式。

目前go的泛型只能用interface{}+reflect来完成，interface{}可以理解成c的void*，java中的Object，reflect是go的反射机制包，用于在运行时检查类型。下面是不作任何类型检查的泛型Map函数：
```go
func Map(data interface{},fn interface{}) []interface{}{
    vfn := reflect.ValueOf(fn)
    vdata := reflect.ValueOf(data)
    result := make([]interface{},vdata.len())
    for i := 0;i < vdata.len();i++{
        result[i] = vfn.Call([]reflect.Value{vdata.Index(i)})[0].Interface()
    }
    return result
}
```
- 通过reflect.ValueOf()来获得interface{}的值，其中一个是数据vdata，另一个是函数vfn
- 通过vfn.Call()方法来调用函数，通过[]reflect.Value{vdata.Index(i)}来获得数据

于是，不用类型的数据可以使用相同逻辑的Map()代码：
```go
square := func(x int) int{
    return x * x
}
nums := []int{1,2,3,4}

squared_arr := Map(nums)
fmt.Println(squared_arr)
upcase := func(s string) string{
    return strings.ToUpper(s)
}
strs := []string{"HAO","Chen","MegaEase"}
upstrs := Map(strs,upcase)
fmt.Println(upstrs)
```
但是因为反射是运行时的事，所以，如果类型什么出问题的话，就会有运行时的错误。

# go generation

## go的类型检查
因为go目前不支持真正的泛型，所以只能用interface{}这样的类似void*这种过渡泛型来玩，这就导致在实际过程中就需要进行类型检查。go的类型检查有两种技术，一种是Type Assert，一种是Reflection。

## Type Assert
一般是对某个变量进行(.type)的转型操作，其中返回两个值，variable,error，第一个返回值是被转换好的类型，第二个是如果不能转换类型，就会报错。

例如，我们有一个通用类型的容器，可以进行Put(val)和Get()，注意其使用了interface{}作泛型：
```go
type Container []interface{}

func(c *Container) Put(elem interface{}){
    *c = append(*c,elem)
}

func(c *Container) Get() interface{}{
    elem := (*c)[0]
    *c = (*c)[1:]
    return elem
}

intContainer := &Container{}
intContainer.Put(7)
intContainer.Put(65)

elem,ok := intContainer.Get().(int)
if !ok{
    fmt.Println("Unable to read an int from intContainer")
}
fmt.Printf("assertExample: %d (%T)\n", elem, elem)
```
## Reflection
对于反射，修改上面代码如下：
```go
type Container struct{
    s reflect.Value
}
func NewContainer(t reflect.Type,size int) *Container{
    if size <= 0{size=64}
    return &Container{
        s : reflect.MakeSlice(reflect.Sliceof(t),0,size)
    }
}
func (c *Container)Put(val interface{}) error {
    if reflect.ValueOf(val).Type() != c.s.Type().Elem(){
        return fmt.Errorf("Put:cannot put a %T into a slice of %s",val,c.s.Type().Elem())
    }
    c.s = reflect.Append(c.s,reflect.ValueOf(val))
    return nil
}
func (c *Container) Get(refval interface{}) error{
    if reflect.ValueOf(refval).Kind() != reflect.Ptr ||
        reflect.ValueOf(refval).Elem().Type() != c.s.Type().Elem(){
            return fmt.Errorf("Get:needs %s but got %T",c.s.Type().Elem(),refval)
        }
        reflect.ValueOf(refval).Elem().Set(c.s.Index(0))
        c.s = c.s.Slice(1,c.s.len())
        return nil
}

f1 := 3.1415926
f2 := 1.141421356237
c := NewContainer(reflect.TypeOf(f1),16)

if err := c.Put(f1);err != nil{
    panic(err)
}
if err := c.Put(f2);err != nil{
    panic(err)
}
g := 0.0
if err := c.Get(&g);err != nil{
    panic(err)
}
fmt.Printf("%v (%T)\n", g, g)
fmt.Println(c.s.Index(0))
```
- 在NewContainer()会根据参数的类型初始化一个Slice
- Put()会检查val是否和Slice的类型一致
- Get()我们需要用一个入参的方式，因为我们没有办法返回reflect.Value或是interface{}，不然还要做Type Assert
- 但是有类型检查，所以必然会有检查不对的时候，因此需要返回error

## go generator
要玩go的代码生成，你需要三件事：
1、 一个函数模板，其中设置好相应的占位符
2、 一个脚本，用于按规则来替换文本并生成新的代码
3、 一行注释代码

### 函数模板
我们把之前的示例改称模板。取名为container.tmp.go放在./template/下
```go
package PACKAGE_NAME
type GENERIC_NAMEContainer struct{
    s []GENERIC_TYPE
}
func NewGENERIC_NAMEContainer()*GENERIC_NAMEContainer{
    return &GENERIC_NAMEContainer{s:[]GENERIC_TYPE{}}
}
func(c *GENERIC_NAMEContainer)Put(val GENERIC_TYPE){
    c.s = append(c.s, val)
}
func (c *GENERIC_NAMEContainer) Get() GENERIC_TYPE {
    r := c.s[0]
    c.s = c.s[1:]
    return r
}
```
可以看到函数模板中我们有如下占位符：
- PACKAGE_NAME - 包名
- GENERIC_NAME - 名字
- GENERIC_TYPE - 实际的类型

### 函数生成脚本
有一个gen.sh的生成脚本：
```sh
#!/bin/bash

set -e

SRC_FILE=${1}
PACKAGE=${2}
TYPE=${3}
DES=${4}
PREFIX="$(tr '[:lower:]' '[:upper:]' <<< ${TYPE:0:1})${TYPE:1}"
DES_FILE=$(echo ${TYPE}| tr '[:upper:]' '[:lower:]')_${DES}.go
sed 's/PACKAGE_NAME/'"${PACKAGE}"'/g' ${SRC_FILE} | \
    sed 's/GENERIC_TYPE/'"${TYPE}"'/g' | \
    sed 's/GENERIC_NAME/'"${PREFIX}"'/g' > ${DES_FILE}
```
其需要4个参数：
- 模板源文件
- 包名
- 世纪需要具体化的类型
- 用于构造目标文件名的后缀
然后会用sed命令去替换上面的函数模板，并生成到目标文件中。

### 生成代码
```go 
//go:generate ./gen.sh ./template/container.tmp.go gen uint32 container
func generateUint32Example() {
    var u uint32 = 42
    c := NewUint32Container()
    c.Put(u)
    v := c.Get()
    fmt.Printf("generateExample: %d (%T)\n", v, v)
}
//go:generate ./gen.sh ./template/container.tmp.go gen string container
func generateStringExample() {
    var s string = "Hello"
    c := NewStringContainer()
    c.Put(s)
    v := c.Get()
    fmt.Printf("generateExample: %s (%T)\n", v, v)
}
```
- 第一个注释是生成包名为gen类型为uint32目标文件名以container为后缀
- 第二个注释是生成包名为gen类型为string目标文件名以container为后缀
然后在工程目录直接执行go generate命令，就会生成如下两份代码：
uint32_container.go
```go
package gen
type Uint32Container struct {
    s []uint32
}
func NewUint32Container() *Uint32Container {
    return &Uint32Container{s: []uint32{}}
}
func (c *Uint32Container) Put(val uint32) {
    c.s = append(c.s, val)
}
func (c *Uint32Container) Get() uint32 {
    r := c.s[0]
    c.s = c.s[1:]
    return r
}
```
string_container.go
```go
package gen
type StringContainer struct {
    s []string
}
func NewStringContainer() *StringContainer {
    return &StringContainer{s: []string{}}
}
func (c *StringContainer) Put(val string) {
    c.s = append(c.s, val)
}
func (c *StringContainer) Get() string {
    r := c.s[0]
    c.s = c.s[1:]
    return r
}
```
这两份代码可以让我们的代码完全编译通过，所付出的代价就是需要多执行一步 go generate 命令。
### 第三方工具
我们并不需要自己手写 gen.sh 这样的工具类，已经有很多第三方的已经写好的可以使用。下面是一个列表：
- Genny –  https://github.com/cheekybits/genny
- Generic – https://github.com/taylorchu/generic
- GenGen – https://github.com/joeshaw/gengen
- Gen – https://github.com/clipperhouse/gen

# 修饰器
## 简单示例
```go
package main
import "fmt"
func decorator(f func(s string)) func(s string){
    return func(s string){
        fmt.Println("Started")
        f(s)
        fmt.Println("Done")
    }
}

func Hello(s string){
    fmt.Println(s)
}

func main(){
    decorator(Hello)("Hello,World!")
}
```

我们动用了一个高阶函数decoratoe()，在调用的时候，先把Hello()函数传进去，然后返回一个匿名函数，这个匿名函数中除了运行自己的代码，也调用了被传入的Hello()函数。

```go
package main

import (
    "fmt"
    "reflect"
    "runtime"
    "time"
)

type SumFunc func(int64,int64) int64

func getFuncionName(i interface{}) string{
    return runtime.FuncForPC(reflect.ValueOf(i).Pointer()).Name()
}
func timedSumFunc(f SumFunc) SumFunc{
    return func(start,end int64) int64{
        defer func(t time.Time){
            fmt.Printf("--- Time Elapsed (%s): %v ---\n", 
                getFunctionName(f), time.Since(t))
        }(time.Now())
        return f(start,end)
    }
}

func Sum1(start,end int64) int64{
    var sum int64
    sum = 0
    if start > end{
        start,end = end,start
    }
    for i := start;i <= end;i++{
        sum += 1
    }
    return sum
}

func Sum2(start,end int64) int64{
    if start > end{
        start,end = end,start
    }
    return (end - start + 1) * (end + start) / 2
}

func main(){
    sum1 := timedSumFunc(Sum1)
    sum2 := timedSumFunc(Sum2)

    fmt.Printf("%d, %d\n", sum1(-10000, 10000000), sum2(-10000, 10000000))
}
```
- 有两个Sum函数，Sum1()函数就是简单的做个循环。Sum2()函数动用了数据公式
- 代码中使用了go的反射机制来获取函数名
- 修饰器函数是timedSumFunc()

## HTTP相关示例
```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strings"
)

func WithServerHeader(h http.HandlerFunc) http.HandlerFunc{
    return func(w http.ResponseWriter,r *http.Request){
        log.Println("WithServerHeader()")
        w.Header().Set("Server","HelloServer v0.0.1")
        h(w,r)
    }
}

func hello(w http.ResponseWriter,r *http.Request){
    log.Printf("Recieved Request %s from %s\n", r.URL.Path, r.RemoteAddr)
    fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}

func main(){
    http.HandleFunc("/v1/hello",WithServerHeader(hello))
    err := http.ListenAndServe(":8080",nil)
    if err != nil{
        log.Fatal("ListenAndServe:",err)
    }
}

```

```go
package main
import (
    "fmt"
    "log"
    "net/http"
    "strings"
)
func WithServerHeader(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithServerHeader()")
        w.Header().Set("Server", "HelloServer v0.0.1")
        h(w, r)
    }
}
func WithAuthCookie(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithAuthCookie()")
        cookie := &http.Cookie{Name: "Auth", Value: "Pass", Path: "/"}
        http.SetCookie(w, cookie)
        h(w, r)
    }
}
func WithBasicAuth(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithBasicAuth()")
        cookie, err := r.Cookie("Auth")
        if err != nil || cookie.Value != "Pass" {
            w.WriteHeader(http.StatusForbidden)
            return
        }
        h(w, r)
    }
}
func WithDebugLog(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithDebugLog")
        r.ParseForm()
        log.Println(r.Form)
        log.Println("path", r.URL.Path)
        log.Println("scheme", r.URL.Scheme)
        log.Println(r.Form["url_long"])
        for k, v := range r.Form {
            log.Println("key:", k)
            log.Println("val:", strings.Join(v, ""))
        }
        h(w, r)
    }
}
func hello(w http.ResponseWriter, r *http.Request) {
    log.Printf("Recieved Request %s from %s\n", r.URL.Path, r.RemoteAddr)
    fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}
func main() {
    http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))
    http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))
    http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```
上面代码中使用到了修饰模式，WithServerHeader()函数就是一个Decorator，其传入一个http.HandlerFunc，然后返回一个改写的版本。

### 多个修饰器的Pipeline
如果需要decorator比较多的话，代码会比较难看了。下面重构以下，需要先写一个工具函数，用来遍历并调用各个decorator：
```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc

func Handler(h http.HandlerFunc,decors ..HttpHandlerDecorator)http.HandlerFunc{
    for i := range decors{
        d := decors[len(decors)-1-i]
        h = d(h)
    }
    return h
}

http.HandleFunc("/v4/hello",Handler(hello,WithServerHeader,WithBasicAuth,WithDebugLog))
```
这样代码更易读了一些，pipeline的功能也就出来了。

### 泛型修饰器
```go
func Decorator(decoPtr,fn interface{})(err error){
    var decoratedFunc,targetFunc reflect.Value
    decoratedFunc = reflect.ValueOf(decoPtr).Elem()
    targetFunc = reflect.ValueOf(fn)

    v := reflect.MakeFunc(targetFunc.Type(),
            func(in []reflect.Value)(out []reflect.Value){
                fmt.Println("before")
                out = targetFunc.Call(in)
                fmt.Println("after")
                return
            })
    decoratedFunc.Set(v)
    return
}

func foo(a,b,c int)int{
    fmt.Printf("%d,%d,%d\n",a,b,c)
    return a + b + c
}

func bar(a,b string) string{
    fmt.Printf("%s,%s\n",a,b)
    return a + b
}

type MyFoo func(int,int,int) int
var myFoo MyFoo
Decorator(&myFoo,foo)
myFoo(1,2,3)
```

使用Decorator()时，还需要先声明一个函数签名，如果不想声明，也可以这样：
```go
mybar := bar
Decorator(&mybar,bar)
mybar("hello,","world!")
```

# Pipeline

## HTTP处理
在上节修饰器中有过一个示例用到Pipeline

## Channel管理
当然，如果要写出一个泛型的Pipeline框架并不容易，而使用Go Generation，但是别忘了go的goroutine和channel这两个神器完全可以被用来构造这种编程。

### Channel转发函数
首先需要一个echo()函数，其会把一个整型数组放到一个Channel中，并返回这个Channel：
```go
func echo(nums []int) <-chan int{
    out := make(chan int)
    go func(){
        for _,n := range nums{
            out <- n
        }
        close(out)
    }()
    return out
}
```

### 平方函数
```go
func sq(in <-chan int) <-chan int{
    out := make(chan int)
    go func(){
        for n := range in{
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

### 过滤奇数函数
```go
func odd(in <-chan int) <-chan int{
    out := make(chan int)
    go func(){
        for n := range in{
            if n%2 != 0{
                out <- in
            }
        }
        close(out)
    }()
    return out
}
```

### 求和函数
```go
func sum(in <-chan int) <-chan int{
    out := make(chan int)
    go func(){
        var sum = 0
        for n := range in{
            sum += n
        }
        out <- sum
        close(out)
    }()
    return out
}
```
可以通过之前的Map/Reduce编程模式或是go generation的方式来合并以下
```go
var nums = []int{1,2,3,4,5,6,7,8,9}
for n := range sum(sq(odd(echo(nums)))){
    fmt.Println(n)
}
```
上面的代码类似于执行Unix/Linux命令：echo $nums | sq | sum
如果不想有那么多的函数嵌套，可以使用一个代理函数来完成：
```go
type EchoFunc func ([]int)(<-chan int)
type PipeFunc func (<-chan int)(<-chan int)
func pipeline(nums []int,echo EchoFunc,pipeFns ...PipeFunc)<-chan int{
    ch := echo(nums)
    for i := range pipeFns{
        ch = pipeFns[i](ch)
    }
    return ch
}

var nums = []int{1,2,3,4,5,6,7,8,9}
for n := range pipeline(nums,gen,odd,sq,sum){
    fmt.Println(n)
}
```

### Fan in/out
动用goroutine和channel还有一个好处，就是可以写出一对多，或多对一的pipeline，也就是Fan in/out。

我们想通过并发的方式来对一个很长的数组中的质数进行求和运算，我们想先把数组分段求和，然后再把其集中起来。
```go
func makeRange(min,max int) []int{
    a := make([]int,max-min+1)
    for i := range a {
        a[i] = min + i
    }
    return a 
}
func is_prime(value int) bool{
    for i := 2;i <=int(math.Floor(float64(value) / 2));i++{
        if value%i == 0{
            return false
        }
    }
    return value > 1
}

func prime(in <-chan int) <-chan int{
    out := make(chan int)
    go func(){
        for n := range in{
            if is_prime(n){
                out <- n
            }
        }
        close(out)
    }()
    return out
}

func merge(cs []<-chan int) <-chan int{
    var wg sync.WaitGroup
    out := make(chan int)

    wg.Add(len(cs))
    for _,c := range cs{
        go func(c <-chan int){
            for n := range c{
                out <- n
            }
            wg.Done()
        }(c)
    }
    go func(){
        wg.Wait()
        close(out)
    }()
    return out
}

func main(){
    nums := makeRange(1,10000)
    in := echo(nums)

    const nProcess = 5
    var chans [nProcess]<-chan int
    for i := range chans{
        chans[i] = sum(prime(in))
    }

    for n := range sum(merge(chans[:])){
        fmt.Println(n)
    }
}
```
整个程序结构如下图所示：
![3](img/goProgrammingMode/3.png)
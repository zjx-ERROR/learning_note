<!--
 * @Author: zhangjiaxi
 * @Date: 2021-02-25 10:17:38
 * @LastEditors: zhangjiaxi
 * @LastEditTime: 2021-02-25 16:46:02
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
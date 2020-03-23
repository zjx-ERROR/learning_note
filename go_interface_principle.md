# go_interface_principle

GoLang可以通过定义接口，将具体的实现和调用完全分离，其本质就是引入一个中间层对不同的模块进行解耦，上层的模块就不需要依赖某一个具体的实现，而是只需要依赖定义好的接口。

interface底层上是分别由两个struct实现：iface和eface。eface表示empty interface，不包含任何方法，iface表示non-empty interface，即包含方法的接口。从概念上来讲，iface和eface均由两部分组成：type和value，type表示interface的类型描述，主要提供concrete type相关的信息，value指向interface绑定的具体数据。

具体类型实例传递给接口称为接口的实例化，这里有个地方值得注意，interface变量默认值为nil，需要初始化后才有意义。

## eface
空接口eface结构比较简单，由两个属性构成，一个是类型信息_type，一个是数据信息。其数据结构声明如下：
```go
type eface struct{
    _type *_type
    data unsafe.Pointer
}
```
其中_type是GoLang语言中所有类型的公共描述，GoLang几乎所有的数据结构都可以抽象成_type，是所有类型的公共描述，type负责决定data应该如何解释和操作，type的结构代码如下：
```go
type _type struct{
    size uintptr
    ptrdata uintptr
    hash uint32
    tflag tflag
    align uint8
    fieldalign uint8
    kind uint8
    alg *typeAlg
    gcdata *byte
    str nameOff
    ptrToThis typeOff
}
```
data表示指向具体的实例数据，由于GoLang的参数传递规则为值传递，如果希望可以通过interface对实例数据修改，则需要传入指针，此时data指向的是指针的副本，但指针指向的实例地址不变，仍然可以对实例数据产生修改。
## iface
iface表示non-empty interface的数据结构，非空接口初始化的过程就是初始化一个iface类型的结构，其中data的作用通eface的相同。
```go
type iface struct{
    tab *itab
    data unsafe.Pointer
}
```
iface结构中最重要的是itab结构，每一个itab都占32字节的空间。itab可以理解为pair<interafce type,concrete type>。itab里面包含了interface的一些关键信息，比如method的具体实现。
```go
type itab struct{
    inter *interfacetype
    _type *_type
    link *itab
    bad int32
    hash int32
    fun [1]uintptr
}

type interfacetype struct{
    typ _type
    pkgpath name
    mhdr []imethod
}

type imethod struct{
    name nameOff
    ityp typeOff
}
```

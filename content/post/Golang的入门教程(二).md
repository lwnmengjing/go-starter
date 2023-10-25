---
title: "Golang的入门教程(二)"
date: 2023-10-25T10:13:13+08:00
Description: ""
Tags: [k8s]
Categories: [k8s]
DisableComments: false
---
## 搭建开发环境
[语言安装包](https://studygolang.com/dl)

[开发IDE goland](https://www.jetbrains.com/go/)

<!--more-->
配置GOROOT
```shell
export GOROOT=go安装目录/bin
```
配置GOPROXY 
```shell
export GOPROXY=https://goproxy.cn 
#或 
export GOPROXY=https://goproxy.cn
```

## Go语言基本语法与使用
### 数据类型分
- 整型： 
  - 按长度分为: int8 int16 int32 int64
  - 还有对应的无符号整型:uint8 uint16 uint32 uint64
- 浮点型:
  - float32:最大范围约为 3.4e38,可以使用常量定义:math.MaxFloat32
  - float64:最大范围约为 1.8e308,可以使用常量定义:math.MaxFloat64
- 布尔型:
  - 布尔型数据只有true和false
  
    `注意⚠️: Go语言中不允许将整型强制转换为布尔型`
- 字符串:
  - 字符串的值为双引号中的内容,可以在Go语言的源码中直接添加非ASCII码字符
- 切片(slice)--能动态分配空间
  - 切片是一个拥有相同类型元素的可变长度序列,切片的声明方式如下:
    > var name []T
    > 其中T代表切片元素类型,可以是整型,浮点型,布尔型,切片,map,函数等.
    > 切片的元素使用"[]"进行访问,在方括号中提供切片的索引即可访问元素,索引的范围从0开始,切不超过切片的最大容量
```cgo
a := make([]int, 3)//创建一个容量为3的整型切片
a[0] = 1//为切片元素赋值
a[1] = 2
a[2] = 3
字符串也可以按切片的方式进行操作:
str := "hello world"
fmt.Println(str[6:])
代码输出如下:
world
```    
### 声明变量
```cgo
    var a int//声明一个整型类型的变量,可以保存整数数值
    var b string//声明一个字符串类型的变量
    var c []float32//声明一个32位浮点切片类型的变量,浮点切片表示由多个浮点类型组成的数据结构
    var d func() bool//声明一个返回值为布尔类型的函数变量,这种形式一般用于回调函数,即将函数以变量的形式保存下来,在需要的时候重新调用这个函数
    var e struct{//声明一个结构体变量,拥有一个整型的x字段
        x int
    }
```
  - 标准式
    var 变量名 变量类型
  - 批量式
```cgo
var (
a int 
b string
c []float32
d func() bool
e struct{
    x int
    }
)
```
### 初始化变量
  - 标准格式
```cgo
var 变量名 类型 = 表达式
例如:游戏中玩家的血量初始值为100. 可以这样写:
var hp int = 100
这句话中,hp为变量名 类型为int hp的初始值为100
```
  - 编译器推导类型的格式
```cgo
例如:
var hp = 100
```
  - 短变量声明并初始化
```cgo
hp := 100
//ps:如果hp被声明过再使用":="时编译器会报错.代码如下"
var hp = 100
hp := 100
//编译报错如下
//no new variables on left side of :=
//提示, 在:=的左边没有新变量出现,意思就是":="的左边变量已经被声明了.短变量声明的形式在开发中例子比较多,比如:
conn, err := net.Dial("tcp","127.0.0.1:8080")
//net.Dial提供按指令协议和地址发起网络链接,这个函数有两个返回值,一个是链接对象,一个是err对象.如果是标准格式将会变成:
var coon net.Conn
var err error
conn, err = net.Dial("tcp","127.0.0.1.8080")
 //因此,短变量声明并初始化的格式在开发中使用比较普遍
```
另外,在多个短变量声明和赋值中,至少有一个新声明的变量出现在左值中,即便其他变量名可能是重复声明的,编译器也不会报错,代码如下:
```cgo
conn, err := net.Dial("tcp","127.0.0.1:8080")
conn2, err := net.Dial("tcp","127.0.0.1:8080")
```
  - 多个变量同时赋值
    编程最简单的算法之一,莫过于变量交换,传统方法编写变量交换代码如下:
    ```cgo
    var a int  = 100
    var b int  = 200
    var t int
    t = a
    a = b
    b = t
    fmt.Println(a,b)
    //但是计算机内存非常"精贵",所以大牛们就发明了另外一种算法
    var a int = 100
    var b int = 200
    a = a ^ b
    b = b ^ a
    fmt.Println(a,b)

    //到了Go语言时,内存不再是紧缺资源,而且写法可以更简单.使用Go的"多重赋值"特性,可以轻松完成变量交换任务;
    var a int = 100
    var b int = 200
    b, a = a, b
    fmt.Println(a,b)
    ```
### 匿名变量--没有名字的变量
在使用多重赋值时,如果不需要在左值中接收变量,可以使用匿名变量.
匿名变量的表现是一个"_"下划线,使用匿名变量时,只需要在变量声明的地方使用下划线替换即可
```cgo
func GetData() (int,int) {
    return 100, 200
}
a, _ := GetData()

_, b := GetData()

fmt.Println(a,b)
```
`匿名函数不占用命名空间, 不会分配内存.匿名变量与匿名变量之前也不会因为多次声明而无法使用!`

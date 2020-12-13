

**math.sqrt()**

```go
package main

import (
   "fmt"
   "math"
)

func main(){
   /* 声明函数变量 */
   getSquareRoot := func(x float64) float64 {
      return math.Sqrt(x)
   }

   /* 使用函数 */
   fmt.Println(getSquareRoot(9))

}
```









## go编译原理

**1. 词法与语法分析**

**2. 类型检查**

对整棵抽象语法树的遍历，我们在每个节点上都会对当前子树的类型进行验证。源代码中已经不存在语法和类型错误。

**3. 中间代码生成**

[`cmd/compile/internal/gc.compileFunctions`](https://draveness.me/golang/tree/cmd/compile/internal/gc.compileFunctions) 编译整个 Go 语言项目中的全部函数，编译队列，并发执行的 Goroutine 会将所有函数对应的抽象语法树转换成中间代码。

中间代码使用了 SSA 的特性

**4. 机器码生成**

 [`src/cmd/compile/internal`](https://github.com/golang/go/tree/master/src/cmd/compile/internal) 目录中包含了很多机器码生成相关的包（amd64、arm、arm64、mips、mips64、ppc64、s390x、x86 和 wasm）， 目录中包含了很多机器码生成相关的包，不同类型的 CPU 分别使用了不同的包生成机器码。

 **WebAssembly（Wasm）**：一种在栈虚拟机上使用的二进制指令格式，它的设计的主要目标就是在 Web 浏览器上提供一种具有高可移植性的目标语言。

Go 语言经过编译还可以运行在几乎全部的主流机器上，不过它的兼容性在除 Linux 和 Darwin 之外的机器上可能还有一些问题，例如：Go Plugin 至今仍然不支持 Windows[8](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-compile-intro/#fn:8)。



## 数据结构

### 数组

Go 语言数组在初始化之后大小就无法改变，存储元素类型相同、但是大小不同的数组类型在 Go 语言看来也是完全不同的，只有两个条件都相同才是同一类型。

```go
[10]int
[200]interface{}

```

**初始化**：

如果数组中元素的个数小于或者等于 4 个，那么所有的变量会直接在栈上初始化，如果数组元素大于 4 个，变量就会在静态存储区初始化然后拷贝到栈上

```go
arr1 := [3]int{1, 2, 3}
arr2 := [...]int{1, 2, 3} // Go 语言会在编译期间通过源代码推导数组的大小
```

1. 上线推导

通过遍历元素的方式来计算数组中元素的数量。

所以我们可以看出 `[...]T{1, 2, 3}` 和 `[3]T{1, 2, 3}` 在运行时是完全等价的，`[...]T` 这种初始化方式也只是 Go 语言为我们提供的一种语法糖，当我们不想计算数组中的元素个数时可以通过这种方法减少一些工作量。

2. 语句转化

* 当元素数量小于或者等于 4 个时，会直接将数组中的元素放置在栈上；将原有的初始化语句 `[3]int{1, 2, 3}` 拆分成一个声明变量的表达式和几个赋值表达式，这些表达式会完成对数组的初始化

* 当元素数量大于 4 个时，会将数组中的元素放置到静态区并在运行时取出；

但是如果当前数组的元素大于四个，[`cmd/compile/internal/gc.anylit`](https://draveness.me/golang/tree/cmd/compile/internal/gc.anylit) 会先获取一个唯一的 `staticname`，然后调用 [`cmd/compile/internal/gc.fixedlit`](https://draveness.me/golang/tree/cmd/compile/internal/gc.fixedlit) 函数在静态存储区初始化数组中的元素并将临时变量赋值给数组：

```go
var arr [5]int
statictmp_0[0] = 1
statictmp_0[1] = 2
statictmp_0[2] = 3
statictmp_0[3] = 4
statictmp_0[4] = 5
arr = statictmp_0
```

**访问和赋值**

数组在内存中都是一连串的内存空间，我们通过指向数组开头的指针、元素的数量以及元素类型占的空间大小表示数组。

内存空间

数组访问越界是非常严重的错误。数组和字符串的一些简单越界错误都会在编译期间发现，但是如果使用变量去访问数组或者字符串时，编译器就无法提前发现错误。

### 切片

切片，即动态数组。其长度并不固定，我们可以向切片中追加元素，它会在容量不足时自动扩容。

切片类型的声明方式与数组有一些相似，不过由于切片的长度是动态的，所以声明时只需要指定切片中的元素类型：

```go
[]int
[]interface{}
```





### 命令层面

注：

1. 当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头，如：Group1，那么使用这种形式的标识符的对象就可以被外部包的代码所使用（客户端程序需要先导入这个包），这被称为导出（像面向对象语言中的 public）；标识符如果以小写字母开头，则对包外是不可见的，但是他们在整个包的内部是可见并且可用的（像面向对象语言中的 protected ）。
2. 需要注意的是 **{** 不能单独放在一行，所以以下代码在运行时会产生错误：
3. 标识符 第一个字符必须是字母或下划线而不能是数字。

**编译命令**

1. ```shell
   $ go run hello.go
   Hello, World!
   ```

   ```shell
   $ go build hello.go 
   $ ls
   hello    hello.go
   $ ./hello 
   Hello, World!
   ```

**数据类型**

| 1    | **布尔型** 布尔型的值只可以是常量 true 或者 false。一个简单的例子：var b bool = true。 |
| ---- | :----------------------------------------------------------- |
| 2    | **数字类型** 整型 int 和浮点型 float32、float64，Go 语言支持整型和浮点型数字，并且支持复数，其中位的运算采用补码。 |
| 3    | **字符串类型:** 字符串就是一串固定长度的字符连接起来的字符序列。Go 的字符串是由单个字节连接起来的。Go 语言的字符串的字节使用 UTF-8 编码标识 Unicode 文本。 |
| 4    | **派生类型:** 包括：(a) 指针类型（Pointer）(b) 数组类型(c) 结构化类型(struct)(d) Channel 类型(e) 函数类型(f) 切片类型(g) 接口类型（interface）(h) Map 类型 |

数字类型

int（有符号、uint（无符号 、**uintptr**（无符号整型，用于存放一个指针）、float（浮点数）、complex(虚数)。

### 变量

**声明变量**

```
var identifier1, identifier2 type
var a type
```

**未初始化值变量**

- 数值类型（包括complex64/128）为 **0**

- 布尔类型为 **false**

- 字符串为 **""**（空字符串）

- 以下几种类型为 **nil**：

- ```go
  var a *int
  var a []int
  var a map[string] int
  var a chan int
  var a func(string) int
  var a error // error 是接口
  ```

**未初始化类型**

根据值自行判定变量类型。

**省略var**

这是使用变量的首选形式，但是*它只能被用在函数体内*，而不可以用于全局变量的声明与赋值。使用操作符 := 可以高效地创建一个新的变量，称之为初始化声明。

```go
v_name := value //因为 := 是一个声明语句 应该有新的变量
```

综上：可以将 var f string = "Runoob" 简写为 f := "Runoob"：

```
// 这种因式分解关键字的写法一般用于声明全局变量
var (
    vname1 v_type1
    vname2 v_type2
)
```

**值类型-引用类型**

值类型：i = j 实际上是在内存中将 i 的值进行了拷贝，i变j不变

(Int)i ----->      7

(int)j ----->.     7.  

引用类型 i = j 实际上是对i 地址进行拷贝，因此i j 的值指向同一个内存块 i变j变

注：

1. 声明局部变量必须使用，全局变量可以不使用

2. a,b = b,a 

3. 空白标识符 _ 也被用于抛弃值，如值 5 在：_, b = 5, 7 中被抛弃。

   _ 实际上是一个只写变量，你不能得到它的值。这样做是因为 Go 语言中你必须使用所有被声明的变量，但有时你并不需要使用从一个函数得到的所有返回值。

4. 并行赋值也被用于当一个函数返回多个返回值时，比如这里的 val 和错误 err 是通过调用 Func1 函数同时得到：val, err = Func1(var1)。

### **常量**

const

```
const (
    Unknown = 0
    Female = 1
    Male = 2
)
```

常量可以用len(), cap(), unsafe.Sizeof()函数计算表达式的值。常量表达式中，函数必须是内置函数，否则编译不过

**iota ** a=0, b=1, c=2 

```go
const (
    a = iota
    b = iota
    c = iota
)

const (
            a = iota   //0
            b          //1
            c          //2
            d = "ha"   //独立值，iota += 1
            e          //"ha"   iota += 1
            f = 100    //iota +=1
            g          //100  iota +=1
            h = iota   //7,恢复计数
            i          //8
)

    fmt.Println(a,b,c,d,e,f,g,h,i) //=> (0, 1, 2, ha,ha, 100, 100, 7, 8)

const (
    i=1<<iota  // 1
    j=3<<iota	 //3*2
    k					 //3*2^2
    l					 //3*2^3
)

func main() {
    fmt.Println("i=",i) 
    fmt.Println("j=",j)
    fmt.Println("k=",k)
    fmt.Println("l=",l)
```



### 运算符

**逻辑算符** && || !

**位运算符**  & 与, | 或 , 和 ^ 异或 

左移右移

```go
  var a uint = 60      /* 60 = 0011 1100 */  
  c = a << 2     /* 240 = 1111 0000 */
   fmt.Printf("第四行 - c 的值为 %d\n", c )

  c = a >> 2     /* 15 = 0000 1111 */
   fmt.Printf("第五行 - c 的值为 %d\n", c )
```

**其他**

| &    | 返回变量存储地址 | &a; 将给出变量的实际地址。 |
| ---- | ---------------- | -------------------------- |
| *    | 指针变量。       | *a; 是一个指针变量         |



### 条件语句

```go
if a < 20 {
       /* 如果条件为 true 则执行以下语句 */
       fmt.Printf("a 小于 20\n" )
   }

if 布尔表达式 {
   /* 在布尔表达式为 true 时执行 */
} else {
  /* 在布尔表达式为 false 时执行 */
}

switch var1 {
    case val1:
        ...
    case val2:
        ...
    default:
        ...
}

select {
    case communication clause  :
       statement(s);      
    case communication clause  :
       statement(s);
    /* 你可以定义任意数量的 case */
    default : /* 可选 */
       statement(s);
}
```

sselect

- 每个 case 都必须是一个通信

- 所有 channel 表达式都会被求值

- 所有被发送的表达式都会被求值

- 如果任意某个通信可以进行，它就执行，其他被忽略。

- 如果有多个 case 都可以运行，Select 会随机公平地选出一个执行。其他不会执行。

  否则：

  1. 如果有 default 子句，则执行该语句。
  2. 如果没有 default 子句，select 将阻塞，直到某个通信可以运行；Go 不会重新对 channel 或值进行求值。

```go
func main() {
	var c1, c2, c3 chan int //channal
	var i1, i2 int
	select {
	case i1 = <-c1:
		fmt.Printf("received ", i1, " from c1\n")
	case c2 <- i2:
		fmt.Printf("sent ", i2, " to c2\n")
	case i3, ok := (<-c3):  // same as: i3, ok := <-c3
		if ok {
			fmt.Printf("received ", i3, " from c3\n")
		} else {
			fmt.Printf("c3 is closed\n")
		}
	default:
		fmt.Printf("no communication\n")
	}
}
```

### 循环语句

```go
for i := 0; i <= 10; i++ {
           sum += i
       }
for sum <= 10{
           sum += sum
       }
for {
         sum++ *// 无限循环下去*
       }

 for i, s := range strings {
                fmt.Println(i, s)
        }


numbers := [6]int{1, 2, 3, 5}
for i,x:= range numbers {
        fmt.Printf("第 %d 位 x 的值 = %d\n", i,x)
}  
```

break continue goto

### 函数

```go
func function_name( [parameter list] ) [return_types] {
   函数体
}

func swap(x, y string) (string, string) {
   return y, x
}

func main() {
   a, b := swap("Google", "Runoob")
   fmt.Println(a, b)
}//返回多个值
```

| [值传递](https://www.runoob.com/go/go-function-call-by-value.html) | 值传递是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [引用传递](https://www.runoob.com/go/go-function-call-by-reference.html) | 引用传递是指在调用函数时将实际参数的地址传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。 |

```go
func swap(x *int, y *int) {
   var temp int
   temp = *x    /* 保存 x 地址上的值 */
   *x = *y      /* 将 y 值赋给 x */
   *y = temp    /* 将 temp 值赋给 y */
}
swap(&a, &b)
```

**注意**

1. 函数作为实参

2. 闭包

   Go 语言支持匿名函数，可作为闭包。匿名函数是一个"内联"语句或表达式。匿名函数的优越性在于可以直接使用函数内的变量。

   ```go
   package main
   
   import "fmt"
   
   func getSequence() func() int {
      i:=0
      return func() int {
         i+=1
        return i  
      }
   }
   
   func main(){
      /* nextNumber 为一个函数，函数 i 为 0 */
      nextNumber := getSequence()  
   
      /* 调用 nextNumber 函数，i 变量自增 1 并返回 */
      fmt.Println(nextNumber()) // 1
      fmt.Println(nextNumber()) // 2
      fmt.Println(nextNumber()) // 3
      
      /* 创建新的函数 nextNumber1，并查看结果 */
      nextNumber1 := getSequence()  
      fmt.Println(nextNumber1()) // 1
      fmt.Println(nextNumber1()) // 2
   }
   ```

   

3. 方法

   ```go
   package main
   
   import (
      "fmt"  
   )
   
   /* 定义结构体 */
   type Circle struct {
     radius float64
   }
   
   func main() {
     var c1 Circle
     c1.radius = 10.00
     fmt.Println("圆的面积 = ", c1.getArea())
   }
   
   //该 method 属于 Circle 类型对象中的方法
   func (c Circle) getArea() float64 {
     //c.radius 即为 Circle 类型对象中的属性
     return 3.14 * c.radius * c.radius
   }
   ```

### 数组

声明

```go
b1 := [...]int{1,2,3}
b1 := [10]int{1,2,3}
a = [3][4]int{  
 {0, 1, 2, 3} ,   /*  第一行索引为 0 */
 {4, 5, 6, 7} ,   /*  第二行索引为 1 */
 {8, 9, 10, 11},   /* 第三行索引为 2 */
}
val := a[2][3]
func getAverage(arr []int, size int) float32
{
   var i int
   var avg, sum float32  

   for i = 0; i < size; ++i {
      sum += arr[i]
   }
   avg = sum / size
   return avg;
}
```

### 指针

变量是一种使用方便的占位符，用于引用计算机内存地址。Go 语言的取地址符是 &，放到一个变量前使用就会返回相应变量的内存地址。

声明

```go
var ip *int        /* 指向整型*/
var fp *float32    /* 指向浮点型 */
```

空指针 nil

一个指针变量通常缩写为 ptr。

```go
if(ptr != nil)     /* ptr 不是空指针 */
if(ptr == nil)    /* ptr 是空指针 */
 fmt.Printf("ptr 的值为 : %x\n", ptr  ) //ptr 的值为 : 0
```

```go
// 指针数组
package main

import "fmt"

const MAX int = 3

func main() {
   a := []int{10,100,200}
   var i int
   var ptr [MAX]*int;

   for  i = 0; i < MAX; i++ {
      ptr[i] = &a[i] /* 整数地址赋值给指针数组 */
   }

   for  i = 0; i < MAX; i++ {
      fmt.Printf("a[%d] = %d\n", i,*ptr[i] )
   }
}
```

```go
//指向指针的指针
var ptr **int;
```

### 结构体

```go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}


func main() {

    // 创建一个新的结构体
    fmt.Println(Books{"Go 语言", "www.runoob.com", "Go 语言教程", 6495407})

    // 也可以使用 key => value 格式
    fmt.Println(Books{title: "Go 语言", author: "www.runoob.com", subject: "Go 语言教程", book_id: 6495407})

    // 忽略的字段为 0 或 空
   fmt.Println(Books{title: "Go 语言", author: "www.runoob.com"})
   Books.title = "Go 语言"
   Books.author = "www.runoob.com"
   Books.subject = "Go 语言教程"
   Books.book_id = 6495407
}

func printBook( book Books ) {
   fmt.Printf( "Book title : %s\n", book.title)
   fmt.Printf( "Book author : %s\n", book.author)
   fmt.Printf( "Book subject : %s\n", book.subject)
   fmt.Printf( "Book book_id : %d\n", book.book_id)
}

var struct_pointer *Books
struct_pointer = &Book1
struct_pointer.title
```

### 切片

Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go中提供了一种灵活，功能强悍的内置类型切片("动态数组"),与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。

```go
//定义切片
//1. 使用关键字 make 创建切片：
var slice1 []type = make([]type, len, capacity)
slice1 := make([]type, len) 
make([]T, length, capacity)


//2. 使用字面量初始化新的切片；
s :=[] int {1,2,3 } 
直接初始化切片，[]表示是切片类型，{1,2,3}初始化值依次是1,2,3.其cap=len=3

//3. 用切片或者数组的一部分形成切片
s := arr[:] 
初始化切片s,是数组arr的引用

s := arr[startIndex:endIndex] 
将arr中从下标startIndex到endIndex-1 下的元素创建为一个新的切片

s := arr[startIndex:] 
默认 endIndex 时将表示一直到arr的最后一个元素

s := arr[:endIndex] 
默认 startIndex 时将表示从arr的第一个元素开始

s1 := s[startIndex:endIndex] 
通过切片s初始化切片s1
```

**len() 和 cap() 函数**

切片是可索引的，并且可以由 len() 方法获取长度。 

切片提供了计算容量的方法 cap() 可以测量切片最长可以达到多少。

一个切片在未初始化之前默认为 nil，长度为 0。

将切片理解成一片连续的内存空间加上长度与容量的标识。切片引入了一个抽象层，提供了对数组中部分连续片段的引用，而作为数组的引用，我们可以在运行区间可以修改它的长度和范围。当切片底层的数组长度不足时就会触发扩容，切片指向的数组可能会发生变化，不过在上层看来切片是没有变化的，上层只需要与切片打交道不需要关心数组的变化。

```go
numbers := []int{0,1,2,3,4,5,6,7,8}  
/* 打印子切片从索引  0(包含) 到索引 2(不包含) */
   number2 := numbers[:2]
   printSlice(number2)//len=2 cap=9 slice=[0 1]

   /* 打印子切片从索引 2(包含) 到索引 5(不包含) */
   number3 := numbers[2:5]
   printSlice(number3)//len=3 cap=7 slice=[2 3 4] 
//切片指针指向索引第一个
```

### Range

Go 语言中 range 关键字用于 for 循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素。在数组和切片中它返回元素的索引和索引对应的值，在集合中返回 key-value 对。

```go
//这是我们使用range去求一个slice的和。使用数组跟这个很类似
    nums := []int{2, 3, 4}
    sum := 0
    for _, num := range nums {
        sum += num
    }
    fmt.Println("sum:", sum)
    //在数组上使用range将传入index和值两个变量。上面那个例子我们不需要使用该元素的序号，所以我们使用空白符"_"省略了。有时侯我们确实需要知道它的索引。
    for i, num := range nums {
        if num == 3 {
            fmt.Println("index:", i)
        }
    }
    //range也可以用在map的键值对上。//如果只有一个变量，指key值
    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {
        fmt.Printf("%s -> %s\n", k, v)
    }
		for country := range countryCapitalMap {
        fmt.Println(country, "首都是", countryCapitalMap [country])
    }
    //range也可以用来枚举Unicode字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
    for i, c := range "go" {
        fmt.Println(i, c)
    }
```

### Map

Map 是一种无序的键值对的集合。Map 最重要的一点是通过 key 来快速检索数据，key 类似于索引，指向数据的值。

Map 是一种集合，所以我们可以像迭代数组和切片那样迭代它。不过，Map 是无序的，我们无法决定它的返回顺序，这是因为 Map 是使用 hash 表来实现的。

```go
/* 声明变量，默认 map 是 nil */
var map_variable map[key_data_type]value_data_type

/* 使用 make 函数 */
map_variable := make(map[key_data_type]value_data_type)
```

Nil map没法存储键值对

```go
func main() {
    var countryCapitalMap map[string]string /*创建集合 */
    countryCapitalMap = make(map[string]string)

    /* map插入key - value对,各个国家对应的首都 */
    countryCapitalMap [ "France" ] = "巴黎"
    countryCapitalMap [ "Italy" ] = "罗马"
    countryCapitalMap [ "Japan" ] = "东京"
    countryCapitalMap [ "India " ] = "新德里"

    /*迭代输出key*/
    for country := range countryCapitalMap {
        fmt.Println(country, "首都是", countryCapitalMap [country])
    }
   /*迭代输出key和value*/
  for country, capital := range countryCapitalMap{
    	fmt.Println(country, "首都是", capital)
  }

    /*查看元素在集合中是否存在 */
    capital, ok := countryCapitalMap [ "American" ] /*如果确定是真实的,则存在,否则不存在 */
    /*fmt.Println(capital) */
    /*fmt.Println(ok) */
    if (ok) {
        fmt.Println("American 的首都是", capital)
    } else {
        fmt.Println("American 的首都不存在")
    }
  
  /*delete() */
  delete(countryCapitalMap, "France") 
}
```

### 递归

### 类型转化

不支持隐式转化

```
type_name(expression)
```

### 接口

Go 语言提供了另外一种数据类型即接口，它把所有的具有共性的方法定义在一起，任何其他类型只要实现了这些方法就是实现了这个接口。

```go
/* 定义接口 */
type interface_name interface {
   method_name1 [return_type]
   method_name2 [return_type]
   method_name3 [return_type]
   ...
   method_namen [return_type]
}

/* 定义结构体 */
type struct_name struct {
   /* variables */
}

/* 实现接口方法 */
func (struct_name_variable struct_name) method_name1() [return_type] {
   /* 方法实现 */
}
...
func (struct_name_variable struct_name) method_namen() [return_type] {
   /* 方法实现*/
}
```

在上面的例子中，我们定义了一个接口Phone，接口里面有一个方法call()。然后我们在main函数里面定义了一个Phone类型变量，并分别为之赋值为NokiaPhone和IPhone。然后调用call()方法，输出结果如下：

```go
package main

import (
    "fmt"
)

type Phone interface {
    call()
}

type NokiaPhone struct {
}

func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}

type IPhone struct {
}

func (iPhone IPhone) call() {
    fmt.Println("I am iPhone, I can call you!")
}

func main() {
    var phone Phone

    phone = new(NokiaPhone)
    phone.call()

    phone = new(IPhone)
    phone.call()

}
```

### go并发

Go 语言支持并发，我们只需要通过 go 关键字来开启 goroutine 即可。

goroutine 是轻量级线程，goroutine 的调度是由 Golang 运行时进行管理的。



 同一个程序中的所有 goroutine 共享同一个地址空间。

```go
package main

import (
        "fmt"
        "time"
)

func say(s string) {
        for i := 0; i < 5; i++ {
                time.Sleep(100 * time.Millisecond)
                fmt.Println(s)
        }
}

func main() {
        go say("world")
        say("hello")
}
```

channel

```go
ch := make(chan int)
ch <- v    // 把 v 发送到通道 ch
v := <-ch  // 从 ch 接收数据
           // 并把值赋给 v
```

带缓冲channel

带缓冲区的通道允许发送端的数据发送和接收端的数据获取处于异步状态，就是说发送端发送的数据可以放在缓冲区里面，可以等待接收端去获取数据，而不是立刻需要接收端去获取数据。

不过由于缓冲区的大小是有限的，所以还是必须有接收端来接收数据的，否则缓冲区一满，数据发送端就无法再发送数据了。

**注意**：如果通道不带缓冲，发送方会阻塞直到接收方从通道中接收了值。如果通道带缓冲，发送方则会阻塞直到发送的值被拷贝到缓冲区内；如果缓冲区已满，则意味着需要等待直到某个接收方获取到一个值。接收方在有值可以接收之前会一直阻塞。

```go
package main

import "fmt"

func main() {
    // 这里我们定义了一个可以存储整数类型的带缓冲通道
        // 缓冲区大小为2
        ch := make(chan int, 2)

        // 因为 ch 是带缓冲的通道，我们可以同时发送两个数据
        // 而不用立刻需要去同步读取数据
        ch <- 1
        ch <- 2

        // 获取这两个数据
        fmt.Println(<-ch)
        fmt.Println(<-ch)
}
```

go便利通道 关闭通道

Go 通过 range 关键字来实现遍历读取到的数据，类似于与数组或切片。格式如下：

```go
v, ok := <-ch //如果channel没有数据就是false
```

如果通道接收不到数据后 ok 就为 false，这时通道就可以使用 **close()** 函数来关闭。

```go
package main

import (
        "fmt"
)

func fibonacci(n int, c chan int) {
        x, y := 0, 1
        for i := 0; i < n; i++ {
                c <- x
                x, y = y, x+y
        }
        close(c)
}

func main() {
        c := make(chan int, 10)
        go fibonacci(cap(c), c)
        // range 函数遍历每个从通道接收到的数据，因为 c 在发送完 10 个
        // 数据之后就关闭了通道，所以这里我们 range 函数在接收到 10 个数据
        // 之后就结束了。如果上面的 c 通道不关闭，那么 range 函数就不
        // 会结束，从而在接收第 11 个数据的时候就阻塞了。
        for i := range c {
                fmt.Println(i)
        }
}
```

go 知识导图

https://www.processon.com/view/link/5a9ba4c8e4b0a9d22eb3bdf0#map


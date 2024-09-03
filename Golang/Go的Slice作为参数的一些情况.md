碰到个slice作为参数的有趣的情况
# 被调用函数append不会影响调用传入的slice

```
package main

import (
	"fmt"
)

func add(res []int) {
	res = append(res, 66)
}

func main() {
	res := make([]int, 0, 10)
	add(res)
	fmt.Printf("%v\n", res)
}
```

上面代码打印出的res是空的
避免append操作导致数组扩容，变更底层的数组的指针，这里还预留了空间。


# 被调用函数用索引修改会影响调用传入的slice

```
package main

import (
	"fmt"
)

func add(res []int) {
	res = append(res, 66)
}

func main() {
	res := []int{1}
	add(res)
	fmt.Printf("%v\n", res)
}
```
这里打印出来的就是66了

# 为什么呢
同一个slice，为什么append操作不会修改原slice，但是根据索引修改就会影响原slice呢？

go的slice本质上就是一个结构体，包括底层的数据数组的指针，slice的数据大小，slice的容量。

slice作为参数传入的时候，就会把这个结构体复制一份。底层的数据数组指针是同一个，但是数据大小和容量都会复制一份。


也就是说，在第一个例子中，append操作，会影响复制出来的数据大小，但是原slice的数据大小还是0。

这点可以取底层的数据的方法实现

```
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func add(res []int) {
	res = append(res, 66)
}

func main() {
	res := make([]int, 0, 10)
	add(res)
	fmt.Printf("%v\n", res)
	sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&res))
	// 将指针转换为指向具体类型的指针，并取出第一个元素的值
	firstElement := *(*int)(unsafe.Pointer(sliceHeader.Data))
	fmt.Printf("firstElement=%d\n", firstElement)
}

```
这里打印出来的就是66了，也就是说数据一直都在，只是数据大小一直为0，打印不出来。如果这里main函数再append一下，就会覆盖原有的66了。
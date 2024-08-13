# Golang的包版本管理

## 间接依赖
如果一个程序依赖A、B两个库，同时A、B分别依赖C库的不同版本，此时，程序将会选择较新的版本。
并且库是同一份，即公共库中的全局变量，是只有一份的
比如
```
package main

import (
	"fmt"

	"github.com/jiangbo9510/go_mod_lib"
	"github.com/jiangbo9510/test_go_direct_lib"
)

func main() {
	go_mod_lib.SetCommon("1")
	fmt.Printf("go_mod_lib = %v\n", go_mod_lib.GetCommon())
	fmt.Printf("test_go_direct_lib = %v\n", test_go_direct_lib.GetCommon())
	test_go_direct_lib.SetCommon("2")
	fmt.Printf("test_go_direct_lib = %v\n", test_go_direct_lib.GetCommon())
}

```
对应gomod
```
go 1.18

require (
	github.com/jiangbo9510/go_mod_lib v0.0.2
	github.com/jiangbo9510/test_go_direct_lib v0.0.3
	github.com/lxzan/gws v1.2.12
)

require github.com/jiangbo9510/test_base_lib v0.0.4 // indirect

```

go build后，查看依赖的库
`go version -m test`

可以看到，依赖的是较新的
**那如果新老版本的库不兼容呢？**

```
module main

go 1.18

require (
	github.com/jiangbo9510/go_mod_lib v0.0.3
	github.com/jiangbo9510/test_go_direct_lib v0.0.3
	github.com/lxzan/gws v1.2.12
)

require github.com/jiangbo9510/test_base_lib v0.0.5 // indirect

```
更新gomod为上述，结果还是可以正常运行，并且编译后的bin文件依赖的还是最新的test_base_lib的v0.0.5版本的库


## Go Plugin场景

会直接报错，说库版本不一致
`panic: plugin.Open("lib1"): plugin was built with a different version of package github.com/jiangbo9510/test_base_lib`

主代码
```
package main

import (
	"fmt"
	"plugin"
)

func plugin1() {
	lib1, err1 := plugin.Open("lib1.so")
	if err1 != nil {
		panic(err1)
	}
	symbolSet, err := lib1.Lookup("Lib1Set")
	if err != nil {
		panic(err)
	}

	symbolGet, err := lib1.Lookup("Lib1Get")
	if err != nil {
		panic(err)
	}

	setFunc := symbolSet.(func(s string))
	getFunc := symbolGet.(func() string)
	fmt.Printf("before lib1 = %v\n", getFunc())
	setFunc("lib1")
	fmt.Printf("after lib1 = %v\n", getFunc())
}

func plugin2() {
	lib2, err2 := plugin.Open("lib2.so")
	if err2 != nil {
		panic(err2)
	}
	symbolSet, err := lib2.Lookup("Lib2Set")
	if err != nil {
		panic(err)
	}
	symbolGet, err := lib2.Lookup("Lib2Get")
	if err != nil {
		panic(err)
	}
	setFunc := symbolSet.(func(s string))
	getFunc := symbolGet.(func() string)
	fmt.Printf("before in lib2 = %v\n", getFunc())
	setFunc("lib2")
	fmt.Printf("after in lib2 = %v\n", getFunc())
}

func main() {
	plugin1()
	plugin2()
	plugin1()
}
```

lib1
```
package main

import "github.com/jiangbo9510/test_base_lib"

func Lib1Set(s string) {
	test_base_lib.SetCommon(s, "")
}

func Lib1Get() string {
	return test_base_lib.GetCommon()
}

```

```
module lib1

go 1.18

require github.com/jiangbo9510/test_base_lib v0.0.5

```

lib2

```
package main

import "github.com/jiangbo9510/test_base_lib"

func Lib2Set(s string) {
	test_base_lib.SetCommon(s)
}

func Lib2Get() string {
	return test_base_lib.GetCommon()
}

```

```
module lib2

go 1.18

require github.com/jiangbo9510/test_base_lib v0.0.3

replace github.com/jiangbo9510/test_base_lib => github.com/jiangbo9510/test_base_lib v0.0.4

```


编译上述两个库
`go build -buildmode=plugin -o lib1.so`
`go build -buildmode=plugin -o lib2.so`

运行主文件，就可以看到报错
`panic: plugin.Open("lib1"): plugin was built with a different version of package github.com/jiangbo9510/test_base_lib`
两个库文件不一致，第一个库正常运行，第二个库报错。
同理，so库和主程序共同依赖的库的版本不一致也会报这个错误
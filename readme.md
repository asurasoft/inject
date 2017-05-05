# package inject #

`import "github.com/facebookgo/inject"`


包装反射的注射器。主要考虑到使用依赖注入构建的大型应用程序时通常将涉及大量设置对象图标的无聊工作。 这个类库试图通过创建和连接各种对象来接管这个无聊的工作。它的用途是将对象图与某些（可能不完整的）对象进行对接，其中底层类型被标记为注入。 综上，类库将根据依赖填充对象。 它默认使用单例，支持可选的私有实例以及命名实例。


它的工作原理是使用Go的反射软件包。


库的使用模式涉及到 struct 标签。 它需要各种标准库使用的标签格式，如json,xml 等。它涉及以下三种格式之一的标签：

```
`inject:""`
`inject:"private"`
`inject:"dev logger"`
```

第一个无值语法是针对关联类型的单例依赖的常见情况。 第二个触发器创建关联类型的私有实例。 最后一个是要求一个名为 "dev logger" 的依赖关系。

## 自测用例 ##
```
package main


import (
	"math/rand"
	"testing"
	"github.com/facebookgo/inject"
	"fmt"
	"time"
)

type TypeAnswerStruct struct {
	answer  int
	private int
}

func (t *TypeAnswerStruct) Answer() int {
	return t.answer
}

type TypeNestedStruct struct {
	A *TypeAnswerStruct `inject:""`
}

func (t *TypeNestedStruct) Answer() int {
	return t.A.Answer()
}

var c struct {
	A *TypeNestedStruct `inject:"foo"`
}

var v struct {
	A *TypeAnswerStruct `inject:""`
	B *TypeNestedStruct `inject:""`
}

func init() {
	// we rely on math.Rand in Graph.Objects() and this gives it some randomness.
	//使用 math.Rand 在 Graph.Objects() 给出一些随机性
	rand.Seed(time.Now().UnixNano())
}

//简单注入操作
func TestInjectSimple(t *testing.T) {
	if err := inject.Populate(&v); err != nil {
		t.Fatal(err)
	}

	fmt.Println(v.B.A.Answer());
}

//测试实例别名依赖
func TestNamedInstanceWithDependencies(t *testing.T) {
	var g inject.Graph
	a1 := &TypeAnswerStruct{answer:100}
	a2 := &TypeNestedStruct{}
	if err := g.Provide(&inject.Object{Value: a1},&inject.Object{Value: a2, Name: "foo"}); err != nil {
		t.Fatal(err)
	}

	var c struct {
		A *TypeNestedStruct `inject:"foo"`
	}
	if err := g.Provide(&inject.Object{Value: &c}); err != nil {
		t.Fatal(err)
	}

	if err := g.Populate(); err != nil {
		t.Fatal(err)
	}

	if c.A.A == nil {
		t.Fatal("c.A.A was not injected")
	}

	fmt.Println(c.A.A.Answer());
}
```

## 官方demo ##

```
package main

import (
    "fmt"
    "net/http"
    "os"

    "github.com/facebookgo/inject"
)

// 在HomePlanetRenderApp上构造两个API类
type HomePlanetRenderApp struct {

    //下面的标签向注入库指示这些字段
    //表示有资格注射。 他们没有指定任何选项，并将会
    //导致为每个API创建的单例实例。

    NameAPI   *NameAPI   `inject:""`
    PlanetAPI *PlanetAPI `inject:""`
}

func (a *HomePlanetRenderApp) Render(id uint64) string {
    return fmt.Sprintf(
        "%s is from the planet %s.",
        a.NameAPI.Name(id),
        a.PlanetAPI.Planet(id),
    )
}

// 伪造NameAPI
type NameAPI struct {
    //在PlanetAPI中下面和这里下面我们将标签添加到接口值中。该值不能自动创建（按定义），因此必须显式提供给图形。
    HTTPTransport http.RoundTripper `inject:""`
}

func (n *NameAPI) Name(id uint64) string {
    return "Spock"
}

// Our fake Planet API.
type PlanetAPI struct {
    HTTPTransport http.RoundTripper `inject:""`
}

func (p *PlanetAPI) Planet(id uint64) string {
    return "Vulcan"
}

func main() {
    // 通常，应用程序将只有一个对象图形，您将创建它并在主要功能中使用它：
    var g inject.Graph

    //我们向我们提供了两个“种子”对象，一个是我们的空的
    //我们自动填充的HomePlanetRenderApp实例，并且需要满足PlanetAPI对HTTPTransport依赖性。http.RoundTripper是接口所以我们只需要提供一个派生类即可：

    var a HomePlanetRenderApp
    err := g.Provide(
        &inject.Object{Value: &a},
        &inject.Object{Value: http.DefaultTransport},
    )
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }

    //这里Populate调用正在创建NameAPI＆
    // PlanetAPI，并将HTTPTransport设置为
    // http.DefaultTransport提供如下：
    
    if err := g.Populate(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }

    //有一个简化的case的简写API，它结合了
    //以上三个调用可用作inject.Populate：
    //inject.Populate（＆a，http.DefaultTransport）
    //上述API显示了还允许使用的底层API更复杂场景的命名实例。
    fmt.Println(a.Render(42))
}
```

## func Populate ##
`func Populate(values ...interface{}) error`

使用给定的不完整对象值填补对象图形,填充简单的手段。

## type Graph ##
`type Graph struct {
     Logger Logger //可选，将触发调试日志记录。
     //包含过滤或未导出的字段
 }
`
对象图

### func (*Graph) Objects ###
`func (g *Graph) Objects() []*Object`

对象返回所有已知对象，命名以及未命名。 返回的元素不是稳定的顺序。

### func (*Graph) Populate ###

`func (g *Graph) Populate() error`

填充不完整的对象。


### func (*Graph) Provide ###
`func (g *Graph) Provide(objects ...*Object) error`

向Graph提供对象。 对象文档描述了各个领域的影响。


## type Logger ##

```
type Logger interface {
    Debugf(format string, v ...interface{})
}
```

记录器允许简单记录，因为注入遍历并填充对象图。

## type Object ##

```
type Object struct {
    Value    interface{}
    Name     string             //可选
    Complete bool               //如果为true，则值将被视为完成
    Fields   map[string]*Object //填充注入的字段名称及其对应的*对象。
    //包含过滤或未导出的字段
}
```
图的结构

## func (*Object) String ##

```func (o *Object) String() string```

字符串表示适合消费。


# fork 的gin

## 最终效果
```
uid在前端可以传 字符串,int 。如字符串无法转换成int会解析失败

post json 传参数
{
"uid":["111",222]
}

gin解析
type AA struct {
	UID []int `json:"uid"`
}
var a AA
err := c.ShouldBind(&a)
```

## 为什么要改

### 1. string to int
因为老项目中使用的动态语言php，好多uid的类型是 123 也是"123" 

gin解析到struct有问题，因为不能更改所有老项目代码，所以只能改gin

找到gin的json decoder 设置在 gin/internal/json 这个“内部包”中，无法修改，只能fork了

### 2. time.Time format
time.Time 格式化成 "2006-01-02 15:03:04"

## 修改内容

实际上修改的是标准库的 encoding/json包的内容

但是为了不影响标准库，我拷贝了一份json放在了gin/internal/json中

然后让gin使用我修改后的json包。

代码修改：

```
包名github.com/gin-gonic/gin 替换成 github.com/chenxiao1990/gin
```

```
gin/internal/json/json.go中
import "github.com/chenxiao1990/gin/internal/json/json"

```

```
gin/internal/json/json/decode.go  946行增加

if len(s) == 0 {
	v.SetInt(0)
} else {
	n, err := strconv.ParseInt(string(s), 10, 64)
	if err != nil || v.OverflowInt(n) {
		d.saveError(&UnmarshalTypeError{Value: "string " + string(s), Type: v.Type(), Offset: int64(d.readIndex())})
		break
	}
	v.SetInt(n)
}
```

```
为了gin输出 time.Time的格式为 "2006-01-02 15:04:05"

gin/internal/json/json/encode.go  456行增加

t := reflect.TypeOf(m)
s := t.Name()
s2 := t.PkgPath()
if s2+"."+s == "time.Time" {
	timeFormart := "2006-01-02 15:04:05"
	b = make([]byte, 0, len(timeFormart)+2)
	b = append(b, '"')
	b = m.(time.Time).AppendFormat(b, timeFormart)
	b = append(b, '"')
}
```

## 后续使用可能会遇到的问题

比如你引用了 "github.com/DeanThompson/ginpprof"

会报类型错误

这种 github.com/chenxiao1990 与 github.com/gin-gonic 类型错误 都可以使用下面的方法强制转换

```
引用原始的gin  
import  ygin "github.com/gin-gonic/gin"

GRouter = gin.Default()
up := unsafe.Pointer(GRouter)
ginpprof.Wrapper((*ygin.Engine)(up))
```
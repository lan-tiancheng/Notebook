# 🧠 Golang 反射机制（Reflect）

反射是 Go 中在运行时动态地**获取类型信息**（type）和**操作变量的值**（value）的一种机制。

------

## 一、反射的核心类型

反射由 `reflect` 包提供，最重要的两个函数是：

| 函数                 | 返回类型        | 作用                                   |
| -------------------- | --------------- | -------------------------------------- |
| `reflect.TypeOf(x)`  | `reflect.Type`  | 获取变量的类型信息（结构、方法签名等） |
| `reflect.ValueOf(x)` | `reflect.Value` | 获取变量的值信息（可以读/写/调用）     |

> 一般搭配使用：`TypeOf` 看结构，`ValueOf` 操作数据。

------

## 二、通过反射获取类型

```go
package main

import (
	"fmt"
	"reflect"
)

func GetType(obj any) {
	t := reflect.TypeOf(obj)

	switch t.Kind() {
	case reflect.String:
		fmt.Println("string")
	case reflect.Int:
		fmt.Println("int")
	case reflect.Struct:
		fmt.Println("struct")
	default:
		fmt.Println("unknown type:", t.Kind())
	}
}

func main() {
	GetType("23")
	GetType(23)
	GetType(struct{ name string }{"mike"})
}
```

✅ **说明：**

- `TypeOf()` 返回的是一个接口类型 `reflect.Type`；
- `t.Kind()` 返回基础类型（`int`, `string`, `struct` 等）；
- 使用 `switch` 匹配类型时，需要从 `reflect` 包取常量。

------

## 三、通过反射获取值

```go
package main

import (
	"fmt"
	"reflect"
)

func GetValue(obj any) {
	v := reflect.ValueOf(obj)

	switch v.Kind() {
	case reflect.String:
		fmt.Println("String:", v.String())
	case reflect.Bool:
		fmt.Println("Bool:", v.Bool())
	case reflect.Int:
		fmt.Println("Int:", v.Int())
	default:
		fmt.Println("Unsupported type:", v.Kind())
	}
}

func main() {
	GetValue("hello world")
	GetValue(true)
	GetValue(12)
}
```

✅ **说明：**

- 不同类型有不同的取值方法：
  - `v.String()`、`v.Int()`、`v.Bool()`
- 若类型不匹配会 panic，建议配合 `Kind()` 检查类型后再调用。

------

## 四、通过反射修改值

> ⚠️ 修改值必须传入**指针**，否则反射无法修改原变量。

```go
package main

import (
	"fmt"
	"reflect"
)

func SetVal(obj any, val any) {
	v1 := reflect.ValueOf(obj)
	v2 := reflect.ValueOf(val)

	// 必须是指针
	if v1.Kind() != reflect.Ptr {
		fmt.Println("Must be a pointer")
		return
	}

	// 解引用
	v1 = v1.Elem()

	// 类型必须匹配
	if v1.Kind() != v2.Kind() {
		fmt.Println("Type mismatch")
		return
	}

	switch v1.Kind() {
	case reflect.String:
		v1.SetString(v2.String())
	case reflect.Int:
		v1.SetInt(v2.Int())
	case reflect.Bool:
		v1.SetBool(v2.Bool())
	default:
		fmt.Println("Unsupported type:", v1.Kind())
	}
}

func main() {
	name1 := "Mike"
	SetVal(&name1, "Amy")
	fmt.Println(name1)

	age1 := 20
	SetVal(&age1, 21)
	fmt.Println(age1)

	judge := false
	SetVal(&judge, true)
	fmt.Println(judge)
}
```

✅ **要点：**

- `reflect.ValueOf(obj).Elem()` 解引用；
- 修改必须是**可寻址（addressable）**的值；
- 若类型不同，会 panic 或报错。

------

## 五、反射与结构体

### 1️⃣ 获取结构体字段与标签

```go
package main

import (
	"fmt"
	"reflect"
)

type Student struct {
	Name  string `json:"name"`
	Age   int
	IsMan bool
}

func ParseStruct(obj any) {
	t := reflect.TypeOf(obj)
	v := reflect.ValueOf(obj)

	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		value := v.Field(i)
		fmt.Printf("%s = %v, tag = %q\n", field.Name, value, field.Tag.Get("json"))
	}
}

func main() {
	s := Student{"Mike", 20, true}
	ParseStruct(s)
}
```

✅ **说明：**

- `t.NumField()` 获取字段数量；
- `t.Field(i)` 获取字段元信息；
- `v.Field(i)` 获取字段值；
- `Tag.Get("json")` 获取标签中指定键的值。

------

### 2️⃣ 根据标签修改结构体字段

```go
package main

import (
	"fmt"
	"reflect"
	"strings"
)

type User struct {
	Name1 string `upper:"y"`
	Name2 string
}

func SetUpper(obj any) {
	v := reflect.ValueOf(obj).Elem()
	t := reflect.TypeOf(obj).Elem()

	for i := 0; i < v.NumField(); i++ {
		fieldVal := v.Field(i)
		tag := t.Field(i).Tag.Get("upper")
		if tag == "y" && fieldVal.Kind() == reflect.String {
			fieldVal.SetString(strings.ToUpper(fieldVal.String()))
		}
	}
}

func main() {
	u := User{"mike", "mike"}
	SetUpper(&u)
	fmt.Println(u)
}
```

✅ **说明：**

- 修改结构体时必须传入指针；
- 标签可自定义语义（如 `"upper":"y"` 表示转大写）；
- 使用 `.Tag.Get()` 可直接取键值。

------

### 3️⃣ 调用结构体方法

```go
package main

import (
	"fmt"
	"reflect"
)

type Student struct {
	Name string
}

func (Student) SayHi(msg string) {
	fmt.Println("SayHi:", msg)
}

func (s *Student) SayBye(msg string) {
	fmt.Println("SayBye:", msg)
}

func CallByName(obj any, method string, args ...any) error {
	rv := reflect.ValueOf(obj)

	// 自动处理指针和值接收者
	if rv.Kind() != reflect.Ptr && rv.CanAddr() {
		rv = rv.Addr()
	}

	mv := rv.MethodByName(method)
	if !mv.IsValid() {
		return fmt.Errorf("method %q not found on %T", method, obj)
	}

	in := make([]reflect.Value, len(args))
	for i, a := range args {
		in[i] = reflect.ValueOf(a)
	}
	mv.Call(in)
	return nil
}

func main() {
	s := Student{"Mike"}
	CallByName(s, "SayHi", "hello")
	CallByName(&s, "SayBye", "goodbye")
}
```

✅ **说明：**

- `t.Method(i)` 查看方法信息；
- `v.Method(i)` 或 `v.MethodByName()` 获取可调用的函数；
- 小写方法是 **私有**，只能在包内通过反射调用；
- `Call` 的参数必须是 `reflect.Value` 切片。

------

## 六、反射实现 ORM 示例：自动生成 SQL

```go
package main

import (
	"errors"
	"fmt"
	"reflect"
	"strings"
)

type ClassModel struct {
	Name string `orm:"name"`
	Id   int    `orm:"id"`
}

func Find(obj any, query ...any) (string, error) {
	t := reflect.TypeOf(obj)
	if t.Kind() != reflect.Struct {
		return "", errors.New("obj must be a struct")
	}

	// 构造 where 子句
	where := ""
	if len(query) > 0 {
		cond, ok := query[0].(string)
		if !ok {
			return "", errors.New("query[0] must be a string")
		}
		if strings.Count(cond, "?")+1 != len(query) {
			return "", errors.New("invalid number of query parameters")
		}
		for _, arg := range query[1:] {
			switch v := arg.(type) {
			case string:
				cond = strings.Replace(cond, "?", fmt.Sprintf("'%s'", v), 1)
			case int:
				cond = strings.Replace(cond, "?", fmt.Sprintf("%d", v), 1)
			default:
				return "", fmt.Errorf("unsupported query type: %T", arg)
			}
		}
		where = "WHERE " + cond
	}

	// 拼接列名
	var cols []string
	for i := 0; i < t.NumField(); i++ {
		col := t.Field(i).Tag.Get("orm")
		if col != "" {
			cols = append(cols, col)
		}
	}
	table := strings.ToLower(t.Name()) + "s"
	sql := fmt.Sprintf("SELECT %s FROM %s %s;", strings.Join(cols, ","), table, where)
	return sql, nil
}

func main() {
	sql, _ := Find(ClassModel{})
	fmt.Println(sql)

	sql, _ = Find(ClassModel{}, "name = ?", "三年一班")
	fmt.Println(sql)

	sql, _ = Find(ClassModel{}, "name = ? and id = ?", "三年一班", 1)
	fmt.Println(sql)
}
```

✅ **说明：**

- `query ...any` 是变长参数，本质是 `[]any`；
- 标签 `orm:"name"` 表示数据库列名；
- 用反射提取字段名与标签生成 SQL；
- 支持简单的 `WHERE` 条件构建。

------

## ✅ 小结

| 功能           | 方法                               | 注意事项         |
| -------------- | ---------------------------------- | ---------------- |
| 获取类型       | `reflect.TypeOf()`                 | 只能查看类型信息 |
| 获取值         | `reflect.ValueOf()`                | 支持读取值       |
| 修改值         | `Value.SetXxx()`                   | 必须传入指针     |
| 获取结构体字段 | `Type.Field(i)` / `Value.Field(i)` | 可访问 Tag 与值  |
| 调用方法       | `Value.MethodByName(name).Call()`  | 参数类型要匹配   |
| 获取标签       | `StructField.Tag.Get("key")`       | 支持自定义标签   |

| 方法名               | 作用             | 返回值        | 典型用途                          |
| -------------------- | ---------------- | ------------- | --------------------------------- |
| `CanSet()`           | 判断是否能修改值 | bool          | 判断 `reflect.Value` 是否可被 Set |
| `SetInt(int64)`      | 设置整型值       | 无            | 修改被反射的整型变量的值          |
| `MethodByName(name)` | 按名字查找方法   | reflect.Value | 动态调用方法                      |

```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct{}

func (Person) Hello() {
	fmt.Println("Hello")
}

func InvokeIfExist(obj any, methodName string) {
	v := reflect.ValueOf(obj)
	m := v.MethodByName(methodName) // ✅ 直接按名字查方法
	if !m.IsValid() {
		fmt.Println("method not found")
		return
	}
	m.Call(nil)
}

func main() {
	InvokeIfExist(Person{}, "Hello") // 输出: Hello
	InvokeIfExist(Person{}, "Bye")   // 输出: method not found
}
```


# reflect-cheat-sheet

### Table Of Content
- [Read struct tags](#read-struct-tags)
- [Get and set struct fields](#get-and-set-struct-fields)
- [Function calls](#function-calls)
  - [Call to a method without prameters, and without return value](#call-method-without-prameters-and-without-return-value)
  - [Call to a function with list of arguments, and validate return values](#call-function-with-list-of-arguments-and-validate-return-values)
  - [Call to a function dynamically. similar to the template/text package](#call-to-a-function-dynamically-similar-to-the-templatetext-package)
  - [Call to a function with variadic parameter](#call-function-with-variadic-parameter)
  - [Create function at runtime](#create-function-at-runtime)
- [Fill slice with values](#fill-slice-with-strings-without-knowing-its-type-use-case-decoder)
- [Set a value of a number](#set-a-value-of-a-number-use-case-decoder)
- [Decode key-value pairs into map](#decode-key-value-pairs-into-map)
- [Decode key-value pairs into struct](#decode-key-value-pairs-into-struct)

### Read struct tags

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email  string `mcl:"email"`
	Name   string `mcl:"name"`
	Age    int    `mcl:"age"`
	Github string `mcl:"github" default:"a8m"`
}

func main() {
	var u interface{} = User{}
	// TypeOf returns the reflection Type that represents the dynamic type of u.
	t := reflect.TypeOf(u)
	// Kind returns the specific kind of this type. 
	if t.Kind() != reflect.Struct {
		return
	}
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Println(f.Tag.Get("mcl"), f.Tag.Get("default"))
	}
}
```
Calling to `Kind()` can returns one of [this list](https://golang.org/pkg/reflect/#Kind).

### Get and set struct fields
```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email  string `mcl:"email"`
	Name   string `mcl:"name"`
	Age    int    `mcl:"age"`
	Github string `mcl:"github" default:"a8m"`
}

func main() {
	u := &User{Name: "Ariel Mashraki"}
	// Elem returns the value that the pointer u points to.
	v := reflect.ValueOf(u).Elem()
	f := v.FieldByName("Github")
	// make sure that this field is defined, and can be changed.
	if !f.IsValid() || !f.CanSet() {
		return
	}
	if f.Kind() != reflect.String || f.String() != "" {
		return
	}
	f.SetString("a8m")
	fmt.Printf("Github username was changed to: %q\n", u.Github)
}
```

### Function calls

#### Call method without prameters, and without return value
```go
package main

import (
	"fmt"
	"reflect"
)

type A struct{}

func (A) Hello() { fmt.Println("World") }

func main() {
	// ValueOf returns a new Value, which is the reflection interface to a Go value.
	v := reflect.ValueOf(A{})
	m := v.MethodByName("Hello")
	if m.Kind() != reflect.Func {
		return
	}
	m.Call(nil)
}
```

#### Call function with list of arguments, and validate return values
```go
package main

import (
	"fmt"
	"reflect"
)

func Add(a, b int) int { return a + b }

func main() {
	v := reflect.ValueOf(Add)
	if v.Kind() != reflect.Func {
		return
	}
	t := v.Type()
	argv := make([]reflect.Value, t.NumIn())
	for i := range argv {
		// validate the type of parameter "i".
		if t.In(i).Kind() != reflect.Int {
			return
		}
		argv[i] = reflect.ValueOf(i)
	}
	// note that, len(result) == t.NumOut()
	result := v.Call(argv)
	if len(result) != 1 || result[0].Kind() != reflect.Int {
		return
	}
	fmt.Println(result[0].Int())
}
```

#### Call to a function dynamically. similar to the template/text package
```go
package main

import (
	"fmt"
	"html/template"
	"reflect"
	"strconv"
	"strings"
)

func main() {
	funcs := template.FuncMap{
		"trim":    strings.Trim,
		"lower":   strings.ToLower,
		"repeat":  strings.Repeat,
		"replace": strings.Replace,
	}
	fmt.Println(eval(`{{ "hello" 4 | repeat }}`, funcs))
	fmt.Println(eval(`{{ "Hello-World" | lower }}`, funcs))
	fmt.Println(eval(`{{ "foobarfoo" "foo" "bar" -1 | replace }}`, funcs))
	fmt.Println(eval(`{{ "-^-Hello-^-" "-^" | trim }}`, funcs))
}

// evaluate an expression. note that this implemetation is assuming that the
// input is valid, and also very limited. for example, whitespaces are not allowed
// inside a quoted string.
func eval(s string, funcs template.FuncMap) (string, error) {
	args, name := parseArgs(s)
	fn, ok := funcs[name]
	if !ok {
		return "", fmt.Errorf("function %s is not defined", name)
	}
	v := reflect.ValueOf(fn)
	t := v.Type()
	if len(args) != t.NumIn() {
		return "", fmt.Errorf("invalid number of arguments. got: %v, want: %v", len(args), t.NumIn())
	}
	argv := make([]reflect.Value, len(args))
	// go over the arguments, validate and build them.
	// note that we support only int, and string in this simple example.
	for i := range argv {
		var argType reflect.Kind
		// if the argument "i" is string.
		if strings.HasPrefix(args[i], "\"") {
			argType = reflect.String
			argv[i] = reflect.ValueOf(strings.Trim(args[i], "\""))
		} else {
			argType = reflect.Int
			// assume that the input is valid.
			num, _ := strconv.Atoi(args[i])
			argv[i] = reflect.ValueOf(num)
		}
		if t.In(i).Kind() != argType {
			return "", fmt.Errorf("Invalid argument. got: %v, want: %v", argType, t.In(i).Kind())
		}
	}
	result := v.Call(argv)
	// in real-world code, we validate it before executing the function,
	// using the v.NumOut() method. similiar to the text/template package.
	if len(result) != 1 || result[0].Kind() != reflect.String {
		return "", fmt.Errorf("function %s must return a one string value", name)
	}
	return result[0].String(), nil
}

// parseArgs is an auxiliary function, that extract the function and its
// parameter from the given expression.
func parseArgs(s string) ([]string, string) {
	args := strings.Split(strings.Trim(s, "{ }"), "|")
	return strings.Split(strings.Trim(args[0], " "), " "), strings.Trim(args[len(args)-1], " ")
}
```

#### Call function with variadic parameter
```go
package main

import (
	"fmt"
	"math/rand"
	"reflect"
)

func Sum(x1, x2 int, xs ...int) int {
	sum := x1 + x2
	for _, xi := range xs {
		sum += xi
	}
	return sum
}

func main() {
	v := reflect.ValueOf(Sum)
	if v.Kind() != reflect.Func {
		return
	}
	t := v.Type()
	argc := t.NumIn()
	if t.IsVariadic() {
		argc += rand.Intn(10)
	}
	argv := make([]reflect.Value, argc)
	for i := range argv {
		argv[i] = reflect.ValueOf(i)
	}
	result := v.Call(argv)
	fmt.Println(result[0].Int()) // assume that t.NumOut() > 0 tested above.
}
```

### Create function at runtime
```go
package main

import (
	"fmt"
	"reflect"
)

type Add func(int64, int64) int64

func main() {
	t := reflect.TypeOf(Add(nil))
	mul := reflect.MakeFunc(t, func(args []reflect.Value) []reflect.Value {
		a := args[0].Int()
		b := args[1].Int()
		return []reflect.Value{reflect.ValueOf(a+b)}
	})
	fn, ok := mul.Interface().(Add)
	if !ok {
		return
	}
	fmt.Println(fn(2,3))
}
```

### Fill slice with strings, without knowing its type. Use case: decoder.
```go
package main

import (
	"fmt"
	"io"
	"reflect"
)

func main() {
	var (
		a []string
		b []interface{}
		c []io.Writer
	)
	fmt.Println(fill(&a), a) // pass
	fmt.Println(fill(&b), b) // pass
	fmt.Println(fill(&c), c) // fail
}

func fill(i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	if v.Kind() != reflect.Slice {
		return fmt.Errorf("can't fill non-slice value")
	}
	v.Set(reflect.MakeSlice(v.Type(), 3, 3))
	// validate the type of the slice. see below.
	if !canAssign(v.Index(0)) {
		return fmt.Errorf("can't assign string to slice elements")
	}
	for i, w := range []string{"foo", "bar", "baz"} {
		v.Index(i).Set(reflect.ValueOf(w))
	}
	return nil
}

// we accept string, or empty interface.
func canAssign(v reflect.Value) bool {
	return v.Kind() == reflect.String || (v.Kind() == reflect.Interface && v.NumMethod() == 0)
}
```

### Set a value of a number. Use case: decoder. 
```go
package main

import (
	"fmt"
	"reflect"
)

const n = 255

func main() {
	var (
		a int8
		b int16
		c uint
		d float32
		e string
	)
	fmt.Println(fill(&a), a)
	fmt.Println(fill(&b), b)
	fmt.Println(fill(&c), c)
	fmt.Println(fill(&d), c)
	fmt.Println(fill(&e), e)
}

func fill(i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	switch v.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		if v.OverflowInt(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetInt(n)
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		if v.OverflowUint(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetUint(n)
	case reflect.Float32, reflect.Float64:
		if v.OverflowFloat(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetFloat(n)
	default:
		return fmt.Errorf("can't assign value to a non-number type")
	}
	return nil
}
```

### Decode key-value pairs into map

```go
package main

import (
	"fmt"
	"reflect"
	"strconv"
	"strings"
)

func main() {
	var (
		m0 = make(map[int]string)
		m1 = make(map[int64]string)
		m2 map[interface{}]string
		m3 map[interface{}]interface{}
		m4 map[bool]string
	)
	s := "1=foo,2=bar,3=baz"
	fmt.Println(decodeMap(s, &m0), m0) // pass
	fmt.Println(decodeMap(s, &m1), m1) // pass
	fmt.Println(decodeMap(s, &m2), m2) // pass
	fmt.Println(decodeMap(s, &m3), m3) // pass
	fmt.Println(decodeMap(s, &m4), m4) // fail
}

func decodeMap(s string, i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	t := v.Type()
	// allocate a new map, if v is nil. see: m2, m3, m4.
	if v.IsNil() {
		v.Set(reflect.MakeMap(t))
	}
	// assume that the input is valid.
	for _, kv := range strings.Split(s, ",") {
		s := strings.Split(kv, "=")
		n, err := strconv.Atoi(s[0])
		if err != nil {
			return fmt.Errorf("failed to parse number: %v", err)
		}
		k, e := reflect.ValueOf(n), reflect.ValueOf(s[1])
		// get the type of the key.
		kt := t.Key()
		if !k.Type().ConvertibleTo(kt) {
			return fmt.Errorf("can't convert key to type %v", kt.Kind())
		}
		k = k.Convert(kt)
		// get the element type.
		et := t.Elem()
		if et.Kind() != v.Kind() && !e.Type().ConvertibleTo(et) {
			return fmt.Errorf("can't assign value to type %v", kt.Kind())
		}
		v.SetMapIndex(k, e.Convert(et))
	}
	return nil
}
```

### Decode key-value pairs into struct
```go
package main

import (
	"fmt"
	"reflect"
	"strings"
)

type User struct {
	Name    string
	Github  string
	private string
}

func main() {
	var (
		v0 User
		v1 *User
		v2 = new(User)
		v3 struct{ Name string }
		s  = "Name=Ariel,Github=a8m"
	)
	fmt.Println(decode(s, &v0), v0) // pass
	fmt.Println(decode(s, v1), v1)  // fail
	fmt.Println(decode(s, v2), v2)  // pass
	fmt.Println(decode(s, v3), v3)  // fail
	fmt.Println(decode(s, &v3), v3) // pass
}

func decode(s string, i interface{}) error {
	v := reflect.ValueOf(i)

	if v.Kind() != reflect.Ptr || v.IsNil() {
		return fmt.Errorf("decode requires non-nil pointer")
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	// assume that the input is valid.
	for _, kv := range strings.Split(s, ",") {
		s := strings.Split(kv, "=")
		f := v.FieldByName(s[0])
		// make sure that this field is defined, and can be changed.
		if !f.IsValid() || !f.CanSet() {
			continue
		}
		// assume all the fields are type string.
		f.SetString(s[1])
	}
	return nil
}
```

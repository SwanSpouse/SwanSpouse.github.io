---
title: golang 反射中的Type Value 和 Kind
tags:
  - golang
date: 2018-11-18 14:25:49
---



### reflect.Type & reflect.Value

#### reflect.Type

TypeOf returns the reflection Type that represents the dynamic type of i.  If i is a nil interface value, TypeOf returns nil.

```golang
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```

```golang

t := reflect.TypeOf(3)   // a reflect.Type
fmt.Println(t.String())  // "int"
fmt.Println(t)           // "int"

var w io.Writer = os.Stdout
fmt.Println(reflect.TypeOf(w)) // "*os.File"

```

reflect.TypeOf 返回的是i 具体的类型。 所以w的类型是*os.File而不是io.Writer。

为方便进行debug，可以用fmt.Printf("%T\n", 3) 来代替 fmt.Println(reflect.TypeOf(3))。

#### reflect.Value

As with reflect.TypeOf, the results of reflect.ValueOf are always concrete. 和reflect.TypeOf一样，reflect.ValueOf返回也是一个具体的值。

``` golang
v := reflect.ValueOf(3)   // a reflect.Value
fmt.Println(v)  // "3"
fmt.Printf("%v\n",v)  // "3"
fmt.Println(v.String())  // NOTE: "<int Value>"

```

### Kind

Although there are infinitely many types, there are only a finite number of kinds of type: the basic types Bool, String and all the numbers; the aggregate types Array and Struct, the reference types Chan, Func, Ptr, Slice and Map; interface types; and finally Invalid, meaning no value at all (The zero value of a reflect.Value has kind Invalid).

尽管具体的数据类型可以有无限种，但是它们可以被分为几种类型。这个就是reflect.Kind.

```golang
// Any formats any value as a string.
func Any(value interface{}) string {
	return formatAtom(reflect.ValueOf(value))
}

// formatAtom formats a value without inspecting its internal structure.
func formatAtom(v reflect.Value) string {
	switch v.Kind() {
	case reflect.Invalid:
		return "invalid"
	case reflect.Int, reflect.Int8, reflect.Int16,
		reflect.Int32, reflect.Int64:
		return strconv.FormatInt(v.Int(), 10)
	case reflect.Uint, reflect.Uint8, reflect.Uint16,
		reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return strconv.FormatUint(v.Uint(), 10)
		// ...floating-point and complex cases omitted for brevity...
	case reflect.Bool:
		return strconv.FormatBool(v.Bool())
	case reflect.String:
		return strconv.Quote(v.String())
	case reflect.Chan, reflect.Func, reflect.Ptr, reflect.Slice, reflect.Map:
		return v.Type().String() + " 0x" +
			strconv.FormatUint(uint64(v.Pointer()), 16)
	default: // reflect.Array, reflect.Struct, reflect.Interface
		return v.Type().String() + " value"
	}
}

func main() {
	var x int64 = 1
	var d time.Duration = 1 * time.Nanosecond
	fmt.Println(Any(x))                  // "1"
	fmt.Println(Any(d))                  // "1"
	fmt.Println(Any([]int64{x}))         // "[]int64 0x8202b87b0"
	fmt.Println(Any([]time.Duration{d})) // "[]time.Duration 0x8202b87e0"
}
```


### reference
* 《The Go Programming Lauguage》
* https://docs.hacknode.org/gopl-zh/ch12/ch12-02.html

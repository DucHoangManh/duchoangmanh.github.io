---
title: "Go Tips and Optimization Notes"
date: 2022-06-14T13:31:06+07:00
tags: ["go", "efective-go"]
---
Some Go tips (maybe some dark side too) and notes for writing better code. 
 
### 1. Efficiently converse between string and byte slice
Gain some performance with the price of maintainability. 
Using the unsafe package is not advised but we can use it to efficiently converse between a string and byte slice. 
Use at your own risk. 
#### byte slice to string
```go
func String(b []byte) (s string) {
   if len(b) == 0 {
       return ""
   }
   return *(*string)(unsafe.Pointer(&b))
}
```
#### string to byte slice
```go
// mutate the returned slice can cause undefined behavior
func ByteSlice(s string) []byte {
   var b []byte
   hdr := (*reflect.SliceHeader)(unsafe.Pointer(&b))
   hdr.Data = (*reflect.StringHeader)(unsafe.Pointer(&s)).Data
   hdr.Cap = len(s)
   hdr.Len = len(s)
   return b
}
```
 
### 2. Change function return value with defer
Normally, if we change the returned value with defer, it will not works
```go
func someFunc() int {
   i := 1
   defer func() {
       i = 2
   }()
   return i
}
 
func main() {
   fmt.Println(someFunc()) // 1
}
```
But with a named return value, defer can change return value of a function
```go
func someFunc() (i int) {
   i = 1
   defer func() {
       i = 2
   }()
   return i
}
 
func main() {
   fmt.Println(someFunc()) // 2
}
```
 
### 3. Mutate a slice the right way
Slice has a pointer to a backing array no not mean that it can be mutated by value
Let's look at the slice header from reflect package (a runtime representation of a slice but the idea is the same):
```go
type SliceHeader struct {
   Data uintptr // pointer to the underlying array but can change when slice grows
   Len  int // not safe for mutating via value
   Cap  int // not safe for mutating via value
}
```
Don't
```go
func main() {
   var s []int
   mutate(s)
   fmt.Println(s) // []
}
 
func mutate(s []int) {
   s = append(s, 1)
}
```
Do
```go
func main() {
   var s []int
   mutate(&s)
   fmt.Println(s) // [1]
}
 
func mutate(s *[]int) {
   *s = append(*s, 1)
}
```
 
### 4. Check if the `any` have desired method
Sometimes we want to check if a variable of any type has some methods or not, consider using an anonymous interface
```go
import "fmt"
 
type Me int
 
func (m Me) Pr() {
   fmt.Println(m)
}
 
func main() {
   var m any
   m = Me(1)
   if callable, ok := m.(interface {
       Pr()
   }); ok {
       // some action when the method exist
   }
}
```
If the interface is used more than once, better define a proper interface.
 
### 5. Allocate memory for slices in advance
We should use the third parameter when making a slice to allocate some memory for that slice to reduce slice grow.
```go
someSlice := make([]int, 0, size)
```
And keep in mind that even if we use a slice with a part of an underlying array, the rest of that array may live in memory much longer than needed, consider copying to a new slice if necessary. 
Pre-allocating can be used for maps too!
```go
someMap:= make(map[int]int, size)
```
 
### 6. Efficient String concatenation
There's some way to do string concatenation like use `+` or fmt package
```go
result := "hello" + "world" // clean but remember that strings in go are immutable, so not very efficient
```
```go
result := fmt.Sprintf("%s %s", "hello", "world") // use reflect under the hood
```
Instead, consider `strings.Builder`
```go
var b strings.Builder
b.WriteString("hello")
b.WriteString("world")
result := b.String()
```
In most cases, the difference can be neglected, but if we are doing heavy string concatenation like in a for loop, better use `strings.Builder`
 
### 7. Channel special behaviors
They are well known:
- A send to a nil channel blocks forever
- A receive from a nil channel blocks forever
- A send to a closed channel panics
- A receive from a closed channel returns the zero value immediately
 
### 8. Do not copy a sync type
sync types shouldn’t be copied. This rule applies to:
- sync.Cond
- sync.Map
- sync.Mutex
- sync.RWMutex
- sync.Once
- sync.Pool
- sync.WaitGroup
 
### 9. io.Reader can only be read once
Most type that implements the `io.Reader` interface (`os.File, http.Request.Body...`) behave like a stream and can only be read once.
We can work around this with `io.TeeReader` that duplicate the stream, but in most case just avoid read an `io.Reader` a second time.
 
### 10. Always close transient resource
Remember to close transient resources to avoid memory leaks. 
Includes `http.Response.Body`, `sql.Rows`, `os.File` and more
 


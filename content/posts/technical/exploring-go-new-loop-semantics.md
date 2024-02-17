---
title: "Go: Exploring new loop semantics"
date: 2024-02-15T20:40:05+07:00
tags: ["go", "efective-go"]
---

Go 1.22 has just been released with a bunch of new features and improvements. In this article, we will explore the new loop semantics and how they can be used to write more expressive and readable code.

## 1. Loop variable is no longer be shared between iterations
Previously, the loop variable was shared between iterations, from go 1.21 (experimental) and now with go 1.22, the loop variable is created anew with each iteration, effectively eliminating one of the most common foot gun in Go (for both experienced and new gophers).
This is no longer needed:
```go
for i := 0; i < 10; i++ {
    i := i
    go func () {
        fmt.Println(i)
    }()
}
```
or this
```go
for i := 0; i < 10; i++ {
    go func (i int) {
        fmt.Println(i)
    }(i)
}
```
now you can do the more intuitive way:
```go
for i := 0; i < 10; i++ {
    go func () {
        fmt.Println(i)
    }()
}
```


## 2. Range over an integer

This feature is straightforward, instead of
```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```
now you can do
```go
for i := range 10 {
    fmt.Println(i)
}
```
and achieve the same result, pretty neat, right?

## 3. Range over a function
In my humble opinion, this is one of the most exciting updates to the Go language in a long time. Now Go have a standard way to handle iterator, which is a common pattern in other languages.  
With preceding go versions, there wasn't a standard way to iterate through a data structure, generic is not yet available in the language so, you couldn't write a simple iterator for different data structures.

- `bufio.Scanner` is an iterator through an `io.Reader`, where the `Scan` method advances to the next value. The value is returned by a `Bytes` method. Errors are collected and returned by an `Err` method.
- `database/sql.Rows` iterates through the results of a database query, where the `Next` method advances to the next row and the value is returned by a `Scan` method which can return an error.

### Let's have a look at the new loop semantics in action:
**_please note that despite this feature is available in go 1.22, it's still experimental and may change in the future, plus you have to build your program using `GOEXPERIMENT=rangefunc`_**
```go
// iterate from 0 to 9
In10 := func(yield func(int) bool) {
    for i := range 10 {
        if !yield(i) {
            return
        }
    }
}
for v := range In10 {
    fmt.Println(v) // 0, 1, 2, 3, 4, 5, 6, 7, 8, 9
}
```
- `In10` is a function that takes a function `yield` as an argument.
- The `yield` function takes an integer and returns a boolean, whenever `In10` is used, the loop body will specify the `yield` function, and the `In10` function will call the `yield` function with the current value of the loop variable.
You can easily see that the yield function have a signature that return a bool but the loop body itself does not return anything, this is because inside the loop body, `continue` or nothing will be translated to `return true` and `break` will be translated to `return false`. Give the user the ability to control the iteration from the loop body.

The compiler will change the for over function to something that looks like this:
```go
Int10(func(i int) bool {
    fmt.Println(i)
    return true
})
```
**_this explanation is somewhat oversimplified, the actual implementation is more complex, you can find more about it [here](https://go.googlesource.com/go/+/refs/changes/41/510541/7/src/cmd/compile/internal/rangefunc/rewrite.go)_**
Besides the one parameter yield function mentioned above, functions that can be ranged over can have zero or two parameters, as long as they have the following signature:
```go
package iter

type Seq0 func(yield func() bool) bool
type Seq[V any] func(yield func(V) bool) bool
type Seq2[K, V any] func(yield func(K, V) bool) bool
```

**More examples:**

```go
// iterate over a specific range with start and end value
InRange := func(start, end int) func(yield func(int) bool) {
    return func(yield func(int) bool) {
        for i := start; i < end; i++ {
            if !yield(i) {
                return
            }
        }
    }
}

for x := range InRange(5, 10) {
    fmt.Println(x) // 5, 6, 7, 8, 9
}
```

```go
// iterate over words in a string separated by space
Words := func(s string) func(yield func(int, string) bool) {
    words := strings.Split(s, " ")
    return func(yield func(int, string) bool) {
        for i, word := range words {
            if !yield(i, word) {
                return
            }
        }
    }
}

for i, word := range Words("sun rises in the east") {
    fmt.Println(i, word) // 0 sun 1 rises 2 in 3 the 4 east
}
```

usually, while using some polling based library, you have to write a loop like this:
```go
for {
    m, err := reader.ReadMessage(context.Background()) 
    if err != nil {
        // handle error
        continue
    }
    // handle message
}		
```
you can leverage the new loop semantics to write a more expressive message poller:
```go
ReaderIterator := func (reader Reader) func (func (Message, error) bool) {
    return func (yield func (Message, error) bool) {
        for {
            m, err := reader.ReadMessage(context.Background())
            if !yield(m, err) {
                break
            }
        }
    }
}

for message err := range ReaderIterator(reader) {
    if err != nil {
        // handle error
    }
    // handle message
}
```
### Pull iterator
All the examples above are push iterators, pushing values to the yield function. But that is not always the case in the real world, sometimes you want to pull values from the iterator.  
The `Pull` function from `iter` package converse a `Seq`- standard push iterator to a pull iterator. Calling `Pull` will start an iteration and returns a pair
of functions `next` and `stop`, which return the next value from the iterator and stop it, respectively.
```go
InRange := func(start, end int) func(yield func(int) bool) {
    return func(yield func(int) bool) {
        for i := start; i < end; i++ {
            if !yield(i) {
                return
            }
        }
    }
}
next, stop := iter.Pull(InRange(5, 7))
defer stop()
for value, more := next(); more; value, more = next() {
    fmt.Println(value) // 5, 6
}
```

**The new loop semantics surely is a great addition to the Go language as it opens the door for more idiomatic APIs with range functions.**

## References:
- [Go 1.22 Release Notes](https://tip.golang.org/doc/go1.22)
- [spec: add range over int, range over func](https://github.com/golang/go/issues/61405)
- [iter: new package for iterators](https://github.com/golang/go/issues/61897)
- [Go Wiki: Rangefunc Experiment](https://go.dev/wiki/RangefuncExperiment#what-will-idiomatic-apis-with-range-functions-look-like)
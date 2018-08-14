+++
draft = false
tags = ["Programming"]
topics = ["Golang"]
description = "A summary of the nil in Golang based on some articles."
title = "Understanding the nil in Golang"
date = "2018-08-10T17:00:00+08:00"
+++

# Understanding the nil in Golang

## Reference

[GopherCon 2016: Francesc Campoy - Understanding nil](https://www.youtube.com/watch?v=ynoY2xz-F8s)

[理解Go语言的nil](https://www.jianshu.com/p/dd80f6be7969)

## Is it a bug?

{{< highlight go >}}
package main

import (
    "fmt"
)

type doError struct{}

func (e *doError) Error() string {
    return ""
}

func do() error {
    var err *doError
    return err
}

func main() {
    err := do()
    fmt.Println(err == nil) // What's the result?
}
{{< /highlight >}}

## What's the nil?

See a very, very, very familiar code.

{{< highlight go >}}
if err != nil {
    // do something....
}
{{< /highlight >}}

Here is a summary about zero value of all types.

{{< highlight go >}}
bool -> false                              
numbers -> 0                                 
string -> ""      

pointers -> nil
slices -> nil
maps -> nil
channels -> nil
functions -> nil
interfaces -> nil
{{< /highlight >}}

See an example.

{{< highlight go >}}
type Person struct {
    AgeYears int
    Name     string
    Friends  []Person
}

var p Person // Person{0, "", nil}
{{< /highlight >}}

If a variable is only declared and has no assignment, it is zero value.

`nil` is a pre-defined variable, not a keyword.

{{< highlight go >}}
type Type int
var nil Type
{{< /highlight >}}

You can even change its value.

{{< highlight go >}}
var nil = errors.New("hi")
{{< /highlight >}}

But you should not do it.

## What's the usage of nil?

### For Pointers

{{< highlight go >}}
var p *int
p == nil    // true
*p          // panic: invalid memory address or nil pointer dereference
{{< /highlight >}}

Dereference to `nil` pointer will cause panic.

What's the usage of `nil`? Firstly see a code.

{{< highlight go >}}
type tree struct {
    v int
    l *tree
    r *tree
}

// first solution
func (t *tree) Sum() int {
    sum := t.v
    if t.l != nil {
        sum += t.l.Sum()
    }
    if t.r != nil {
        sum += t.r.Sum()
    }
    return sum
}
{{< /highlight >}}

Two problems exist in this code. First is redundant code like:

{{< highlight go >}}
if v != nil {
    v.m()
}
{{< /highlight >}}

Second is it will cause panic when `t` is `nil`.

{{< highlight go >}}
var t *tree
sum := t.Sum()   // panic: invalid memory address or nil pointer dereference
{{< /highlight >}}

How to solve? Let's see an example of receiver.

{{< highlight go >}}
type person struct {}
func sayHi(p *person) { fmt.Println("hi") }
func (p *person) sayHi() { fmt.Println("hi") }
var p *person
p.sayHi() // hi
{{< /highlight >}}

**For pointer, even it is `nil`, the method of its object can be invoked.**

So we can optimize the code like:

{{< highlight go >}}
func(t *tree) Sum() int {
    if t == nil {
        return 0
    }
    return t.v + t.l.Sum() + t.r.Sum()
}
{{< /highlight >}}

See more examples:

{{< highlight go >}}
func(t *tree) String() string {
    if t == nil {
        return ""
    }
    return fmt.Sprint(t.l, t.v, t.r)
}

// nil receiver are useful: Find
func (t *tree) Find(v int) bool {
    if t == nil {
        return false
    }
    return t.v == v || t.l.Find(v) || t.r.Find(v)
}
{{< /highlight >}}

So if no special reason, avoid to use initialize function like `NewX()`. Just use the default value.

## For Slices

{{< highlight go >}}
// nil slices
var s []slice
len(s)  // 0
cap(s)  // 0
for range s  // iterates zero times
s[i]  // panic: index out of range
{{< /highlight >}}

A `nil` slice can only not be indexed and all other operations can be done. Use `append` and it will be extended automatically. See slice data structure:

{{< fluid_imgs
  "slice-structure|/img/post/slice_structure.png|"
>}}

When there is an element in it, it will be like:

{{< fluid_imgs
  "slice-with-element|/img/post/slice_with_element.png|"
>}}

So no need to care about size of slice.

## For Maps

Map, function and channel are special pointers in Golang and have their own implementation.

{{< highlight go >}}
// nil maps
var m map[t]u
len(m)  // 0
for range m // iterates zero times
v, ok := m[i] // zero(u), false
m[i] = x // panic: assignment to entry in nil map
{{< /highlight >}}

`nil` map can be a readonly map.

What's the usage of nil? See an example:

{{< highlight go >}}
func NewGet(url string, headers map[string]string) (*http.Request, error) {
    req, err := http.NewRequest(http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }

    for k, v := range headers {
        req.Header.Set(k, v)
    }
    return req, nil
}
{{< /highlight >}}

When need to set header, we can:

{{< highlight go >}}
NewGet("http://google.com", map[string]string{
  "USER_AGENT": "golang/gopher",
},)
{{< /highlight >}}

When no need to set header, we can:

{{< highlight go >}}
NewGet("http://google.com", map[string]string{})
{{< /highlight >}}

Or we use `nil`.

{{< highlight go >}}
NewGet("http://google.com", nil)
{{< /highlight >}}

## For Channels

{{< highlight go >}}
// nil channels
var c chan t
<- c      // blocks forever
c <- x    // blocks forever
close(c)  // panic: close of nil channel
{{< /highlight >}}

Close a `nil` channel will cause panic.

What's the usage of `nil` channel? See an example.

{{< highlight go >}}
func merge(out chan<- int, a, b <-chan int) {
	for {
		select {
		case v := <-a:
			out <- v
		case v := <-b:
			out <- v
		}
	}
}
{{< /highlight >}}

If `a` or `b` is closed, `<-a` or `<-b` will return zero value without a stop. This is not expected. We want to stop merge value from `a` if it is closed. Change the code:

{{< highlight go >}}
func merge(out chan<- int, a, b <-chan int) {
  for a != nil || b != nil {
    select {
      case v, ok := <-a:
          if !ok {
            a = nil
            fmt.Println("a is nil")
            continue
          }
          out <- v
      case v, ok := <-b:
          if !ok {
            b = nil
            fmt.Println("b is nil")
            continue
          }
          out <- v
    }
  }
  fmt.Println("close out")
  close(out)
}
{{< /highlight >}}

When `a` or `b` is closed, set it as `nil`, it means the `case` will not be used because `nil` channel will be blocked forever.

## For Interfaces (TBD)

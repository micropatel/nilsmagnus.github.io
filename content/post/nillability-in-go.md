+++
date = "2018-11-06T20:43:58+01:00"
draft = false
title = "Nillability and zero-values in go"
toc = true
tags        = [ "go", "nil", "null","nils", "nullability", "nillability", "basics"]
+++


Beeing a long time java-developer, I am obsessed with null-checking and handling null values.
In golang, the story is somewhat different. In this post I will try to describe how `nil` and `zero-values`
are used in golang.

# non-nillable and nillable types

Types can be either nillable or non-nillable in go. The non-nillable types can never be nil
and will never cause you a `nil-panic` (the java equivalent of nullpointerexception) But when
are dealing with the nillable types, we have to take a bit of caution although not as 
much as in java(or other languages with nillable types). 


## The non-nillables basic types
In go, the basic types are not nillable. A statement like 

    var a int = nil
    
does not compile because an `int` can never be nil. The default value of an unassigned `int` type is 0. 
Running the statement 

    var a int // default value of int, cannot be nil
    fmt.Println(a) // 0
    
will output the default value of `int`; "`0`". We call this the zero-value of the type. 


The same way `int` defaults to 0, these are the other basic types with their zero-values:

| Types | Zero value |
|------|------------|
| int, int8, int16, int32, int64  |    0     |   
| uint, uint8, uint16, uint32, uint64  |    0     |   
|uintptr| 0|
| float32, float64|    0.0     |   
| byte| 0 |
| rune | 0 |
|string | "" (empty string)|
| complex64, complex128| (0,0i)|
| arrays of non-nillable types | array of zero-values |
| arrays of nillable types | array of nil-values |


## Non-nillable structs 

Composed `struct` types are also non-nillable, and the default value of a struct will contain 
the default value for all its fields. 

Consider the code with the struct type Person, 

    type Person struct {
    	Name string
    	Age  int
    }
    var p Person // zero-value of type Person
    
    fmt.Printf("[%#v]\n", p)

will print  `[main.Person{Name:"", Age:0}]` when run in main. You can test this [on this snippet](https://play.golang.org/p/DEVSf5s5OVh) on The Go Playground.

## The nillable types

More advanced types are nillable and _can_ cause panic if they are not initialized. 

The nillable types are _functions, channels, slices, maps, interface-types and pointers_.

However, nil-slices and nil-maps can still be used and does not have to be initialized before we start using them. 

## nil-maps
Maps will always return the zero-value of the value if it is nil, the same behaviour as if the key of the map is non-existent. 
The code 

    var p map[int]string // nil map
    fmt.Printf(" %#v  length %d \n",  p[99], len(p))
    
 simply prints `"" length 0`, the extracted value for key 99 is the zero value of `string`. 
 
Assigning values to a nil-map, will however cause panic:

   	var p map[string]int	// nil map 
    p["nils"] = 19 // panic: assignment to entry in nil map

## nil-slices

Refering to out-of-bounds on slices _will_ cause panic, but operations like `len()` and `cap()` will not panic. They will simply return `0`, since both capacity and length is zero 
for an uninitialized slice. Append can be safely called on the nil-slice. So the code 

    var p []string // nil slice
   	fmt.Printf("uninitialized -> %d, %d\n",  len(p), cap(p))
   	p1 := append(p, "nils") // create a new slice p1 from p
   	fmt.Printf("after append  -> %d, %d %#v\n",  len(p1), cap(p1), p1)

Will print

    uninitialized -> 0, 0
    after append  -> 1, 1 []string{"nils"}
    
Play with this example on [the playground](https://play.golang.org/p/_NQNqQniPcx).


## nillable pointers, functions and interface-types can cause panic

Pointers and interface-types are however nillable. Whenever dealing with these types, we have to consider if 
they are nil or not to avoid panics. These code-snippets for instance, will cause a panic:

    var p *int // pointer to an int
    *p++ // panic: runtime error: invalid memory address or nil pointer dereference
    // p is an address to nothing, and therefore is nil

and    
   
    var p error // nil-value of type error
    error.Error() // panic: runtime error: invalid memory address or nil pointer dereference

and 

    var f func(string) // nil-function
    f("oh oh") // panic: runtime error: invalid memory address or nil pointer dereference
 
## nil channels blocks forever

Trying to read from a nil-channel or write to a nil-channel will block forever. Closing a nil-channel 
will cause panic. 


# Wrapup

`nil` is well defined in go. Knowing what can be nil and how to handle nil values of the different
types increases your understanding of whats happening and can help you write better go-code. 

 

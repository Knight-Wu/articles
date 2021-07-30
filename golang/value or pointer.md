
### https://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go 总结
func (t T)MyMethod(s string) {
    // ...
}

is a function of type  `func(T, string)`; method receivers are passed into the function by value just like any other parameter. by value means copy.

* Any changes to the receiver made inside of a method defined on a value type (e.g., `func (d Dog) Speak() { ... }`) will not be seen by the caller because the caller is scoping a completely separate `Dog` value.

* This works because a pointer type can access the methods of its associated value type, but not vice versa.( a pointer type may call the methods of its associated value type, but not vice versa)

* Since everything is passed by value, it should be obvious why a `*Cat` method is not usable by a `Cat` value; any one `Cat` value may have any number of `*Cat` pointers that point to it. If we try to call a `*Cat` method by using a `Cat` value, we never had a `*Cat` pointer to begin with. Conversely, if we have a method on the `Dog` type, and we have a `*Dog` pointer, we know exactly which `Dog` value to use when calling this method, because the `*Dog` pointer points to exactly one `Dog` value
( 所以 method receiver 最好是 pointer ? )


* Remember: everything is pass-by-value in Go. That means that inside of the `UnmarshalJSON` method, the pointer `t` is not the same pointer as the pointer in its calling context; it is a copy. If you were to assign `t` to another value directly, you would just be reassigning a function-local pointer; the change would not be seen by the caller.
(包括指针也是, 所以在函数内把指针重新指向, 对函数外也不生效)

* as a rule of thumb you can just remember that it’s typically better to take in an `interface{}` value as a parameter than it is to return an `interface{}` value 
(interface 最好作为参数而不是返回值) 

-   an  `interface{}`  value is not of any type; it is of  `interface{}`  type
-   interfaces are two words wide; schematically they look like  `(type, value)`

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMyNjI4MjgzNywtMTM5Mjk3MTE1OSwtNj
kxNTU5ODA2LDE1MjYxNzU1OTksLTExMzcxMDgxNThdfQ==
-->
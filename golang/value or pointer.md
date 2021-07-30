
func (t T)MyMethod(s string) {
    // ...
}

is a function of type  `func(T, string)`; method receivers are passed into the function by value just like any other parameter. by value means copy.

Any changes to the receiver made inside of a method defined on a value type (e.g., `func (d Dog) Speak() { ... }`) will not be seen by the caller because the caller is scoping a completely separate `Dog` value.

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExMzcxMDgxNThdfQ==
-->
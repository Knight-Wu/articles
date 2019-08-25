because [Michael Jones explained this well](https://groups.google.com/d/msg/golang-nuts/wcrZ3P1zeAk/WI88iQgFMvwJ)
摘录如下
> I think the question is really "but how would you inspect the object pointer to find and dispatch the function, as in C++ vtable." We might answer this from that angle:

  

> Method dispatch, as is used in some other languages with something like objectpointer->methodName() involves inspection and indirection via the indicated pointer (or implicitly taking a pointer to the object) to determine what function to invoke using the named method. Some of these languages use pointer dereferencing syntax to access the named function. In any such cases, calling a member function or method on a zero pointer understandably has no valid meaning.
> 
>   
> 
> In Go, however, the function to be called by the Expression.Name() syntax is entirely determined by the type of Expression and not by the particular run-time value of that expression, including nil. In this manner, the invocation of a method on a nil pointer of a specific type has a clear and logical meaning. Those familiar with vtable[] implementations will find this odd at first, yet, when thinking of methods this way, it is even simpler and makes sense. Think of:
> 
>   
> 
> func (p *Sometype) Somemethod (firstArg int) {}
> 
>   
> 
> as having the literal meaning:
> 
>   
> 
> func SometypeSomemethod(p *Sometype, firstArg int) {}
> 
>   
> 
> and in this view, the body of SometypeSomemethod() is certainly free to test it's (actual) first argument (p *Sometype) for a value of nil. Note though that the calling site invoking on a nil value must have a context of the expected type. An effort to invoke an unadorned nil.Somemethod would not work in Go because there is be no implicit "Sometype" for the typeless value nil to expand the Somemethod() call into "SometypeSomemethod()"

大致意思就是如果 object 是 nil , 那是如何找到这个 function 的呢? 在 golang 中与其他语言不同, 通过 [Expression.Name](http://expression.name/)()
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI3NzU2ODUyMiwtMTYwMjM5OTMwNV19
-->
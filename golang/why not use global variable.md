* 背景
 今天领导看我没完全写好的代码, 说不要用全局变量, 然后两个人对原因都不太了解, 于是花了一个小时研究了一下

https://www.reddit.com/r/golang/comments/9i7ipd/passing_through_layers_vs_global_variables/

找到了三点原因: 
1. 依赖的问题, 用全局变量的话, 使用的那个 package 需要多一个 import
2. 封装,代码整洁的问题, 不用全局变量, 全部代码都封装在某一个 package 里面
3. 方便测试, mock 很简单, 相比于 interface 而言,  如果用 interface 接口, 则很方便 mock 

```
// 1
var (
  db *sql.DB
)
type UserRepo struct {}
func (ur UserRepo) GetUser(id string) (*User, error) {
  r, err := db.Query(...)
  ...
}

```

```
// 2
type UserRepo struct {
  db *sql.DB
}
func (ur UserRepo) GetUser(id string) (*User, error) {
  r, err := ur.db.Query(...)
  ...
}

```

The advantage of #2 is a little nuanced(细微差别).

First, there's the possibility that the dependency we're needing here would actually be instantiated in another package; by doing this, assuming the type of that dependency is something like a  `sql.DB`  in the stdlib, this package no longer needs to import that other package. Very nice.

There's also a benefit in encapsulation, though its harder to explicitly outline why this leads to cleaner code. All of the things which UserRepo needs to do its job is now defined inside the UserRepo itself.

```
// 3
type Queryer interface {
  Query(query string, args ...interface{}) (*sql.Rows, error)
}
type UserRepo struct {
  queryer Queryer
}
func (ur UserRepo) GetUser(id string) (*User, error) {
  r, err := ur.queryer.Query(...)
  ...
}

```

The advantage of #3 is that it is significantly easier to test, because you only need to create mock types that fulfill the interface  `GetUser`  requires, versus mocking out the entire  `sql.DB`  struct. We also now get to forget that "qualification" on the first reason why #2 is advantageous; even if the dependency has a concrete type implementation in another package, this package doesn't have to import it, period.

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjg3NzY4MTkyXX0=
-->
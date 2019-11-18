#### 当go 中引用了cgo, cgo 不能在mac 编译linux , 所以需要在本地的linux docker 中编译, 本地编译方便, 

1. 启动docker container
 docker run --rm -it -v ~/.ssh/:/root/.ssh/ -v /usr/local/go/src/:/go/src -v /Users/xxx/go:/go/code  golang:1.12.5 /bin/bash

2. 在container 里面指定GOPATH, 否则找不到依赖. 






> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyODc3MDIyNjddfQ==
-->
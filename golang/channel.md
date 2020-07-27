
* close
 send on a close channel will cause panic, receive on close channel 接收所有已经发送的 val, 直到 val 接收完, 若 val 已经接收完, 则后续的 receive 会立马返回, 并且返回 val 的 zero value

* unbufferer channel
>A send operation on an unbuffered channel blocks the sending goroutine until another goroutine executes a corresponding receive on the same channel, at which point the value is transmitted and both goroutines may continue. Conversely, if the receive operation was attempted first, the receiving goroutine is blocked until another goroutine performs a send on the same channel.

send 和 receive 在一个 unbuffered channel, 会互相阻塞, 例如若没有 goroutine 在 receive , send 会阻塞, 反之亦然
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAzMDg0Mzg2NCwtNDM2NDE0MzZdfQ==
-->
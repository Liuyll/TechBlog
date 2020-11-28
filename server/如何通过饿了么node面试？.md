## 事件循环

### Console是同步还是异步？

事实上，console不是一致的同步，也非一致的异步，甚至它的实现没有被标准规定。它完全由实现者对io的异步推迟处理算法决定。

> Warning: The global console object’s methods are neither consistently synchronous like the browser APIs they resemble, nor are they consistently asynchronous like all other Node.js streams. See the note on process I/O for more information.

而在chrome上，console是同步的，但打印出来的object跟点击展开的object会经历完全不同的两次取值。

[详见cnode](https://cnodejs.org/topic/59142d12ba8670562a40ef4d)

#### console的本质？

console是process.stdin.fd的输出流




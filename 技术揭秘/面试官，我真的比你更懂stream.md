## 从readableStream谈起

#### 先看看实现一个readable

```
function MyReadable(options){
		Readable.call(this,options)
		...doSomething
}
Util.inherits(MyReadable,Readable)

new MyReadable({
		read!:new Function()
})
```

#### read和_read做了什么？

由命名可以看出,_read是底层方法，而read是我们实现的方法。

#### _read

> 当 `readable._read()` 被调用时，如果从资源读取到数据，则需要开始使用 [`this.push(dataChunk)`](http://nodejs.cn/s/8s3paZ) 推送数据到读取队列。 `_read()` 应该持续从资源读取数据并推送数据，直到 `readable.push()` 返回 `false`。 若想再次调用 `_read()` 方法，则需要恢复推送数据到队列。
>
> 一旦 `readable._read()` 方法被调用，将不会再次调用它，直到更多数据通过 [`readable.push()`](http://nodejs.cn/s/8s3paZ) 方法被推送。 空的数据（例如空的 buffer 和字符串）将不会导致 `readable._read()` 被调用。

文档里明确指出，_read调用后将不会再被调用，直至调用push后才会重新被调用。

```javascript
 _read() {
    const data = this.dataSource.makeData();
    setTimeout(() => this.push(data),1000);
 }
// 每隔1s输出一次
```

话不多说，看下push做了什么

```javascript
Readable.prototype.push = function(chunk, encoding) {
  return readableAddChunk(this, chunk, encoding, false);
};
```

readableAddChunk:

```javascript
function readableAddChunk(stream, chunk, encoding, addToFront) {
  debug('readableAddChunk', chunk);
  const state = stream._readableState;

  let skipChunkCheck;

  if (!state.objectMode) {
    if (typeof chunk === 'string') {
      encoding = encoding || state.defaultEncoding;
      if (addToFront && state.encoding && state.encoding !== encoding) {
        // When unshifting, if state.encoding is set, we have to save
        // the string in the BufferList with the state encoding
        chunk = Buffer.from(chunk, encoding).toString(state.encoding);
      } else if (encoding !== state.encoding) {
        chunk = Buffer.from(chunk, encoding);
        encoding = '';
      }
      skipChunkCheck = true;
    }
  } else {
    skipChunkCheck = true;
  }

  if (chunk === null) {
    state.reading = false;
    onEofChunk(stream, state);
  } else {
    var er;
    if (!skipChunkCheck)
      er = chunkInvalid(state, chunk);
    if (er) {
      errorOrDestroy(stream, er);
    } else if (state.objectMode || (chunk && chunk.length > 0)) {
      if (typeof chunk !== 'string' &&
          !state.objectMode &&
          // Do not use Object.getPrototypeOf as it is slower since V8 7.3.
          !(chunk instanceof Buffer)) {
        chunk = Stream._uint8ArrayToBuffer(chunk);
      }

      if (addToFront) {
       ...
    } else if (!addToFront) {
      state.reading = false;
      maybeReadMore(stream, state);
    }
  }
```



#### read(size)

read的行为应该是实现者来定义，比如在fs里，read会触发 `readable`事件

<b>事实上，read是依靠触发底层的_read函数来触发readable事件的</b>

而对`data`事件的监听，则自动会触发read

~~而每次read的size，我们又可以通过设置highWaterMark来指定~~

***注意，上面这句话是有问题的，标准的read实现里，size只应为0来触发底层_read***



### 可控

+ pause
+ resume
+ destory



#### 处理背压问题(backpressure)

对于一个可读流来说，当它被pipe到一个可写流的一端时。`highwatermark`就会类似于滑动窗口一样来控制读写的速度了

一旦可读流的***highwatermark***高于标志位，可写流就无法write了，我们可以通过<b>ok</b>标志位来表示是否背压

```javascript
function write(){
  do {
    ok = writer.write(data)
  } while(ok)
  
  writer.pause()
  writer.once('drain',write)
}
  
write()
```

然而，即使一个可读流的`highwatermark`已经超出，即`write`返回一个`false`时，依然可以继续写入可读流。

此时`nodejs`会把数据缓存到内存，直到内存超出nodejs指定的上限为止。



#### 考虑一下createReadStream(file)是怎么实现的

实际上，可读流我们已经分析清楚了。原理就是通过一个容量为`highwatermark`的`buffer`来进行分批读写。

然而，我么还是没有弄清楚，这个机制如何结合我们的文件系统来进行。

实际上，也是调用`fs.read`来进行读取的，获取句柄后，就可以按照我们上面的逻辑来进行了。

```
 fs.read(
        this.fd,
        this.buffer,
        0,
        howMuchToRead,
        this.pos,
        (err, bytesRead) => {
            // 如果读到内容执行下面代码，读不到则触发 end 事件并关闭文件
            if (bytesRead > 0) {
                // 维护下次读取文件位置
                this.pos += bytesRead;

                // 保留有效的 Buffer
                let realBuf = this.buffer.slice(0, bytesRead);

                // 根据编码处理 data 回调返回的数据
                realBuf = this.encoding
                    ? realBuf.toString(this.encoding)
                    : realBuf;

                // 触发 data 事件并传递数据
                this.emit("data", realBuf);

                // 递归读取
                if (this.flowing) {
                    this.read();
                }
            } else {
                this.isEnd = true;
                this.emit("end"); // 触发 end 事件
                this.detroy(); // 关闭文件
            }
        }

```

当然，我们每次都要更新这个`bytesRead`变量

默认情况下，这个变量等于`HW`。但是如果我们设置了结束位置，或者剩余数据不足`HW`时，就会选择合适的值。

```
let howMuchToRead = this.end
        ? Math.min(this.highWaterMark, this.end - this.pos + 1)
        : this.highWaterMark;
```

这实际上跟我们上面提到的`_read`的逻辑是一致的，也是真正用于读取的部分

当然，我们还需要不断的更新`highwatermark`。



### 深入Readable Stream

#### readable的几种模式

+ <b>flow</b>

![Flowing](https://www.barretlee.com/blogimgs/2017/06/06/node-stream-readable-flowing.png)

这就是readable工作的原理，资源相当于一个水垒，不断向缓存池冲水，水平面为highWaterMark

+ 静止模式

该模式的触发除了显示调用pause外，在监听readable后也会触发

```
readable.on('readable',() => {
		while((data = readable.read(size)) != null){
				doSomething
		}
})
```



[参考](https://www.barretlee.com/blog/2017/06/06/dive-to-nodejs-at-stream-module/)



## transform双工流

***这里不提及duplex流，因为duplex与transform的唯一区别就是前者是读写流不相关，而后者读写流相关，只做中间处理***

#### _transform

我们用一个删除第一行的文件转换函数来说明这个方法

```javascript
function removeFirstLine(){
  	transform.call(this)
  	this._removed = false
}
util.inherits(removeFirstLine,transfrom)

removeFirstLine.prototype._transfrom = function(chunk,encoding,done){
  if(this._removed){
    this.push(chunk)
  }
  else {
    let string = chunk.toString()
    string = string.slice(string.indexOf('\n')+2)
    this.push(string)
    this._remove = true
  }
  done()
}

input
.pipe(new removeFirstLine())
.pipe(output)
```

<b>注意这里的done,它是必须被触发的回调函数，第一个参数是error</b>



### 谈谈pipe

#### ReadStream.pipe(WriteStream,[options])

这是最常见的形式，但是有很多细节依然值得我们关心:

1. option提供了一个`end`参数，它代表着`ReadStream end`后，是否关闭可写流，它默认是`true`，开启它以实现多次pipe同一个`WriteStream`

2. 可读流结束后会触发`ReadStream.on('end')`end事件
3. 可读流发生错误后，可写流将不会自动关闭
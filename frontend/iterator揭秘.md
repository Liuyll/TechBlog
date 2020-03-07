### async是iterator的语法糖吗？

#### 在ES7之前，如何消除`then`的回调地狱

#### co是如何实现的？



####Symbol.iterator

```javascript
let o = {
    a(){
        console.log(this)
        setTimeout(() => {
            console.log('a')
            this.next() // 这里要注意是以o[Symbol.iterator]来调用
        },1000)
    },
    b(){
        setTimeout(() => {
            console.log('b')
            this.next()
        },1000)
    },
    c(){
        setTimeout(() => {
            console.log('c')

            this.next()
        },1000)
    },
}

Object.defineProperty(o,Symbol.iterator,{
    value(){
        let that = this
        let obj = Object.keys(this)
        let idx = 0
        return {
            next(){
                let value
                if(typeof that[obj[idx]] == 'function'){
                    value = that[obj[idx]].call(this)
                }
                idx++
                return {
                    value,
                    done:idx >= obj.length
                }
            }
        }
    }
})

let c = o[Symbol.iterator]()
```

#### Symbol.asyncIterator

`asyncIterator`的接口跟`iterator`没有任何区别，它只是实现了一个`async next`方法，同时它的写法变成了`for await of ...`



##### 如何写一个async iterator呢？

也非常简单，在寻常的iterator前加上`async`就好了

```javascript
function *sync(){}
async function *async(await ...){}
```




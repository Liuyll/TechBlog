## 从写一个co开始

```javascript
function co(gen){
	let g = gen()

	function next(err,preValue){
		let temp
		if(err) temp = g.throw(err)
		else temp = g.next(preValue)
		
		if(!temp.done){
      handleGeneratorValue(temp.value)
    }
	}
  
  function handleGeneratorValue(value){
    if(isPromise(value)){
      value.then((_v) => {
        next(null,_v)
      }).catch(err = > {
        next(err,null)
      })
    }
  }
}
```



### 增强co

我们对`handleGeneratorValue`做一些增强，让它能处理一些情况

```javascript
function handleGeneratorValue(value){
    if(isPromise(value)){
      value.then((_v) => {
        if(_v.type == 'fetch') {
          xxxx
        }
        else if(_v.type == 'delete'){
          xxxx
        }
        next(null,_v)
      }).catch(err = > {
        next(err,null)
      })
    }
  }
```



### 再看看saga的take是什么？


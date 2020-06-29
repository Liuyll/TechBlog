### ts

[mocha ts依赖](https://42coders.com/testing-typescript-with-mocha-and-chai/)



#### 一个实例

```
// test/clock.test.tsx
import { App } from '../src/app'
import { expect } from 'chai'
import Enzyme,{ shallow } from 'enzyme'
import React from 'react'
import Adapter from 'enzyme-adapter-react-16'

Enzyme.configure({
    adapter: new Adapter()
})

describe('First', () => {
    it('returnValue equal 123',() => {
        const Wrapper = shallow(<App />)
        expect(Wrapper.children()).to.have.length(1)
    })
})
```

```
"test": "mocha --require ts-node/register node_modules/@babel/register test/clock.test.tsx"
// 指定目录可以用shell
mocha $(find test -name '*.spec.tsx')
```

// TODO:  babel怎么配置?



### window global undefined

你可能会调用formData这些dom上的实例

此时window.formData会报错

`referenceError:window is not defined`



此时安装`jsdom jsdom-global`

```
add:mocha -r jsdom-global/register
```








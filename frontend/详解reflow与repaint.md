### 一个元素从加入到呈现经历了什么？

![image-20200216022602636](/Users/liuyl/Library/Application Support/typora-user-images/image-20200216022602636.png)

这是`devtools performance`的性能报告，我们可以看到，一个元素从加入DOM到呈现到页面上会经历以下五个步骤：

1. Recalculate style
2. <b>Layout</b> (reflow)
3. Update <b>Layer Tree</b>
4. <b>Paint</b> (repaint)
5. Composite Layers



### 定义reflow

> 在行为上，reflow表现为:非仅当前Graphics Layers变动

#### 什么会reflow

+ 该属性不止影响自身元素，而且可能影响其他元素布局(任何文本流的属性:height,width,etc)

+ js显式获得的属性(height,width,clientWidth,offsetWidth etc,这些东西都需要浏览器重新计算布局才能获得),大概有以下几种:

  1. 相对计算的: Element.offsetTop/clientTop

  2. 针对自身的: Element.clientHeight/style.height 

  3. 针对整体的: Element.scrollTop ..
  4. Js API: getComputedStyle/getBoundingRect etc...
  5. 修改css样式表

#### 浏览器优化

+ 浏览器对reflow会做patch render，也就是说会对在某个时间内reflow的行为进行合并。

+ 浏览器只能优化render带来的reflow，而不能优化js计算带来的reflow，这意味着要尽量少在代码里获得元素的属性，若一定要获取，请善用缓存。

### 定义repaint

> 一言以蔽之，repaint是浏览器重绘制<u><b>当前</b></u>图层的过程。

#### 什么会repaint

+ 绝对只会影响自身的属性(也就是脱离文本流的属性):
  1. transformX(Y) (这也是为什么transfrom百分比是针对自身的)
  2. visibility
  3. background



### 区分repaint和reflow

其实我们可能已经明白，要把reflow 优化为 repaint，只需要把更改全局图层的操作移动到当前图层进行即可。

css3提供了一个优化属性:`will-change`,它会为当前元素开启一个新的图层



### 关于reflow的优化

+ 缓存会导致reflow的元素的属性

+ 缓存多次reflow至一次修改:

  + 不要修改css样式表，采取`cssText`的修改方法 

    `eg: e.cssText += ';height:300px;width:300px;'`

  + 采取className的方式切换样式

    `eg: e.classList += 'newClassName'`

  ps:浏览器本身会进行`batch render`，所以这样优化的效果不是很大

+ 下线元素后进行操作:

  + `display:nont`
  + `createFragment`

### 什么是Composite

> ![img](https://user-gold-cdn.xitu.io/2020/1/7/16f7edfc2027de42?imageslim)

浏览器把整个页面渲染分为了几个层:

1. Dom 
2. Render Object 记录node内容，向GraphicsContext(绘图上下文)发起绘制请求

3. RenderLayers 复制处理dom子树 <b>CPU绘制</b>
4. GraphicsLayers 储存该层的GraphicsContext <b>GPU绘制</b>

该层级中，<b>后层级的处理前一层级</b>

> Composite Layers 就是GPU合成所有的GraphicsLayers

有一下几个重点:

+ GraphicsLayers由GPU绘制，速度远胜于RenderLayers
+ 一旦开启硬件加速，则会提升为GraphicsLayers(这也是`translateZ`起效的原因)
+ 若非进过profiling，不要滥用硬件加速


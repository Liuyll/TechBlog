### CSS3 Animation

它有几个必须要讲的重要属性：

+ animation-delay
+ animation-timing-function
+ animation-fill-mode
+ animation-iteration-count
+ animation-play-state



#### delay

delay规定了从x开始动画，为负则从后往前



#### fill-mode

#### 使用图片来表示 `translateY` 的值与 **时间** 的关系:

- 横轴为表示 **时间**，为 0 时表示动画开始的时间，也就是向 box 加上 on 类名的时间，横轴一格表示 `0.5s`
- 纵轴表示`translateY`的值，为 0 时表示 `translateY` 的值为 0，纵轴一格表示 `50px`

1. `animation-fill-mode: none`
   ![animation-fill-mode: none](https://segmentfault.com/img/bVyA0C)
2. `animation-fill-mode: backwards`
   ![animation-fill-mode: before](https://segmentfault.com/img/bVyA0D)
3. `animation-fill-mode: forwards`
   ![animation-fill-mode: after](https://segmentfault.com/img/bVyA0F)
4. `animation-fill-mode: both`
   ![animation-fill-mode: both](https://segmentfault.com/img/bVyA0O)

+ 为none时，动画生效前后不添加任何style属性
+ both，动画生效前后分别添加from或to的属性



#### iteration-count

生效次数，可以为小数(表示起效到百分之多少)

也可以为Infinity



#### direction

+ reverse 简单反向倒退
+ alternate 反向播放，且timing-function 也会被同步反向
+ alternate-reverse



#### play-state

暂停/开始
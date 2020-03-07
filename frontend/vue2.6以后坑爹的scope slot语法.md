## 前言

不得不说，vue一天一个样，偷跑了一下3.0的源码，全部都不一样了。

好久没写vue，发现slot-scope已经要被废弃，赶紧看一下新语法长什么样子。



## 只有默认插槽的情况下

```
// child
<template>
  <div>
    <slot :user="go">{{name}}</slot>
  </div>
</template>
```

```
// father
<template>
  <child v-slot="default1">{{default1.user.a}}</child>
</template>
```

<b>跟原来唯一的区别是，现在可以给插槽命名了，用命名后的插槽去取值就行了</b>

什么？你嫌麻烦，那好，我们使用es6的析构

```
// father
<template>
  <child v-slot={user={default:'hahahha'}}>{{user.a}}</child>
</template>
```





## 有多个插槽的情况

```
// child 
<template>
  <div>
    <slot :user="go">{{name}}</slot>
    <slot name="fuck" :nouser="nogo"></slot>
  </div>
</template>

<script>
export default {
  data() {
    return {
      a: 1,
      name: "erzi",
      go: {
        a: 1,
        b: 1
      },
      nogo:{
          a:3,
          b:3
      }
    };
  }
};
</script>
```

```
// father
<template>
  <child>
    <template v-slot="slot1">{{slot1.user.b}}</template>
    <template v-slot:fuck="slot2">{{slot2.nouser.a}}</template>
  </child>
</template>
```

<b>好简单清晰，比以前的弱智slot-scoped清晰多了...</b>



## 只有默认插槽的简写

```
// father
<template>
  <child v-slot="slot1">
    {{slot1.user.b}}
  </child>
</template>
```

<b>方便吧，直接写在组件上就行了，再也不需要template来包裹了</b>



## v-slot的简写

如果你嫌v-slot麻烦的话，不妨使用#来替代它，但是要注意，不要加:号。

```
eg:v-slot:default = #default
```



## 简单解释一下原理

翻下源码就知道，slot被解析为一个函数，不管是不是scoped都在这个函数里被处理，所以我们可以绑定不同的作用于给不同的插槽，你甚至可以绑定函数和computed，毕竟本质就是函数嘛。
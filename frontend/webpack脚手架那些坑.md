### config

#### publicPath

publicPath一般是用来配置在线上环境时，cdn的相对路径。通过publicPath加上



### @types

typescript只引入@types文件夹下的声明文件

通过指定`typeRoots`修改`@types`为任意名字的文件夹，比如下面指定为`typings`

```
{
   "compilerOptions": {
       "typeRoots" : ["./typings"]
   }
}
```



通过指定`types`,来指定哪些包下的`typeRoots`要被引入

```
{
   "compilerOptions": {
        "types" : ["node", "lodash", "express"]
   }
}
```



[参考](https://www.tslang.cn/docs/handbook/tsconfig-json.html)
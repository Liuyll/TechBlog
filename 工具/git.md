### git rebase

#### rebase

注意一个点，rebase中间的节点，比如

```
c1 a
c2 b
c3 c
// drop c2时 c3也会死掉
```

+ reword 修改提交信息
+ edit 编辑某个节点
+ drop 删除某个节点和之后的节点

+ squash

  这个是很常用的方法，一般

  ```
  c1 a
  c2 b
  c3 c
  // 此时c3是最后提交
  // 一般压缩cb到a，千万不能压缩ab到c，否则会丢失代码
  ```

+ fixup

  跟`squash`不同的是，它不保留前一个提交的提交信息。一般在删除commit时使用这个。

### git pull

`git pull br1 br2`是以下命令的集合:

+ `git fetch br2 temp`
+ `git merge temp`
+ `git branch -d temp`



#### rebase

考虑这样的情况

```
A: cr1---cr2---cr3----
                     |---cra4---cra5

B: cr1---cr2---cr3---
										|---crb4---crb5
```

此时从B执行`git pull`肯定会导致cra4，cra5和crb4，crb5分叉。此时就会形成分叉的图，提交日志会不好看。

```
										|---cra4---cra5---
cr1---cr2---cr3---										|---head
										|---crb4---crb5---
```



使用rebase的话，git会把cra4和cra5的提交合并到crb4和crb5上，即

```
B: cr1---cr2---cr3---
										|---crb4---crb5---cra4---cra5---head
```





#### 冲突

merge后遇到冲突时:

+ 当前更改是自己写入的内容
+ 传入的更改拉取的内容

### git merge

分析merge的几种情况

![image-20201004133558753](/Users/liuyl/Library/Application Support/typora-user-images/image-20201004133558753.png)

#### --no-ff

故意添加no-ff情况下的分叉



#### ff(fast-forward)

类似于`rebase`，保证无分叉合并。

此时无需手动提交代码，如无冲突，直接把旧代码合并上本分支。

为保证ff，可以使用`git pull --ff-only`



#### no-ff

如上图，允许分叉合并。

需要一个新的提交来作为两个分叉分支(即使没有分叉)的新头部。

同时合并提交记录。



#### squash

简单的把merge分支的提交记录全部压缩为0。

可以理解为：只合并代码，不合并任何提交记录。

此时需要额外进行一次手动提交来记录合并代码的信息。

#### 



### git cherry-pick

以一次新提交合并某次提交的代码

<b>常用:</b>

+ 我们经常用`git pull`去拉取一个分支，如果不想把分支的提交记录全部拉过来，即希望使用`squash`的拉取方式，可以使用`cherry-pick`创建一个新提交来拉取。

+ 相反，如果想撤回代码，但不想撤回之前的提交记录，也可以使用`cherry-pick`重新创建一个提交来撤回代码，并且可恢复到任意提交记录里。

  



### diff

#### 分支diff

`git diff branch1 branch2 [path]`

#### 远端diff

`git diff branch1 origin/branch2 [path]`



### log

#### 查看远端仓库

我们有时候需要直接查看远端仓库的commit id

这时候直接使用

`git log remoteName/remoteBranch`

eg:`git log origin/master`

#### 美化日志

##### 限制输出行

利用`-n linecount `可以限制输出的行数



我们可以使用pretty来格式化输出

比如可以美化git log的输出,使用`git log --pretty=xxx`

##### oneline

可以用`oneline`来美化`git log --graph`的点线图

<b>用法：</b>

`git log --pretty=oneline`

也可以使用shortcut,`git log --oneline`

<b>实例:</b>

非`oneline`下的graph



![image-20201004124038258](/Users/liuyl/Library/Application Support/typora-user-images/image-20201004124038258.png)

使用`oneline`后的graph:

![image-20201004124102993](/Users/liuyl/Library/Application Support/typora-user-images/image-20201004124102993.png)

明显要简洁的多。
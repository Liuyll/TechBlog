## 提交



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

+ 试试看`git pull --rebase`

  此时，当前分支的分叉节点会放在远端分支的后面。这样的话，git在其他人拉取分支的时候就不会出现分叉了

  ```
  REMOTE: cr1---cr2---re1
  LOCAL: cr1---cr2---lo1
  git pull --rebase
  resolve conflict
  cr1---cr2---re1---lo1
  git push
  ```

+ 试试看`git rebase feature`

  此时跟`pull`的情况相反，它会把`feature`分支的头部作为当前头部加入到当前分支

  



![rebase](https://static.bookstack.cn/projects/git-recipes/27e2f1f44cfe11ba7bac3de25ae8e684.svg)

此时提交树被修改，自然的需要`git push --force`才能强行推上去。要么`git pull`来解决冲突问题。

不过强推会导致很麻烦的事情：

其他人的分支本来没有`feature`这个提交，自然也没有它的代码。此时再`pull`时，会多出`feature`的代码，又不会有`feature`的提交(因为头部都是正确的远端头部，feature是合并在master前面的)，所以会感到困惑。

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

实际上，`ff-only`只能处理分支不分离的情况，也就是说远程分支的头部也是当前分支的头部，此时git只需要滑动提交即可

```
...-o--C        <-- origin/feature
     \   `-_
      A--B--M   <-- feature
```

#### no-ff

如上图，允许分叉合并。

需要一个新的提交来作为两个分叉分支(即使没有分叉)的新头部。

同时合并提交记录。

很显然，`no-ff`不需要远端分支和当前分支的头部相同。



#### squash

简单的把merge分支的提交记录全部压缩为0。

可以理解为：只合并代码，不合并任何提交记录。

此时需要额外进行一次手动提交来记录合并代码的信息。



### git rebase 和 git merge --ff-only 的区别

上面提到过了，`ff-only`只支持远端和当前分支头部相同的情况。而rebase不一致。

`ff-only`的原理是滑动提交链，(实际上远端分支已经merge了当前分支)

`rebase`的原理是复制当前分支到远端的末尾，作为合并分支的新头部(也就是把当前分支头插进远端分支链表)

### git cherry-pick

以一次新提交合并某次提交的代码

<b>常用:</b>

+ 我们经常用`git pull`去拉取一个分支，如果不想把分支的提交记录全部拉过来，即希望使用`squash`的拉取方式，可以使用`cherry-pick`创建一个新提交来拉取。

+ 相反，如果想撤回代码，但不想撤回之前的提交记录，也可以使用`cherry-pick`重新创建一个提交来撤回代码，并且可恢复到任意提交记录里。

  



## diff

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



##### --abbrev-commit

使用`--abbrev-commit`来减少commit-id的长度



#### 图形化界面

当然我们可以用图形化界面来显示`git log`

执行`gitk --all `即可



## 撤销

git的撤销同样是门艺术，我们可以使用`revert`或者`reset`来处理不同的情况

### revert

revert的撤销很有趣，它跟`cherry-pick`一样是依靠重做来完成功能

```
a--->b--->c--->d
```

如果我们想要撤销`c`，但保留`d`的提交，此时`reset`肯定是做不到的。只能反向重做`c`，把c的提交撤销掉，但保留完全的日志。



### reflog

我们也可以用硬编码的方法去撤销操作。

使用`reflog`查看操作历史，然后回滚到历史版本。



## 信息

有一些查询信息的命令也值得记录



### git ls-remote

通过`ls-remote`，我们可以查询远端的标签

```
git ls-remote --tags /url/to/upstream/repo
```




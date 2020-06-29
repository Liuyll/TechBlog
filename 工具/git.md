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

#### ===

1. `git fetch tmp `
2. `git merge tmp`

#### ff(fast-forward)

当前分支与远端分支提交记录同步，也就是说无分叉

为保证ff，可以使用`git pull --ff-only`

#### no-ff

远端有其他提交记录，本地版本库与远端分叉，必须合并

#### rebase

消除no-ff情况下的分叉

#### merge --no-ff

故意添加no-ff情况下的分叉

#### 冲突

merge后遇到冲突时:

+ 当前更改是自己写入的内容
+ 传入的更改拉取的内容



### git cherry-pick

拉取某个分支上的某次提交代码



### diff

#### 分支diff

`git diff branch1 branch2 [path]`

#### 远端diff

`git diff branch1 origin/branch2 [path]`




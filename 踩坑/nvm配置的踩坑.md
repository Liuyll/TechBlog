踩坑问题：

npm install -g 全局安装却找不到命令



解决：

npm的全局安装路径没有被加入环境变量

```
npm prefix -g // 查看npm全局安装路径
```

```
echo ${npm prefix -g} >> ~/.bashrc && source ~/.bashrc 加入环境变量即可
```



注意点：

加入环境变量要判断是

1. 临时环境变量 `export PATH=$PATH:xxx`
2. 当前用户环境变量 `~/.bashrc`

3. 永久 `/etc/profile`
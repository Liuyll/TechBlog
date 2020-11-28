### 调试

通过`ssh xx -vv`来查看调试信息，排除bug



### 添加公钥

主机储存位置：`~/.ssh/authorized_keys`

`ssh-keygen -t rsa xxx`：生成公钥

或者直接使用: `ssh-copy-id -i [identify_file] root@server `来拷贝



+ ssh-keygen
  + -t: 指定加密方式
  + -C: 指定提示信息
## GCP ##

先在sshd_config上2修改参数，使得可以通过密钥登录

然后在本地端要重生成ssh-keygen 

此时的命令中要有-R参数， 

如：

```SH
ssh-keygen -R 13.45.67.87 #13.45.67.87为gcp的ip地址
```

这个时候密钥已经
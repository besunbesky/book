# Git 设置代理

1.设置代理:

```shell
git config --global http.proxy http://IP:Port
```

2.代理设置完成后，查看设置是否生效：

```shell
git config -–get -–global http.proxy
```

3.删除代理设置

```shell
git config --global --unset http.proxy
```

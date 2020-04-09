因误操作，将mysql相关文件的权限都改成了777，导致启动mysql失败。

![](/Users/masonhua/Library/Application%20Support/marktext/images/2020-04-08-18-06-07-image.png)

从文档看，Linux/Unix基于安全的考虑，要求my.cnf文件不可被其他用户更改。

![](/Users/masonhua/Library/Application%20Support/marktext/images/2020-04-08-18-09-36-image.png)

修改my.cnf权限为644即可。

```
chmod 644 my.cnf
```

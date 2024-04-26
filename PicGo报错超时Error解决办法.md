默认图床是github，考虑了几种情况：

1. 代理的问题：搜索得到的方案，在浏览器设置搜索代理，打开系统代理设置，切换手动代理的状态，没用

2. 图床的问题：检查了更新；在github仓库上传了图片，图床相册没有更新，直接在图床程序内上传图片也失败；考虑过监听端口占用的问题，似乎无关；检查了github图床的配置，路径没有问题

3. 网络的问题：看了picgo.log日志文件，报错为RequestError: Error: connect ECONNREFUSED 127.0.0.1:443

在PicGo的github仓库issue搜索该报错，有一模一样的[报错](https://github.com/Molunerfinn/PicGo/issues/911)，据开发者说是运营商DNS解析https://api.github.com 域名有问题，picgo没法帮你解决网络问题。

处理方式：换dns域名解析服务器、换host

具体解决方式：

1. 查找 api.github.com 的正确域名（不要直接 ping，建议通过 [DNS 查询工具](https://tool.chinaz.com/dns/)进行查询）

2. 将其添加到对应的 hosts 文件中

3. hosts文件在末尾加入：
```shell
 140.82.112.6 api.github.com
 140.82.114.5 api.github.com
 //github域名会定期更新，如果不行需要自行查询
```
4. 刷新本地 dns 缓存：cmd执行命令行ipconfig /flushdns


> 原文链接：https://blog.csdn.net/m0_53277621/article/details/125121042
# dpdk-doc
DPDK文档中文版（17.05.0-rc4）
使用[Sphinx](http://sphinx-doc.org/)构建，主题使用的是[theme](https://github.com/snide/sphinx_rtd_theme)。
## 使用方法
```
#安装Sphinx
[root@host ~]# pip install Sphinx

#安装主题
[root@host ~]# pip install sphinx-rtd-theme

#构建中文文档
[root@host ~]# cd guides_zh
[root@host ~]# sphinx-build . html

```
构建完成的文档在 guides_zh/html/index.html

在线版本 [http://www.0211.life/zh](http://www.0211.life/zh)

# IntelliJ_IDEA_License_Server本地搭建教程

内容参考IntelliJ IDEA License Server本地搭建教程 Window版

1.下载软件包

最新版下载链接

磁力链接: magnet:?xt=urn:btih:2289E4F8CEB346AC44E54C8C0DA706CC537301AA
```bash
cd /usr/local/software
wget https://coding.net/u/ilanyu/p/IntelliJIDEALicenseServer/git/raw/master/IntelliJIDEALicenseServer_linux_amd64
```

2.运行注册服务

下载后，从压缩包里找到 适合自己版本的文件 
这里使用 linux举例 
IntelliJIDEALicenseServer_linux_amd64 文件放到 `/usr/local/software` 目录

# 使用su权限
# 进入/usr/local/software 目录
```bash
cd /usr/local/software
```
#设置 IntelliJIDEALicenseServer_linux_amd64可执行权限
```bash
chmod +x  ./IntelliJIDEALicenseServer_linux_amd64
# 运行注册服务
# -p 配置端口号
# -u 配置注册用户名
./IntelliJIDEALicenseServer_linux_amd64 -p 1017 -u yihy
```

3.使用

在idea注册界面选择授权服务器，填写http://127.0.0.1:1017 ，然后点击“OK”，如图
![image-20190505184338247](images/image-20190505184338247.png)
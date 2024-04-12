这里所有提到的python程序都是自己写的，因为只考虑了自己的需求，没有考虑到其他情况，所以要拷贝使用的话还是根据自己的情况改下比较好

> 代码位置：https://github.com/Leah-98/Python-Automation/tree/main/Gitbook



将自己的笔记处理成Gitbook最大的好处是可以全局搜索



# 第一步 准备md笔记仓库



**1.将所有文件处理成md，或引用到md文件中**

gitbook是只能够渲染markdown页面的。

我之前的笔记里，有一部分比较古早的笔记来自evernote、有道云的导出，是网页格式。网页格式我就转成PDF了，引用写在除了根目录的README.md文件夹下（append_other_files_to_readme.py）。

还有些零零散散的txt笔记，就直接将后缀改成md了（txt_to_md.py）



**2.将md引用的图片放置在md文件的同级目录的子文件夹下，子文件夹命名为md文件的名字_imgs**

之前的笔记引用图片比较混乱，有的是网络图片，有的是typora帮我写了绝对路径，但是在各个电脑上操作同一个笔记仓库，绝对路径不一致，有的以相对路径放在imgs文件夹下，但imgs文件夹在笔记仓库的各个位置，也比较乱。

还是用python程序处理，统一放置到md文件的同级目录的子文件夹下，子文件夹命名为md文件的名字_imgs，md里的图片引用重写为相对路径（reformat_imgs.py）

另外我自己的书摘很多时候是直接拍的实体书，手机照片太大了，需要压缩下（compress_pictures.py）



**3.生成SUMMARY.md**

将所有的md文件按文件路径写好SUMMARY.md，如果是子文件夹且没有README.md，则自己创建一个（generate_summary.py）

比gitbook init会好用一些，gitbook init生成文件有各种bug，且没办法根据所有md文件生成summary





# 第二步 准备Gitbook搭建环境

直接看的网络上的教程，踩了一堆坑。

> 这个教程太简单了，可以先看一遍了解一下大概，但不要按里面的流程走
>
> https://tonydeng.github.io/gitbook-zh/
>
> 这个更详细更好
>
> https://1927344728.github.io/fed-knowledge/tools/Gitbook/



**0.踩坑前置**

刚开始直接去官网下载安装的

> 官网：https://nodejs.org/en

但是官网这个版本(v18.x)太新了，gitbook有各种适配问题，后来降级成(v.10.x才正常使用gitbook)

讲一下踩坑过程吧，使用v18.x版本的nodejs，安装gitbook-cli后，在命令行调用gitbook init，报错：

```shell
/home/travis/.nvm/versions/node/v12.18.3/lib/node_modules/gitbook-cli/node_modules/npm/node_modules/graceful-fs/polyfills.js:287
      if (cb) cb.apply(this, arguments)
                 ^
TypeError: cb.apply is not a function
    at /home/travis/.nvm/versions/node/v12.18.3/lib/node_modules/gitbook-cli/node_modules/npm/node_modules/graceful-fs/polyfills.js:287:18
    at FSReqCallback.oncomplete (fs.js:169:5)
```

> 解决方法：https://stackoverflow.com/questions/64211386/gitbook-cli-install-error-typeerror-cb-apply-is-not-a-function-inside-graceful

需要更新下graceful-fs版本

我使用npm更新的时候还踩了个坑，调用任何命令都是报错 ERR! Cannot read property 'insert' of undefined

> 解决方法：https://github.com/npm/cli/issues/4876

就是需要修改镜像到https://registry.npmmirror.com

如果还需要设置proxy的话，这里有教程

> npm设置代理：https://blog.csdn.net/yanzi1225627/article/details/80247758

更新完之后，gitbook 调用任何命令没有反应，没有报错没有任何信息，本地目录没有任何变化，之后就将nodejs降级成了10.x

期间还有一个坑，gitbook和gitbook-cli是不能共存的，使用gitbook-cli(gitbook的命令行工具)得把gitbook卸载掉，安装完gitbook-cli第一次调用gitbook命令时，gitbook -V这种看版本号的也算，gitbook-cli就会去安装对应版本的gitbook

因为我每次调用gitbook命令，都会卡在install gitbook这一步，查了下网上的方法，可以克隆gitbook的github仓库然后自己install自己link

```shell
git clone https://github.com/GitbookIO/gitbook.git
cd gitbook
npm install
npm link
```

首先gitbook仓库里面的说明是使用bun安装而不是npm，其次这样安装还是相当于npm install gitbook，gitbook和gitbook-cli是冲突的，所以这个方法是错的。

下面说正确的方法



**1.安装nvm(nodejs的版本管理工具)和10.x版本的nodejs**

根据自己系统的情况二者选一

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

bash
Copy code
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```

安装nodejs 10.x版本并使用

```shell
nvm install 10
nvm use 10
```

这里还踩了个坑，上面的步骤中会安装nodejs 10.x版本适配的npm，但是我之前使用的18.x版本，切换使用nodejs 10.x版本后，npm的版本没有切换，调用npm任何命令都会报错，后来且切换回使用18版本，再卸载npm，这里的卸载操作会卸载最新版本npm，切换使用10.x版本后，就能正常使用到较低版本的npm了

```shell
nvm use 18
npm uninstall npm -g
```

-g的意思是将操作实施到全局，这个命令之后也会用到



**2.安装使用gitbook-cli&gitbook**

```shell
npm install gitbook-cli -g
```

`npm install -g` 命令用于全局安装 Node.js 模块。这意味着安装的模块将被放置在一个特定的目录中，通常是你的系统的全局 `node_modules` 目录下，而不是当前项目的 `node_modules` 目录下。

如果在 `npm install` 命令中不使用 `-g`（全局）参数，那么该命令将会在当前项目目录下的 `node_modules` 文件夹中安装指定的包，而不是在全局环境中安装。

```shell
gitbook -V
```

查看gitbook版本，第一次调用任何gitbook命令会去安装gitbook

再次运行gitbook -V可以看到gitbook-cli和gitbook的版本

注意不要自己npm install gitbook，要通过gitbook-cli



**3.预览gitbook网站，进行修复**

将命令行切换到自己的自己的md源文件根目录下，运行下面的命令自动生成README.md、SUMMARY.md

建议跑脚本，gitbook init生成文件有各种bug，且没办法根据所有md文件生成summary

```shell
gitbook init
```

注意目录下md文件和子文件不要包含“#”这个字符，也就是说生成的SUMMARY.md引用中不要有“#”字符，如果有的话会导致生成的gitbook页面点击左部的相应导航条目无法跳转

另外md文件中任何位置不要有连续的大括号，比如{{ 和 }}

> 相关说明：https://wkcom.gitee.io/gitbook%E7%9A%84%E4%BD%BF%E7%94%A8/3%E3%80%81gitbook%E8%BF%9E%E7%BB%AD%E5%A4%A7%E6%8B%AC%E5%8F%B7%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E5%BC%8F.html

大括号中间加个空格就能解决

预览gitbook网站

```shell
gitbook server
```

在浏览器打开本地端口page查看

```shell
http://localhost:4000/
```



# 第三步 搭建.io在线网站

有很多解决方案，我选择直接在Github上传静态网站。



生成gitbook静态网站，在自己的md源文件根目录下运行：

```shell
gitbook build
```

会生成个_book目录，里面放置的就是静态网站代码



只要仓库里有gh-pages分支，就可以根据下面的连接访问Gitbook Pages

```shell
https://githubusername.github.io/reponame
```

所以步骤就是：

1. 创建个空的github仓库
2. 切出个gh-pages分支，上传静态网站
3. 我需要多个gitbook，所以我在根目录下创建个子文件夹，在这个子文件夹下上传对应的静态网站

那我就可以根据下面路径访问多个Gitbook

```shell
https://githubusername.github.io/reponame/subfoldername1
https://githubusername.github.io/reponame/subfoldername2
```



暂时没有找到多个github仓库生成同一githubuser下多个gitbook网站的方法








































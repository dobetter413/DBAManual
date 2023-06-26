# **学习使用GitBook**

## 1.安装nodejs 环境
> GitBook 是一个基于 Node.js 的命令行工具，下载安装 Node.js，安装完成之后，你可以使用下面的命令来检验是否安装成功。实测，10.21.0 版本可能安装成功gitbook。（如果版本太高可能导致gitbook 安装失败）

```
$ node -v
v10.21.0
$ 
```

## 2.安装GitBook
```
npm install gitbook-cli -g
```
>在mac 环境中需要使用sudo 权限安装，windows 下则需要以管理员权限打开cmd 进行安装

验证安装是否成功
```
$ gitbook -V
CLI version: 2.3.2
GitBook version: 3.2.3
$ 
```

## 3.初始化目录
在编写新的电子书前应该先创建一个目录，并在该目录下进行初始化

```
$ gitbook init
warn: no summary file in this book 
info: create README.md 
info: create SUMMARY.md 
info: initialization is finished 
$ 
```

初始化完成够会在目录下生成两个文件，如下所示
```
$ ls
README.md       SUMMARY.md
$ 
```
> README.md 为说明文档，SUMMARY.md 为书的章节目录


## 4.预览
```
$ gitbook serve
Live reload server started on port: 35729
Press CTRL+C to quit ...

info: 7 plugins are installed 
info: loading plugin "livereload"... OK 
info: loading plugin "highlight"... OK 
info: loading plugin "search"... OK 
info: loading plugin "lunr"... OK 
info: loading plugin "sharing"... OK 
info: loading plugin "fontsettings"... OK 
info: loading plugin "theme-default"... OK 
info: found 3 pages 
info: found 4 asset files 
info: >> generation finished with success in 0.7s ! 

Starting server ...
Serving book on http://localhost:4000
```

> `gitbook serve`命令会在本地启动4000 端口，通过浏览器访问127.0.0.1:4000 就可以进行预览。

## 5.生成
```
$ gitbook build
info: 7 plugins are installed 
info: 6 explicitly listed 
info: loading plugin "highlight"... OK 
info: loading plugin "search"... OK 
info: loading plugin "lunr"... OK 
info: loading plugin "sharing"... OK 
info: loading plugin "fontsettings"... OK 
info: loading plugin "theme-default"... OK 
info: found 3 pages 
info: found 4 asset files 
info: >> generation finished with success in 0.6s ! 
$ 
```
以上命令会在该目录下生成 _book 目录，通过浏览器打开_book/index.html 即可访问


## 6.目录结构
GitBook 基本目录结构如下:
```
.
├── book.json
├── README.md
├── SUMMARY.md
├── chapter-1/
|   ├── README.md
|   └── something.md
└── chapter-2/
    ├── README.md
    └── something.md
```


## 使用教程
http://caibaojian.com/gitbook/
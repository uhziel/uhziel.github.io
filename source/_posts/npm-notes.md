---
title: npm 使用笔记
date: 2020-12-19 15:12:37
tags:
  - npm
  - nodejs
---

npm 是 Node Package Manager 的缩写，是 nodejs 的包管理器。下面以常用场景为例做下说明。

## 初始化一个项目

在空文件夹中，执行下面命令，按照提示进行操作即可。

```bash
$ npm init
```

## 安装需要的包到项目中

下面的命令，会安装 express 进来，同时记录其到 package.json 的 `dependencies`(依赖)里。
```bash
$ npm install express
```

如果仅仅是在开发中需要，比如只是用 tslint 在开发中进行静态代码检查。那么可以加上选项 `--save-dev` 放到 `devDependencies` 里。
```bash
$ npm install --save-dev eslint
```

如果是 clone 别人的项目，因为 package.json 已经记录着所有需要的包信息，直接执行 `npm install` 就会将所有依赖安装好。

## 其他常用的命令

```bash
$ npm start # 启动项目
$ npm test # 测试项目
```

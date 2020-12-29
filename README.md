<h1 style="color: #E97018;"><center>docsify 使用指南</center></h1>

> 跟着文档手把手教你如何搭建一个功能齐全的 `docsify` 文档网站.

# 第一步：安装及初始化
## 安装 docsify-cli
```bash
$ npm i docsify-cli -g
```
## 初始化项目（books）
```bash
$ docsify init ./books
```
## 启动项目
- 不进入目录启动
```bash
$ docsify serve ./books
```
- 进入目录启动
```bash
$ cd ./books
$ docsify serve
```
## 初始化文件描述
- `index.html` 入口文件
- `README.md` 会做为主页内容渲染
- `.nojekyll` 部署到 GitHub Pages 有用，用于阻止 GitHub Pages 忽略掉下划线开头的文件


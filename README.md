# MIT 6.S081

记录自己的学习笔记，以及 Lab


## 配置

### 下载项目

```sh
$ git clone git://g.csail.mit.edu/xv6-labs-2022
```

### 配置 clangd

在 `.clangd` 内配置头文件索引范围，这样就可以跳转到对于文件了

```sh
CompileFlags:
  Add: [-I/home/sy/MIT-6.S081/xv6-labs-2022]
```
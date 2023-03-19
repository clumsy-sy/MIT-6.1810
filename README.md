# MIT 6.1810

原课程 MIT-6.S081，本项目主要记录自己的学习笔记，以及 Lab 结果


## 配置

### 下载项目

```sh
$ git clone git://g.csail.mit.edu/xv6-labs-2022
```
完成实验用的 [fork](https://github.com/clumsy-sy/xv6-labs-2022-fork)
```sh
$ git clone https://github.com/clumsy-sy/xv6-labs-2022-fork.git
```

### 配置 clangd

在 `.clangd` 内配置头文件索引范围，这样就可以跳转到对应文件了

```sh
CompileFlags:
  Add: [-I/home/sy/MIT-6.S081/xv6-labs-2022]
```

## 作业

已完成：

- [x] [Lab：Xv6 and Unix utilities](https://github.com/clumsy-sy/MIT-6.S081/tree/main/Lab:%20Xv6%20and%20Unix%20utilities)
- [x] [Lab: system calls](https://github.com/clumsy-sy/MIT-6.S081/tree/main/Lab:%20system%20calls)
- [x] [Lab: page tables](https://github.com/clumsy-sy/MIT-6.1810/tree/main/Lab:%20page%20tables) (完成20 年题目，22年题目暂时未完成)
- [x] [Lab: traps](https://github.com/clumsy-sy/MIT-6.1810/tree/main/Lab:%20traps)
- [x] [Lab: Copy-on-Write Fork for xv6](https://github.com/clumsy-sy/MIT-6.1810/tree/main/Lab:%20Copy-on-Write%20Fork%20for%20xv6)
- [x] [Lab: Multithreading](https://github.com/clumsy-sy/MIT-6.1810/tree/main/Lab:%20Multithreading)
- [ ] Lab: networking
- [ ] Lab: locks
- [ ] Lab: file system
- [ ] Lab: mmap (hard)





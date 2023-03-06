# Lab: Xv6 and Unix utilities

This lab will familiarize you with xv6 and its system calls.

## Build and run xv6(easy)

```sh
# build xv6 and startup qemu
$ make qemu
```

然后就进入了 `xv6` 的 `shell` 

```sh
# quit
$ Ctrl-a x
```

## sleep(easy)

在 `xv6-labs-2022/user` 目录下新建 `sleep.c` 文件，写入以下代码
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
  int times = 0;

  if(argc <= 1 || argc > 2) {
    fprintf(2, "usage: sleep pattern [times]\n");
    exit(1);
  }
  times = atoi(argv[1]);
  sleep(times);
  exit(0);
}
```

配置 `Makefile` 文件，在 `UPROGS=\` 变量下添加 `$U/_sleep\`

```sh
# build and run
$ make qemu
# test
$ sleep 100
# 一段时间后
$
```

测试
```sh
❯ ./grade-lab-util sleep
make: 'kernel/kernel' is up to date.
== Test sleep, no arguments == sleep, no arguments: OK (1.4s) 
== Test sleep, returns == sleep, returns: OK (0.8s) 
== Test sleep, makes syscall == sleep, makes syscall: OK (1.0s)
```

## pingpong(easy)

在 `xv6-labs-2022/user` 目录下新建 `pingpong.c` 文件，写入以下代码

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]){
  // init pipe
  int pipePtoC[2], pipeCtoP[2], status;
  pipe(pipePtoC);
  pipe(pipeCtoP);

  // fork
  int pid = fork();
  if(pid > 0) {
    /*parent should send a byte to the child*/
    int p_pid = getpid();
    close(pipePtoC[0]);
    write(pipePtoC[1], "Hello", 5);
    close(pipePtoC[1]);

    wait(&status);
    char read_buff[6] = {0};
    close(pipeCtoP[1]);
    read(pipeCtoP[0],read_buff, 5);
    if(*read_buff == 'H'){
      fprintf(1, "%d: received pong\n", p_pid);
    } else {
      fprintf(1, "ERROR!");
    }
    close(pipeCtoP[0]);

  } else if(pid == 0){
    int c_pid = getpid();
    char read_buff[6] = {0};
    close(pipePtoC[1]);
    read(pipePtoC[0],read_buff, 5);
    if(*read_buff == 'H'){
      fprintf(1, "%d: received ping\n", c_pid);
    } else {
      fprintf(1, "ERROR!");
    }
    close(pipePtoC[0]);

    close(pipeCtoP[0]);
    write(pipeCtoP[1], "Hello", 5);
    close(pipeCtoP[1]);
    exit(0);
  } else {
    write(2, "ERROR: fork unsuccess!", 22);
  }
  exit(0);
}
```

配置 `Makefile` 文件，在 `UPROGS=\` 变量下添加 `$U/_pingpong\`

```sh
# build and run
$ make qemu
# test
$ pingpong
4: received ping
3: received pong
```

测试
```sh
❯ ./grade-lab-util pingpong
make: 'kernel/kernel' is up to date.
== Test pingpong == pingpong: OK (1.0s) 
```

## primes (moderate)/(hard)

用管道实现一个 `Eratosthenes` 筛，在 `xv6-labs-2022/user` 目录下新建 `primes.c` 文件，写入以下代码

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(){
  int numArray[35], cnt = 0;
  for(int i = 2; i <= 35; i ++)
    numArray[cnt ++] = i;
  
  while(1) {
    if(cnt == 0) break;

    printf("prime %d\n", numArray[0]); // print primes
    // creat pipe
    int pip[2];
    pipe(pip);
    
    int pid = fork();
    if(pid < 0) {
      printf("ERROR fork!\n");
    } else if(pid > 0) {
      // parent
      close(pip[0]);
      for(int i = 1; i < cnt; i ++)
        if(numArray[i] % numArray[0] != 0)
          write(pip[1], &numArray[i], sizeof(int));
      close(pip[1]);
      wait(0);
      break; // important
    } else {
      // child
      cnt = 0;
      close(pip[1]);
      while(1) {
        int now, ret;
        ret = read(pip[0], &now, sizeof(int));
        if(ret == 0) break;
        numArray[cnt++] = now;
      }
      close(pip[0]);
    }
  }
  exit(0);
}
```

配置 `Makefile` 文件，在 `UPROGS=\` 变量下添加 `$U/_primes\`

```sh
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
$ 
```

测试

```sh
❯ ./grade-lab-util primes
make: 'kernel/kernel' is up to date.
== Test primes == primes: OK (1.9s) 
```

## find (moderate)

实现一个 `find [path] [filename]`，在 `xv6-labs-2022/user` 目录下新建 `find.c` 文件，写入以下代码

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char*
fmtname(char *path)
{
  static char buf[DIRSIZ+1];
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;

  // Return blank-padded name.
  if(strlen(p) >= DIRSIZ)
    return p;
  memmove(buf, p, strlen(p));
  // different from 'ls.c', here fill with '\0'
  memset(buf+strlen(p), '\0', DIRSIZ-strlen(p));
  return buf;
}

void
find(char *path, char *filename)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }
  
  switch(st.type){
  case T_DEVICE:
  case T_FILE:
    if(strcmp(fmtname(path), filename) == 0)
      printf("%s\n", path);
    break;

  case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("find: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf("find: cannot stat %s\n", buf);
        continue;
      }
      char *fname = fmtname(buf);
      if(strcmp(fname, ".") != 0 && strcmp(fname, "..") != 0)
        find(buf, filename);
    }
    break;
  }
  close(fd);
}

int
main(int argc, char *argv[])
{
  if(argc < 2){
    fprintf(2, "Usage: find path filename...\n");
    exit(1);
  } else {
      find(argv[1], argv[2]);
  }
  exit(0);
}
```
配置 `Makefile` 文件，在 `UPROGS=\` 变量下添加 `$U/_find\`

```sh
init: starting sh
$ echo > b
$ mkdir a
$ echo > a/b
$ mkdir a/aa
$ echo > a/aa/b
$ find . b
./b
./a/b
./a/aa/b
```

测试

```sh
❯ ./grade-lab-util find  
make: 'kernel/kernel' is up to date.
== Test find, in current directory == find, in current directory: OK (1.6s) 
== Test find, recursive == find, recursive: OK (1.0s) 
```

## xargs (moderate)

实现一个 `xargs command argvs...`，在 `xv6-labs-2022/user` 目录下新建 `finxargsd.c` 文件，写入以下代码

```c
#include "kernel/types.h"
#include "kernel/param.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if(argc < 3) {
    fprintf(2, "Usage: xargs command (argv)...\n");
    exit(1);
  }
  int totalargc = 0;
  char *totalargv[MAXARG];
  for(int i = 1; i < argc; i ++)
    totalargv[totalargc ++] = argv[i];

  char buf[512] = {0};
  char ch;

  int cnt = 0;
  while(read(0, &ch, 1)) {
    if(ch == 0) {
      buf[cnt++] = '\0';
      totalargv[totalargc] = (char*)malloc(cnt);
      memmove(totalargv[totalargc ++], buf, cnt);
      break;
    }
    if(ch == ' ' || ch == '\n') {
      if(cnt > 0) {
        buf[cnt ++] = '\0';
        totalargv[totalargc] = (char*)malloc(cnt);
        memmove(totalargv[totalargc], buf, cnt);
        cnt = 0;
        totalargc ++;
        if(totalargc >= MAXARG) {
          fprintf(2, "too many args\n");
          exit(1);
        }
      }
      continue;
    }
    buf[cnt ++] = ch;
    if(cnt >= 512) {
      fprintf(2, "Buffer overflow!");
      exit(1);
    }
  }

  if(fork() == 0) {
    exec(totalargv[0], totalargv);
  } else {
    wait(0);
  }
  exit(0);
}
```
配置 `Makefile` 文件，在 `UPROGS=\` 变量下添加 `$U/_find\`

```sh
$ mkdir test
$ echo hello World > test/b
$ echo hello xv6 > b
$ find . b | xargs grep hello
hello World
hello xv6
$ 
```

测试

```sh
# test 1
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
# test 2
❯ ./grade-lab-util xargs
make: 'kernel/kernel' is up to date.
== Test xargs == xargs: OK (1.9s) 
$ $ 
```


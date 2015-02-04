---
layout: post
title: "Why CLONE_NEWNS is not usable"
date: 2015-2-4 23:24:00
---

通过docker了解到了container技术，然后又通过container技术了解到了namespace、cgroup等技术，果然技术和技术之间是相互联系的。

额。。。好像离题了，收回来。

首先，为啥去玩namespace，为了了解一下隔离技术看看能不能弄点神奇的东东出来，哈哈。

描素一下问题吧，先看一下下面的代码：

{% highlight c %}
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/types.h>
#include <signal.h>
#include <errno.h>
#include <sys/wait.h>
#include <string.h>
#include <sched.h>

#define STACK_SIZE (1024*1024)

int new_ns(void *nul)
{
    execl("/bin/bash", "/bin/bash", NULL);
    printf("Oops\n");
}

int main(int argc, char **argv)
{
    int res;
    int child_id;

    void *stack = (void *)calloc(STACK_SIZE, 1);
    child_id = clone(new_ns, stack+STACK_SIZE,
              CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    printf("child_id is %d\n", child_id);

    waitpid(child_id, &res, 0);
    return 0;
}
{% endhighlight %}

稍微解释一下，就是当运行该程序时，就启动一个新的子进程，让其跑在自己的命名空间中，做到包括UTS、IPC、PID、MNT的空间隔离，然后让子进程启动一个bash shell，进行交互。

了解了这些之后，看下我所谓的问题，我主要测试的是PID隔离以及MNT的隔离。

PID的隔离是没有问题的，执行下面的代码编译上面的代码，同时进入shell

    gcc -o test test.c
    sudo ./test        #CLONE_NEW*均需要sys_admin的权限，或者用setcap或者直接用root用户启动
    
进入shell之后输入

    echo $$
    
这时就可以发现当前的进程id为1，神奇的一b吧，不过如果你在shell内输入ps aux，这时候列出来的依然是父空间的进程，这是怎么回事。

由于ps是去读proc文件系统列出进程列表的，但是当进入子空间的shell之后，proc系统依然是父空间的proc，所以对proc分区重新进行mount即可

    mount --make-private -t proc proc /proc
    
为什么要用--make-private，一会儿说。

然后这个时候再运行ps aux就没有问题了。

继续，测一下MNT的空间隔离，在子空间进行根目录建立一个文件夹mytmp，然后mount一个临时分区上去，执行代码如下：

    sudo mkdir /mytmp
    sudo mount -t tmpfs none /mytmp
    
然后到父空间运行mount，发现。。。诶，怎么他丫的还是看到了mytmp的挂载信息。

后面去看了一大堆博客，妈蛋，一个说这个问题的都没有，害我最后都在怀疑是不是我的内核不支持，不过转念一想，archlinux这更新最快的发行版都不支持，还有什么能支持来着。

功夫不负有心人啊，终于在unshare的man doc内找到了这么一段话

> mount namespace

>> Mounting  and  unmounting  filesystems  will  not  affect  the  rest  of  the  system  (CLONE_NEWNS  flag),  except  for filesystems which are explicitly marked as shared (with mount --make-shared; see /proc/self/mountinfo for the shared flags).
        
>> It's recommended to use mount --make-rprivate or mount --make-rslave after unshare --mount to make sure that mountpoints in the new namespace are really unshared from the parental namespace.
        
看到了吗？except  for filesystems which are explicitly marked as shared，日他个蛋的，赶紧查了一下/proc/self/mountinfo，发现根目录就是shared，接下去就知道该怎么办了

    sudo mount --make-private -o remount rootfs /
    
看到这里也知道--make-private是做什么用的了吧，我就不做解释了，接下去就一切很顺其自然了，在子空间进行mount，在父空间查看，ok了，父空间看不见子空间的mount结点了，在子空间在mount结点新建文件，再写点内容，跑去父空间看，嘿嘿，啥没有，good job！！！

这个问题描素完了，鼓号队带下去。。。
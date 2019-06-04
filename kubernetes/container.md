``` shell
int pid = clone(main_function, stack_size, SIGCHLD, NULL);
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```

```shell
$ mount -t cgroup 
cpuset on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cpu on /sys/fs/cgroup/cpu type cgroup (rw,nosuid,nodev,noexec,relatime,cpu)
cpuacct on /sys/fs/cgroup/cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct)
blkio on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
memory on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
...

```

```shell
$ ls /sys/fs/cgroup/cpu
cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks
```

```shell
root@ubuntu:/sys/fs/cgroup/cpu$ mkdir container
root@ubuntu:/sys/fs/cgroup/cpu$ ls container/
cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks
```

```shell
$ while : ; do : ; done &
```

```shell
$ cat /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us 
-1
$ cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us 
$ echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
$ echo 226 > /sys/fs/cgroup/cpu/container/tasks 
$ docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
$ cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_period_us 
100000
$ cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_quota_us 
20000
```

```c
#define _GNU_SOURCE
#include <sys/mount.h> 
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
char* const container_args[] = {
  "/bin/bash",
  NULL
};

int container_main(void* arg)
{  
  printf("Container - inside the container!\n");
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}

int main()
{
  printf("Parent - start a container!\n");
  int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWNS | SIGCHLD , NULL);
  waitpid(container_pid, NULL, 0);
  printf("Parent - container stopped!\n");
  return 0;
}
```

```shell
int container_main(void* arg)
{
  printf("Container - inside the container!\n");
  // 如果你的机器的根目录的挂载类型是 shared，那必须先重新挂载根目录
  // mount("", "/", NULL, MS_PRIVATE, "");
  mount("none", "/tmp", "tmpfs", 0, "");
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}
```

```shell
mkdir $HOME/chroot_test
mkdir -p $HOME/chroot_test/{bin,lib,lib64}
cp /bin/bash $HOME/chroot_test/bin/
cp /bin/ls $HOME/chroot_test/bin/
```

```shell
ldd /bin/ls
ldd /bin/bash
```

```shell
sudo chroot $HOME/chroot_test /bin/bash
```

```shell
lliu@lliu-VirtualBox:~/Documents/kubernetes-session/ufs$ tree
.
|-- ufs_a
|   |-- a
|   `-- x
`-- ufs_b
    |-- b
    `-- x
```

```shell
mkdir ufs_c
sudo mount -t aufs -o dirs=./ufs_a:./ufs_b none ./ufs_c
lliu@lliu-VirtualBox:~/Documents/kubernetes-session/ufs$ tree
.
|-- ufs_a
|   |-- a
|   `-- x
|-- ufs_b
|   |-- b
|   `-- x
`-- ufs_c
    |-- a
    |-- b
    `-- x
```

```shell
cat /proc/mounts | grep aufs
ls /sys/fs/aufs/
```

```shell
docker run -d ubuntu:latest sleep 3600
```

LowerDir：指向镜像层；
UpperDir：指向容器层，在容器中创建文件后，文件出现在此目录；
MergedDir：容器挂载点 ，lowerdir和upperdir整合起来提供统一的视图给容器，作为根文件系统；
WorkDir：用于实现copy_up操作。
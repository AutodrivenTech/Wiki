<!-- TITLE: 为实验室建立公用GPU服务器 -->
<!-- SUBTITLE: 多用户管理 -->

[原始链接](https://zhuanlan.zhihu.com/p/25710517)


# 需求的产生

现在深度学习风生水起，为了满足人民日益增长的计算需求，相信各个实验室都开始买起了显卡。然而毕竟显卡还是贵，做不到人手一块，所以只能以公用机器的形式出现了。

大家都在公用机器上跑实验，而各自所需要的软件（比如 Cuda、TensorFlow……）版本却可能不一样，这样很容易因为版本问题而导致程序无法运行。解决这些软件冲突问题是一个又繁琐又耗时的工作，而且常常弄得鸡飞狗跳，最终没有人可以正常运行实验。所以说，我们希望在公用的机器上能够有一定的管理，使得不同的用户不会相互影响。

这里我列出一些需求：

- 不同用户之间不能相互影响
- 用户要能方便地访问自己的“虚拟机”
- 用户要有足够大的权限，能自由地安装程序，能自由地访问网络
- 用户不被允许直接操作宿主机
- 用户要能够使用 GPU
- 用户能够方便地使用实验室的 NAS
- 为满足这些需求，额外的开销应该小得可以忽略
- 管理员应该能轻松地添加新的用户
- 公用机器的资源应该能得到最大化的利用
- 这套解决方案不应该太复杂

下面我将叙述我解决以上需求的方法，以供有需要的人参考。本文的受众应该是实验室的公用机器管理员，有一定的 Linux 基础，或者对此感兴趣的普通用户。

# 解决思路
从需求出发，首先解决的问题就是怎么做用户的隔离。最简单粗暴的方法无疑就是虚拟机，然而现在的家用显卡并不支持虚拟化，并且CPU虚拟化的额外开销还是很可观的，另外IO虚拟化的性能更是问题。

目前很多 Hypervisor 都支持 PCI Passthrough，然而如果采用这个技术的话，显卡就只能被一台虚拟机独占，其他虚拟机就无法使用这块显卡。有趣的是，一般的代码是远远无法占满GPU的，GPU利用率只会达到10%~30%左右。在这样的情况下，多个用户共享同一个GPU是合理的，也是提高硬件资源的利用率。

与之类似的就是CPU和内存资源的划分。使用虚拟化之后，一台虚拟机的CPU和内存基本上是定死的。然而，有时候大家需要用多一点的CPU，有时候则不需要那么多；有时候大家需要用巨多的内存，而有时候又只需要一点点。无疑这种定死的策略也是降低了硬件资源的利用率。

所以说我们完全不考虑虚拟化。

因为实验室公用机器是一个相对安全和可控的环境，没有安全上面的顾虑，并且现在主流的深度学习平台都是在 Linux 上，所以我们可以直接利用 Linux 内核提供的隔离机制来解决这个问题。因为只是做隔离，这里带来的额外开销微乎其微。并且因为硬件资源都是共享的，这样就能尽可能地利用硬件资源。

用来做隔离的方法有很多，成熟又出名的就有 Docker / OpenVZ / LXD / LXC 等等。OpenVZ 因为太过复杂就不考虑，LXD 感觉跟 LXC 差不了多少。所以我真正拿来比较的是 Docker 和 LXC。

Docker 现在用的非常多，用起来也非常方便。而 LXC 自从 Docker 出现之后便越来越少地被提及了。仔细考虑 Docker 和 LXC 的哲学和应用场景。Docker 更倾向于部署应用，更倾向于无状态；而 LXC 则相反，它想让人把它当做一台虚拟机来使用。再想想实验室的需求，我们需要的是让每个人有一个“虚拟机”，而不是每个人部署一个个应用。

诚然，不顾 Docker 的哲学，硬要把 Docker 当做虚拟机来用也不是不可以。然而这样一来，Docker 的接口反而成为一种累赘。再说，原先的 Docker 不正是 LXC 上面的一层封装吗？那既然我们的需求更符合 LXC 的哲学，那为什么不直接用 LXC 呢？

所以我们就选定了 LXC。

接下来是我们非常关心的 GPU 问题，在 LXC 容器中能使用 GPU 吗？好在 Linux 有硬件即文件的哲学，我们只要把宿主机中显卡设备对应的文件挂载到 LXC 容器中就能解决这个问题。

因为访问实验室 NAS 的需求非常普遍，让每个用户各自连接到 NAS 无疑不是一个好的选择，一是对用户来说麻烦，二是开了很多冗余的连接。所以说，我们可以在宿主机中把 NAS 挂载好，然后同样地把挂载好的目录挂载到 LXC 容器中。

要限制用户不能直接操作宿主机，很自然地想到我们可以让用户使用 ssh 进入到 LXC 容器中。然而这里就有几个问题要解决：

用户的 LXC 容器可能未启动，怎么使它启动呢？一个方法自然还是让用户登入到宿主机来启动 LXC 容器，然而正如前面所说，这与我们的需求矛盾。另一个方法是轮询所有 LXC 容器的状态，如果没有启动就自动启动。这个思路我感觉不是很优雅。

我的解决方法是编写一个脚本，把它作为用户在宿主机上的 Shell。在脚本中检查用户的 LXC 容器是否启动。如果没有，就启动它。顺带地，脚本还可以输出更多的信息，提示用户一些常用的操作。这样一来就解决了这个问题，并且因为用户的 Shell 已经不是传统的 /bin/bash 等，而是我们特制的脚本，所以用户在 ssh 进入宿主机之后只能执行这个脚本，而不能执行别的命令。这就很好地阻止了用户直接操作宿主机。

LXC 容器的 IP 是只有宿主机能访问得到的内网 IP，用户要怎么访问？有一个方法是先让用户登入宿主机，然后以宿主机作为跳板进入 LXC 容器。然而这样一是对用户来说麻烦，二是这样用户就能直接操作宿主机了，与我们的需求矛盾。

我的解决方法是给每个用户分配一个端口，利用 iptables 把这个端口转发到对应 LXC 容器的22端口。这样用户使用这个分配的端口连接宿主机，就相当于连接容器的22端口。

具体地说怎么在脚本里面得到这些端口呢？我的方法是在 /public/next-port 里面保存一个数字，即下一个新用户的端口，每次新增用户的时候递增。而每个用户的端口，则是保存在 /public/ports/$USER。

为了方便用户使用，管理员需要编写一点简单的文档指引用户。反过来，管理员也得给自己写一点简单的脚本来方便添加删除用户这样的操作。

# 方法概述
下面总结一下上面提到的方法：

使用 LXC 作为隔离机制，给每个用户分配一个 LXC 容器
给每个用户分配一个宿主机上的端口，利用 iptables 把这个端口转发到对应容器的22端口。用户使用 ssh 连接这个端口即可直接进入 LXC 容器。
只要有了 ssh，就可以控制远程电脑、传输文件、转发X11图形界面、使用 sshfs 把远程文件系统挂载到本地


在宿主机上安装 NVIDIA 显卡驱动，把显卡设备文件挂载到 LXC 容器中
在宿主机上挂载 NAS，再把挂载好的路径挂载到 LXC 容器中
编写一个脚本作为宿主机上用户 Shell，这个脚本应该要能做启动、关闭 LXC 容器等操作
为用户编写简单的使用说明
为管理员编写用于添加、删除用户的脚本
宿主机的预先配置
宿主机首先需要装好 NVIDIA 显卡驱动，使用 ls /dev/nvidia* 查看相关的设备文件，使用 nvidia-smi 命令确保显卡驱动正常运行。有趣的是，/dev/nvidia-uvm 这个设备文件并不会自己创建。我们可以用 这里 提供的脚本来解决这个问题。把这个脚本设置成开机自启动（比如说简单粗暴地加在 /etc/rc.local 里面）。

宿主机的 NAS 挂载也要配置好。我们的 NAS 支持 NFS 协议，所以直接在 /etc/fstab 里面添加一些内容就好了。

在后面的用户 Shell 脚本中需要用到 sudo，我们不希望让用户再次输入密码，所以我们在 sudoer 里面设置成不需要使用密码。

```
host$ sudo vim /etc/rc.local     # for the /dev/nvidia-uvm script
host$ sudo vim /etc/fstab
172.16.2.30:/mnt/NAS/Share /NAS/Share nfs rw 0 0
host$ sudo mount -a
host$ sudo service lightdm stop  # if Ubuntu Desktop
host$ sudo sh /NAS/Share/GPU_Server/NVIDIA-Linux-x86_64-375.20.run
host$ ls /dev/nvidia*
/dev/nvidia0  /dev/nvidiactl  /dev/nvidia-modeset  /dev/nvidia-uvm
host$ nvidia-smi                 # should have no error
host$ sudo visudo
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

# 制作 LXC 容器模板
为了方便添加用户，我们先制作一个 LXC 容器模板，之后每次新建容器的时候就从这个模板克隆一份。

首先管理员在宿主机上使用自己的普通权限账号新建一个 LXC 容器，这里可以跟着 LXC官方的文档 进行操作。其中，对于中国用户来说，可以使用清华的镜像来加速镜像的下载。在这里，我把容器的名字就叫做 template，后面添加用户的脚本有时会对 template 这个名字进行替换（比如替换 /etc/hosts 之类的）。

在启动容器之前，我们需要先修改容器的配置文件，把 NVIDIA 显卡设备文件挂载进去，另外因为我们显卡驱动的安装程序放在了 NAS 上，所以顺带也把 NAS 挂上。

然后我们启动容器，再使用 lxc-attch 进入容器，安装一些额外的软件（比如说 openssh-server），以及做一些额外的配置（比如说把软件源换成校内源）。

接着我们需要在容器中安装好显卡驱动。当然，这一步完全可以留给用户做，然而因为宿主机和容器内的显卡驱动要求完全一致，所以我们索性替用户做了，省得出现问题。在安装驱动的时候会提示无法卸载内核模块，这是正常的，毕竟容器和宿主机是共享内核的。实际上，我们在容器内也不需要安装内核模块，只是需要那些库罢了，所以在安装显卡驱动的时候带上 --no-kernel-module 就可以解决这个问题。

配置好了模板容器之后，我们关闭这个容器，并把它复制到 /root/lxc-public-images/template，并且稍微修改其中的配置文件，把 lxc.network.hwaddr, lxc.id_map, lxc.rootfs, lxc.utsname 等容器特有的配置删去。这些配置我们在后面的添加用户的脚本中再把它们生成出来。

```
host$ sudo apt install lxc
host$ sudo vim /etc/lxc/lxc-usernet
host$ lxc-create -t download -n template -- --server mirrors.tuna.tsinghua.edu.cn/lxc-images
host$ vim ~/.local/share/lxc/template/config
lxc.mount.entry = /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry = /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry = /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry = /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry = /NAS/Share NAS/Share none bind,create=dir

host$ lxc-start -d -n template
host$ lxc-attach -n template

container# vim /etc/apt/sources.list
container# apt update && apt install -y openssh-server
container# sh /NVIDIA-Linux-x86_64-375.20.run --no-kernel-module
container# nvidia-smi
container# exit

host$ lxc-stop -n template
host$ sudo mkdir -p /root/lxc-public-images/
host$ sudo cp -r ~/.local/share/lxc/template /root/lxc-public-images/template
host$ sudo vim /root/lxc-public-images/template/config
# delete: lxc.network.hwaddr, lxc.id_map, lxc.rootfs, lxc.utsname
```

# 编写各种脚本

我把所有需要用到的脚本都放在了 https://gist.github.com/abcdabcd987/d9ab8a8a36272678567e9fb23aed475b。 这是用于我们实验室的脚本，如果你需要借鉴，请勿直接复制粘贴，请确保自己明白每一行命令的作用，然后做修改。

# 编写用户文档
这里 有一份我们实验室里的 GPU Server 使用指南可以借鉴。

# 结语

这一套流程完整地做下来确实非常折腾。不过有了这么一套简单的管理方法之后，起码大家就不用因为软件冲突而大大降低工作效率。虽然只提供了 ssh，但只要有了 ssh，就可以控制远程电脑、传输文件、转发X11图形界面、使用 sshfs 把远程文件系统挂载到本地。这基本就能满足一般用户的所有需求了。我之前也写过一篇博客来介绍 SSH基本用法，有需要的可以看看。

至于后续的升级和维护嘛，这就是一个更大的坑了。我只能希望 Cuda 在推出新版本的同时，不要大幅度提高所需的显卡驱动版本，因为一旦这样的事情发生，就必须更新显卡驱动，而更新显卡驱动则需要更新宿主机上的驱动以及所有 LXC 容器的驱动。

另一方面，宿主机最好是能不更新就不更新了。做一次 dist-upgrade 基本上就是得从头来一次。而更新内核也是很危险的事情，要重新在宿主机上装一遍显卡驱动不要紧，怕的是更新之后和 LXC 出现了一些兼容性问题，导致 LXC 容器无法启动。

最后发一张用户登入 LXC 容器的图片。






# Reference
[Setting up CUDA in Linux containers - SQream](http://sqream.com/setting-cuda-linux-containers-2/)
[首发于博客：为实验室建立公用GPU服务器](https://abcdabcd987.com/setup-shared-gpu-server-for-labs/)
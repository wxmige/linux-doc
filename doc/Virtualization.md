# Virtualization

## kvm

kvm是linux内核提供的guest，内核通过字符设备/dev/kvm暴露操作接口，主要使用ioctl操作实现guest的创建和资源分配。

### lkvm

lkvm是一个简单轻量的kvm操作应用，它的底层是直接使用C语言操作/dev/kvm实现的。

#### 内核启动

使用lkvm启动guest所需要最基础的参数就是指定一个内核

```bash
./lkvm run -k bzImage
```

在没有指定文件系统的情况下，内核启动后会进入到lkvm自带的一个简单的busybox中，这是开发人员为了测试方便在lkvm中内置的，kvm本身没有自带busybox。

这个默认的文件系统在host中也可以找到

```bash
    # ls ~/.lkvm/default 
bin  dev  etc  home  host  lib  lib64  proc  root  sbin  sys  tmp  usr  var  virt
```

#### 指定根文件系统

要支持使用外挂的根文件系统需要guest支持CONFIG_VIRTIO_BLK。

加上--disk参数即可为kvm指定一个文件系统。

```bash
./lkvm run -k bzImage --disk rootfs.ext4
```

#### 挂载共享目录

要支持共享目录挂载，guest中必须要支持CONFIG_NET_9P_VIRTIO。

除了指定根文件系统之外，lkvm还支持host和kvm之间使用9p协议共享目录。

```bash
./lkvm run -k bzImage --disk rootfs.ext4 --9p testdir,tagmount
```

启动kvm后在kvm中使用mount命令挂载

```bash
mount -t 9p tagmount /mnt
```

其中“testdir“是host要共享目录的路径，“tagmount“是一个tag名字可以随意取，“/mnt”是在kvm中需要挂载的共享文件路径。

#### 桥接网络

要实现桥接网络，如果要使用vhost-net加速在host中必须要支持 CONFIG_VHOST_NET，在guest中必须要支持CONFIG_VIRTIO_NET。

首先要将host中的网络改为桥接模式

```bash
ip link add name br0 type bridge
ip link set br0 up
ip link set eth0 down
ip link set eth0 master br0
ip link set eth0 up
dhclient br0
```

在这种桥接模式下默认路由需要配置给br0，如果在做dhclient br0之前eth0没有被down，那么eth0会占用默认路由，br0无法获取到默认路由导致发送给网关的数据出问题。

如果发现默认路由在eth0上也可以手动删除并手动给br0配置默认路由

```bash
route del default dev eth0
ip route add default via 192.168.1.1 dev br0
```

如果开启了网络管理服务需要关掉

```bash
systemctl stop NetworkManager
```

在host中创建一个tap网卡给guest使用

```bash
ip tuntap add tap0 mode tap
ip link set tap0 master br0
ip link set tap0 up
```

启动一个guest使用vhost-net指定桥接模式

```bash
./lkvm run -k bzImage --disk rootfs.ext4 --9p testdir,tagmount -n mode=tap,tapif=tap0,vhost=1
```

如果不想使用vhost-net

```bash
./lkvm run -k bzImage --disk rootfs.ext4 --9p testdir,tagmount -n mode=tap,tapif=tap0
```

在guest中使用dhcp配置ip即可正常使用网络

```bash
dhclient eth0
```

此外在有docker服务的系统中，docker会使用iptable配置一定的规则，也会导致guest的桥接失败，可以在docker启动配置文件中禁止其添加规则，重启服务生效。

```bash
    # vim /etc/docker/daemon.json
{
    "iptables": false
}
```

vhost-net是一个专门为kvm和tap桥接服务的模块。

在不使用vhost-net模块时，guest的用户态使用socket将数据发送给guest的内核态的virtio-net驱动，virtio-net驱动获取数据之后会调用read、write将数据通过tap传递给host的内核态，host的内核态网络协议栈再发送给网桥，最终发送给eth0这个真实的物理网卡实现对外通信。

使用vhost-net模块时，guest的用户态使用socket将数据发送给guest的内核态的virtio-net驱动，virtio-net驱动获取数据之后会使用vhost-net实现tap网卡用户态到内核态的数据0拷贝通信，host的内核态网络协议栈再发送给网桥，最终发送给eth0这个真实的物理网卡实现对外通信。

#### PCIE直通

要实现PCIE直通，在guest中必须要支持CONFIG_VFIO_PCI。

使用lspci命令找到要直通的pci设备的 域:总线:设备.功能号，例如0000:01:00.0

查看当前设备是否已经被host的驱动绑定

```bash
cd /sys/bus/pci/devices/0000:01:00.0
ls driver
```

如果当前目录下已经存在driver目录，说明该设备已经被host上的某个驱动绑定，这里需要先将其解绑

```bash
echo 0000:01:00.0 > driver/unbind
```

此时0000:01:00.0设备目录下的driver目录也随即消失，接下来需要将0000:01:00.0这个设备绑定到vfio驱动上，此时需要host上有vfio驱动，在大多数发行版中该驱动不会默认加载，需要手动加载一下模块。

```bash
  # modprobe vfio-pci
  # lsmod |grep vfio
vfio_pci               16384  0
vfio_pci_core          86016  1 vfio_pci
vfio_iommu_type1       45056  0
vfio                   61440  3 vfio_pci_core,vfio_iommu_type1,vfio_pci
```

这时需要将0000:01:00.0设备绑定到vfio驱动上去。

```bash
  # lspci -s 0000:01:00.0 -n  
0000:01:00.0 0580: 10ee:9034
  # cd /sys/bus/pci/drivers/vfio-pci
  # echo "10ee 9034" > new_id
  # echo 0000:01:00.0 > bind
```

绑定成功后有两个标志:

1是/sys/bus/pci/devices/0000:01:00.0目录下又出现了driver目录，此时driver指向的就是vfio的驱动

```bash
  # ls -al /sys/bus/pci/devices/0000:01:00.0/driver
lrwxrwxrwx 1 root root 0 10月30日 14:32 driver -> ../../../../bus/pci/drivers/vfio-pci
```

2是/dev/vfio目录下会出现一个表示iommu组编号的文件

```bash
  # ls /dev/vfio
14  vfio
```

再就能在使用lkvm时使用-vfio-pci指的pci设备号完成PCIE直通了

```bash
./lkvm run -k bzImage --disk rootfs.ext4 --9p testdir,tagmount -n mode=tap,tapif=tap0,vhost=1 --vfio-pci 0000:01:00.0
```

在guest中通过lspci就能看见真实的pcie设备

```bash
    # lspci 
00:00.0 Memory controller: Xilinx Corporation Device 9034
```

### qemu

qemu能真正模拟不同的硬件，它支持模拟与host架构硬件同时支持BootLoader启动。直接使用kvm只能通过virtio驱动来模拟硬件，并且无法实现BootLoader支持。

#### 内核启动

```bash
qemu-system-x86_64 \
-m 2G -smp 2 -machine accel=kvm -cpu host --nographic \
-kernel bzImage \
```

使用 -m 和 -smp 可以指定 guest 的内存大小和使用的核心数。-machine accel=kvm -cpu host即使用 kvm 加速，让模拟 CPU 的核心模块使用kvm实现而无需另外模拟硬件。--nographic 表示不使用图形输出。-kernel bzImage指定运行内核的路径。

在X86上也可以模拟arm64架构的运行运行，但不能使用kvm加速，由于arm64架构的开放性，不同的厂家的芯片差异较大，需要用-M和-cpu参数指定。

```bash
qemu-system-aarch64 -M virt -cpu cortex-a53 -m 2G --nographic \
-kernel Image 
```

#### 指定根文件系统

qemu并没有默认的文件系统，需要手动指定。

```bash
qemu-system-x86_64 \
-m 2G -smp 2 -machine accel=kvm -cpu host --nographic \
-kernel bzImage \
-device virtio-blk-pci,drive=disk0 -drive file=rootfs.ext4,format=raw,id=disk0,if=none \
-append "earlycon=pl011,mmio32,0x9000000 console=ttyS0,115200 root=/dev/vda"
```

与直接使用kvm不同，qemu不会默认使用virtio-blk硬件，而是会自己模拟一个硬盘硬件，可以手动指定使用virtio-blk-pci挂载在disk0上用于硬盘访问加速。

-append 可以指定启动参数，将earlycon打开可以查看启动过程中的内核打印，方便定位启动失败问题，earlycon=pl011,mmio32,0x9000000是qemu默认模拟的串口。指定root=/dev/vda可以指定根文件系统为传入的rootfs.ext4。

#### 网络

##### nat

```bash
qemu-system-x86_64 \
-m 2G -smp 2 -machine accel=kvm -cpu host --nographic \
-kernel bzImage \
-device virtio-blk-pci, drive=disk0 -drive file=rootfs.ext4,format=raw,id=disk0,if=none \
-device virtio-net-pci,netdev=net0 -netdev user,hostfwd=tcp::8080-:22 \
-append "earlycon=pl011,mmio32,0x9000000 console=ttyS0,115200 root=/dev/vda"
```

同样使用virtio-net-pci设备可以对网卡进行加速，-netdev user表示使用nat模式，hostfwd=tcp::8080-:22表示端口映射，即访问host的8080端口即可访问guest的22端口，即通过访问host的8080端口可以使用ssh访问guest。

```bash
ssh 127.0.0.1:8080
```

##### 桥接

```bash
qemu-system-x86_64 \
-m 2G -smp 2 -machine accel=kvm -cpu host --nographic \
-kernel bzImage \
-device virtio-blk-pci, drive=disk0 -drive file=rootfs.ext4,format=raw,id=disk0,if=none \
-device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0 \
-append "earlycon=pl011,mmio32,0x9000000 console=ttyS0,115200 root=/dev/vda"
```

指定-netdev tap，id=net0指的是guest中该网卡的名字，ifname=tap0指的是host中的tap网卡名，在host中配置tap桥接的过程这里就不重复了。

若要使用vhost加速则加上vhost=on

```bash
-device virtio-net-pci,netdev=net0 -netdev tap,id=net0,ifname=tap0,vhost=on \
```

#### gdb调试内核

```bash
qemu-system-x86_64 \
-m 2G -smp 2 --nographic \
-kernel bzImage \
-device virtio-blk-pci,drive=disk0 -drive file=rootfs.ext4,format=raw,id=disk0,if=none \
-append "earlycon=pl011,mmio32,0x9000000 console=ttyS0,115200 root=/dev/vda nokaslr" -S -s
```

在使用KVM时，可能会存在gdb无法访问到的内存，因此在调试时为了方便建议去掉kvm的使用。此外在x86以及armv8以下的版本中存在内核地址空间布局随机化(KASLR)功能，它会导致gdb无法正确找到符号的正确地址，因此需要在启动参数中添加nokaslr来关闭此功能。

bzImage是没有带符号表的，因此在调试内核时需要使用vmlinux，在编译内核时需要注意打开DEBUG_INFO_DWARF4选项，vmlinux才支持调试。

```bash
	$ file vmlinux
vmlinux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=420c91c5390c0a455a8388f1439e6fe54ce57b6e, with debug_info, not stripped
```

vmlinux中需要带with debug_info, not stripped这两个标识。

在qemu中带上-S -s命令，其中-S表示等待gdb发送c指令才运行，-s表示指定远程调试的端口默认为1234。

```bash
	$ gdb
(gdb) symbol-file vmlinux
Reading symbols from vmlinux...
(gdb) target remote 127.0.0.1:1234
Remote debugging using 127.0.0.1:1234
0x000000000000fff0 in exception_stacks ()
(gdb) b start_kernel
Breakpoint 1 at 0xffffffff82dd48c0: file init/main.c, line 904.
(gdb) c
Continuing.

Thread 1 hit Breakpoint 1, start_kernel () at init/main.c:904
904     {
(gdb) 
```

#### gdb调试内核模块

先正常启动一个内核

```bash
qemu-system-x86_64 \
-m 2G -smp 2 --nographic \
-kernel bzImage \
-device virtio-blk-pci,drive=disk0 -drive file=rootfs.ext4,format=raw,id=disk0,if=none \
-append "earlycon=pl011,mmio32,0x9000000 console=ttyS0,115200 root=/dev/vda nokaslr" -S -s
```

在host中gdb直接输入c继续运行

```bash
	$ gdb
(gdb) symbol-file vmlinux
Reading symbols from vmlinux...
(gdb) target remote 127.0.0.1:1234
Remote debugging using 127.0.0.1:1234
0x000000000000fff0 in exception_stacks ()
(gdb) c
Continuing.
```

在guest中

```bash
	$ insmod test.ko
	$ cat /proc/modules 
test 12288 0 - Live 0xffffffffc0000000 (O)
```

在host中gdb使用ctrl+c打断c指令状态，并导入模块加内核加载地址，并打断点，输入c继续

```bash
(gdb) c
Continuing.
^C
(gdb) add-symbol-file ../kvm/vm/work/ko/test.ko 0xffffffffc0000000
(gdb) b sema_init
Breakpoint 2 at 0xffffffff815c353a: sema_init. (9 locations)
(gdb) c
Continuing.
```

在guest中，运行用户态测试代码

```bash
./test
```

在host中gdb看到打断

```bash
(gdb) c
Continuing.
[Switching to Thread 1.1]

Thread 1 hit Breakpoint 2.7, vxdma_mod_exit () at test.c:139
139             file->private_data = vc;
(gdb)
```

## Docker

### Docker 的使用

---

安装 docker

```bash
apt install docker.io
```

---

启动 docker 服务

```bash
systemctl start docker
```

---

网络下载一个 ubuntu 镜像

```bash
docker pull ubuntu
```

---

下载完毕后查看此镜像 

```bash
docker images
```

---

使用 ubuntu 镜像 (Image) 运行一个名为 mir-ubuntu 的容器 (Container)

```bash
docker run --privileged -itd --name mir-ubuntu ubuntu
```

其中–privileged 表示使用特权级别运行,加上后该容器获取能访问 host 中 root 才能访问的相关资源。

---

查看正在运行的容器 (不包括已经停止的)

```bash
docker ps
```

---

在当前容器中执行命令

```bash
docker exec -it mir-ubuntu ls
```

---

运行/bin/bash 程序进入命令交互状态

```bash
docker exec -it mir-ubuntu /bin/bash
```

---

拷贝容器中的文件到 host

```bash
docker cp mir-ubuntu:/test .
```

---

拷贝 host 的文件到容器中

```bash
docker cp ./test mir-ubuntu:/
```

---

保存当前对容器的修改为一个镜像

```bash
docker commit mir-ubuntu mir-ubuntu-base
```

---

保存后就可以通过 docker images 看到这个镜像，使用 docker run 指定这个镜像就能以 mir-ubuntu-base 为基础启动新的容器。

保存的镜像可以进行打包

```bash
docker save -o mir-ubuntu-base.tar mir-ubuntu-base
```

---

其他机器在拿到 mir-ubuntu-base.tar 可以进行加载

```bash
docker load -i mir-ubuntu-base.tar
```

---

停止当前容器的运行

```bash
docker stop mir-ubuntu
```

---

查看所有容器运行状态 (包括已经停止的)

```bash
docker ps -a
```

---

恢复停止运行的容器

```bash
docker start mir-ubuntu
```

---

删除容器，此命令运行后未保存的容器中的修改将会丢失，正在运行的容器是无法删除的，需要先停止其运行

```bash
docker rm mir-ubuntu
```

---

停止所有正在运行的容器 

```bash
docker stop `docker ps -aq`
```

---

如果不再需要 ubuntu 这个镜像了可以将其删除

```bash
docker rmi ubuntu
```

---

docker 分为 docker 镜像 (docker images) 和 docker 容器 (docker container) 。

镜像使用 docker pull 从网络下载或使用 docker load 从本地加载，使用 docker images 查看已经加载的镜像，用 docker run 生成新的容器，用 docker rmi 删除镜像，镜像删除前必需先删除所有以它为基础镜像生成的容器。

容器用 docker ps -a 查看，用 docker start 和 docker stop 控制，用 docker rm 进行删除，容器删除前要先执行 stop。 除了基础命令之外 docker 还提供了一下强大的自动化清理操作。

---

删除所有未使用的镜像、容器、网络和缓存

```bash
docker system prune
```

---

删除所有未使用的镜像、容器、网络和缓存、已经停止的容器

```bash
docker system prune -a
```

---

删除所有停止的容器

```bash
docker container prune
```

---

删除所有未使用的镜像 (有停止的容器的镜像不会被删除)

```bash
docker image prune -a
```

## virt-net

### Linux Bridge 桥接

安装Linux Bridge工具

```bash
apt install bridge-utils
```

使用Linux Bridge创建网桥，将eth0与tap0桥接

```bash
ip link add name br0 type bridge
ip link set br0 up
ip link set eth0 down
ip link set eth0 master br0
ip link set eth0 up
dhclient br0
ip tuntap add tap0 mode tap
ip link set tap0 master br0
ip link set tap0 up
```

删除网桥

```
ip link set tap0 down
ip tuntap del tap0 mode tap
brctl delbr br0
```

### Open vSwitch 桥接

安装Open vSwitch(OVS)工具。

```bash
sudo apt install openvswitch-switch
```

OVS有一个后台服务，需要确保其运行起来。

```bash
systemctl start openvswitch-switch.service
```

使用ovs创建网桥，将eth0与tap0桥接

```bash
ovs-vsctl add-br br0
ip link set br0 up
ip link set eth0 down
ovs-vsctl add-port br0 eth0
ip link set eth0 up
dhclient br0
ip tuntap add tap0 mode tap
ovs-vsctl add-port br0 tap0
ip link set tap0 up
```

删除网桥

```bash
ovs-vsctl del-port br0 eth0
ovs-vsctl del-port br0 tap0
ovs-vsctl del-br br0
systemctl stop openvswitch-switch
```


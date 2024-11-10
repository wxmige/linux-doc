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

要实现桥接网络，在host中必须要支持 CONFIG_VHOST_NET，在guest中必须要支持CONFIG_VIRTIO_NET。

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

在host中创建一个tap网卡给guest使用

```bash
ip tuntap add tap0 mode tap
ip link set tap0 master br0
ip link set tap0 up
```

启动一个guest使用vhost指定桥接模式

```bash
./lkvm run -k bzImage --disk rootfs.ext4 --9p testdir,tagmount -n mode=tap,tapif=tap0,vhost=1
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
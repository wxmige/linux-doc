# UIO / VFIO

## UIO

UIO(userspace I/O)是提供给用户空间的 I/O 系统。对于许多类型的设备来说，创建 Linux 内核驱动程序是多余的。

要使驱动逻辑在用户态实现，一般来说需要获取到驱动设备要访问的内存地址空间，尤其是 IOVA 地址，在Linux中它无法在用户态直接申请。另外就是中断，必需是用Linux内核注册，在触发后通知给用户态。因此 UIO 驱动要解决的就是两点，内存和中断。

### MEM

#### BAR

在 UIO 的模型中，并没有直接获取 IOVA 地址的接口，因此 UIO 必需在未开启 IOMMU 或 IOMMU=pt的模式下使用。

在 Linux 内核中 UIO 是一个核心内核模块，它需要另一个设备驱动模块配合才会注册出用户可用的字符设备，在内核中有常见的设备驱动模块，例如 PCI 设备可以使用uio_pci_generic.c，platform 设备可以使用 uio_pdrv_genirq.c。

对于 PCI 设备来说还有另一个经典的模块就是 igb_uio(git://dpdk.org/dpdk-kmods)，它不仅用于igb网卡，而是一个 PCI 设备通用的 UIO 设备驱动模块，相比于uio_pci_generic.c，其功能更全面，因此常见的 PCI 设备驱动都使用 UIO + igb_uio这两个模块实现。

在加载 UIO + igb_uio 两个模块之后，需要将 PCI 设备与 igb_uio进行手动绑定，绑定后 dev 目录下就会注册出/dev/uio0设备。如果设备已经被其他驱动绑定，则需要先解绑。

```bash
    # echo "10ee 9034" > /sys/bus/pci/drivers/igb_uio/new_id
    # echo 0000:01:00.0 > /sys/bus/pci/drivers/igb_uio/bind
    # ls /dev/uio0
/dev/uio0
```

在使用时uio每次向后映射一个pagesize，失败了则是这个bar设备没有定义。

```c
    #define UIO_DEV_PATH "/dev/uio0"
    int fd = open(UIO_DEV_PATH, O_RDWR);
    if (fd < 0) {
        perror("Failed to open UIO device");
        return -1;
    }

    for (int i = 0; i < MAX_BARS; i++) {
        void *bar_addr = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED, fd, i * getpagesize());
        if (bar_addr == MAP_FAILED) {
            perror("Failed to map BAR");
            continue;
        }
    }
```

#### DMA

在获取 DMA 地址时，需要先通过mmap得到一个虚拟地址 VA，再通过/proc/self/pagemap找到其物理地址 PA，由于是在为开启 IOMMU 或 IOMMU=pt 模式下，因此直接将PA提供给 DMA 使用即可。下面是通过 /proc/self/pagemap 获取 PA 的方法:

```c
unsigned long get_physical_address(void *vaddr) {
    int pagemap_fd = open("/proc/self/pagemap", O_RDONLY);
    if (pagemap_fd < 0) {
        perror("Failed to open /proc/self/pagemap");
        return 0;
    }

    uint64_t paddr = 0;
    uint64_t index = (uint64_t)vaddr / PAGE_SIZE * sizeof(uint64_t);
    if (lseek(pagemap_fd, index, SEEK_SET) == (off_t)-1) {
        perror("Failed to seek in /proc/self/pagemap");
        close(pagemap_fd);
        return 0;
    }

    if (read(pagemap_fd, &paddr, sizeof(paddr)) != sizeof(paddr)) {
        perror("Failed to read /proc/self/pagemap");
        close(pagemap_fd);
        return 0;
    }

    close(pagemap_fd);

    // Check if the page is present
    if (!(paddr & (1ULL << 63))) {
        fprintf(stderr, "Page not present\n");
        return 0;
    }

    // Extract the physical frame number (PFN)
    paddr = (paddr & ((1ULL << 55) - 1)) * PAGE_SIZE;
    paddr |= (uint64_t)vaddr & (PAGE_SIZE - 1);

    return (unsigned long)paddr;
}
```

### IRQ

在 PCI 设备中可能会有 INTX  MSI MSIX 这几种中断，在 config 空间中可以读取设备有哪些中断，通过 UIO 驱动选择使用的中断的类型，报给用户态的时候就只有一个中断信号了，即用户态不需要再对中断类型做区分。

Comment在 UIO 中，选择中断类型是在 PCI 设备绑定 igb_uio 驱动的时候，默认使用的是 msix 中断，如果选择的中断模式该设备不支持，则会按照 msix->msi->legacy 的顺序选择中断。

```c
module_param(intr_mode, charp, S_IRUGO);
MODULE_PARM_DESC(intr_mode,
"igb_uio interrupt mode (default=msix):\n"
"    " RTE_INTR_MODE_MSIX_NAME "       Use MSIX interrupt\n"
"    " RTE_INTR_MODE_MSI_NAME "        Use MSI interrupt\n"
"    " RTE_INTR_MODE_LEGACY_NAME "     Use Legacy interrupt\n"
"\n");
```

在 igb_uio 模块注册成功后，就会注册 /dev/uio0 设备，对其调用 mmap 获取 pci 设备对应的bar空间，对其调用 read 即可获取 pci 设备的中断。

```c
    unsigned int event_count = 0；
    read(fd, &event_count, sizeof(event_count));
```

如果有多个设备可以使用poll，在 igb_uio 中一个设备只支持一个中断。

```c
    struct pollfd fds;
    fds.fd = fd;//open "/dev/uio0"
    fds.events = POLLIN;

    unsigned int irq_count = 0;

    while (1) {
        int ret = poll(&fds, 1, -1);
        if (ret < 0) {
            perror("Poll failed");
            break;
        }
        if (fds.revents & POLLIN) {
            if (read(fd, &irq_count, sizeof(irq_count)) != sizeof(irq_count)) {
                perror("Failed to read IRQ count");
                break;
            }
        }
    }
```

## VFIO

VFIO(Virtual Function I/O)是一个比 UIO 功能更强大、安全性更高的用户态设备访问框架，VFIO 必需依赖于 IOMMU 硬件。

使用 PCI 设备和 platform 设备时也需要不同的模块的支持，PCI 设备使用vfio_pci驱动，platform设备使用vfio_platform(一般在arm上才会使用)。

```bash
    # modprobe vfio-pci
    # lsmod |grep vfio
vfio_pci               16384  0
vfio_pci_core          94208  1 vfio_pci
vfio_iommu_type1       45056  0
vfio                   61440  3 vfio_pci_core,vfio_iommu_type1,vfio_pci
```

如果设备已经被其他驱动绑定，则需要先解绑。

```bash
    # echo "10ee 9034" > /sys/bus/pci/drivers/vfio-pci/new_id
    # echo 0000:01:00.0 > /sys/bus/pci/drivers/vfio-pci/bind
    # ls /dev/vfio/14
/dev/vfio/14
```

### MEM

#### BAR

在使用时VFIO设备时，首先通过 VFIO_DEVICE_GET_REGION_INFO获取每个 BAR 的信息 (bar_info)，从bar_info中可以得知某个 BAR 是否存在以及它的大小，只有在确认 BAR 存在且大小大于 0 的情况下，才会尝试去映射 BAR 地址空间。

```c
    #define VFIO_DEV_PATH      "/dev/vfio/vfio"
    #define VFIO_DEVICE        "/dev/vfio/14"

    // Open the VFIO container
    int container = open(VFIO_DEV_PATH, O_RDWR);
    if (container < 0) {
        perror("Failed to open VFIO container");
        return -1;
    }

    // Set IOMMU type
    if (ioctl(container, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU) < 0) {
        perror("Failed to set IOMMU type");
        close(container);
        return -1;
    }

    // Open the VFIO group
    int group = open(VFIO_DEVICE, O_RDWR);
    if (group < 0) {
        perror("Failed to open VFIO group");
        close(container);
        return -1;
    }

    // Set group to the container
    if (ioctl(group, VFIO_GROUP_SET_CONTAINER, &container) < 0) {
        perror("Failed to set group to container");
        close(group);
        close(container);
        return -1;
    }

    // Get a device FD
    int device_fd = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:01:00.0");
    if (device_fd < 0) {
        perror("Failed to get device FD");
        close(group);
        close(container);
        return -1;
    }

    // Iterate over all possible BARs
    for (int i = 0; i < MAX_BARS; i++) {
        struct vfio_region_info bar_info = { .argsz = sizeof(bar_info), .index = i };
        if (ioctl(device_fd, VFIO_DEVICE_GET_REGION_INFO, &bar_info) < 0) {
            perror("Failed to get BAR info");
            continue;
        }

        // Check if the BAR is present and has size greater than zero
        if (bar_info.size > 0) {
            void *bar_addr = mmap(NULL, bar_info.size, PROT_READ | PROT_WRITE, MAP_SHARED, device_fd, bar_info.offset);
            if (bar_addr == MAP_FAILED) {
                perror("Failed to map BAR");
                continue;
            }
        }
    }
```

#### DMA

在VFIO中iova地址通过ioctl VFIO_IOMMU_MAP_DMA来完成，关键入参是dma_map.vaddr和dma_map.iova。（DPDK中对应的函数为vfio_type1_dma_mem_map）

```c
    #define VFIO_DEV_PATH      "/dev/vfio/vfio"
    // Open the VFIO container
    int vfio_container_fd = open(VFIO_DEV_PATH, O_RDWR);
    if (vfio_container_fd < 0) {
        perror("Failed to open VFIO container");
        return -1;
    }

    // Set IOMMU type
    if (ioctl(vfio_container_fd, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU) < 0) {
        perror("Failed to set IOMMU type");
        close(vfio_container_fd);
        return -1;
    }

	struct vfio_iommu_type1_dma_map dma_map;
	memset(&dma_map, 0, sizeof(dma_map));
	dma_map.argsz = sizeof(struct vfio_iommu_type1_dma_map);
	dma_map.vaddr = vaddr;
	dma_map.size = len;
	dma_map.iova = iova;
	dma_map.flags = VFIO_DMA_MAP_FLAG_READ |
			VFIO_DMA_MAP_FLAG_WRITE;

    ioctl(vfio_container_fd, VFIO_IOMMU_MAP_DMA, &dma_map);
```

其中的dma_map.vaddr指的是mmap申请出来的虚拟地址。

### IRQ

VFIO可以支持多个中断通过eventfd触发，以下是一个用例:

```c
#define VFIO_DEV_PATH      "/dev/vfio/vfio"
#define VFIO_DEVICE        "/dev/vfio/14"

// Open the VFIO group
int group = open(VFIO_DEVICE, O_RDWR);
if (group < 0) {
    perror("Failed to open VFIO group");
    close(container);
    return -1;
}

// Set group to the container
if (ioctl(group, VFIO_GROUP_SET_CONTAINER, &container) < 0) {
    perror("Failed to set group to container");
    close(group);
    close(container);
    return -1;
}

// Get a device FD
int device_fd = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:01:00.0");
if (device_fd < 0) {
    perror("Failed to get device FD");
    close(group);
    close(container);
    return -1;
}

int efd1 = eventfd(0, 0);
int efd2 = eventfd(0, 0);

struct vfio_irq_set *irq_set = malloc(irq_set_size);

irq_set->flags = VFIO_IRQ_SET_DATA_EVENTFD | VFIO_IRQ_SET_ACTION_TRIGGER;
irq_set->index = VFIO_PCI_MSIX_IRQ_INDEX;  // Replace with correct IRQ index, such as INTX or MSI-X
irq_set->start = 0;
irq_set->count = 2;//假设这个设备有两个中断
int *fds = (int *)irq_set->data;
fds[0] = efd1;
fds[1] = efd2;

if (ioctl(device_fd, VFIO_DEVICE_SET_IRQS, irq_set) < 0) {
	perror("Failed to set IRQs");
	free(irq_set);
	return -1;
}
```

接下来只需要监听绑定的fd即可触发不同的中断。
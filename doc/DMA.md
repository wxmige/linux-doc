# DMA

DMA是一种让外设能够直接访问CPU主存，而不需要CPU主动参与数据拷贝，从而提升数据拷贝的性能的技术。

## DMA MAP

### IOMMU

先明确几个地址概念:

PA（Physical Address，物理地址）是指物理地址是硬件级别的真实地址，用于直接定位系统物理内存中的某个存储单元。 

VA（Virtual Address，虚拟地址）是指经过 MMU 映射后的地址，开启 MMU 后，CPU 只能通过 VA 访问 PA。没有在 MMU 中映射的 PA，CPU 将无法直接访问。

IOVA（I/O Virtual Address，I/O 虚拟地址）是指经过 IOMMU 映射后的地址，开启 IOMMU 后，外设只能通过 IOVA 访问 PA。没有在 IOMMU 中映射的PA，外设将无法访问。

在 IOMMU 开启的设备上，外设必需访问 IOVA 而非 PA，而在没有 IOMMU 或未开启 IOMMU 的设备上，DMA才可以通过 PA 访问 CPU 上的主存。

此外IOMMU有一种 pt (Passthrough) 模式，在内核启动时使用 iommu=pt即可开启，开启后物理地址以后外设也可以直接通过 PA 访问 CPU 的主存。

DMA MAP 要做的事情，就是获取IOVA的过程，以下DMA MAP API所返回的地址都是可供DMA直接使用的地址，因此不需要在编程时考虑 IOMMU 是否开启，开启时 MAP 返回的就是 IOVA，不开启则是返回PA，因此直接使用 API 返回的地址用于 DMA 即可 。

### DMA MAP 类型

DMA 的类型主要分为 DMA 一致性映射 (DMA Coherent Mapping) 和 DMA 非一致性映射 (DMA Streaming Mapping)。

在做映射前，需要对 DMA 做掩码设置，该设置是为了决定生成的 IOVA 地址是在哪一段空间，例如将 DMA 掩码设置为64时 DMA MAP 返回的 IOVA 地址将是一个64位地址。

将一致性和非一致性 DMA 的掩码设置都设置为 64 位。

```c
dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
```

单独将一致性 DMA 的掩码设置都设置为 64 位。

```c
dma_set_coherent_mask(dev, DMA_BIT_MASK(64));
```

单独将非一致性 DMA 的掩码设置都设置为 64 位。

```c
dma_set_mask(dev, DMA_BIT_MASK(64));
```

在Linux 5.16以下的版本中，设置掩码的函数不同，以下给出一个用例，作用是给一致性和非一致性DMA设置64位掩码，失败则设置32位掩码。

```c
#if LINUX_VERSION_CODE < KERNEL_VERSION(5, 16, 0)
        if (!pci_set_dma_mask(pdev, DMA_BIT_MASK(64)))
#else
        if (!dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64)))
#endif
        {
#if LINUX_VERSION_CODE < KERNEL_VERSION(5, 16, 0)
                if (!pci_set_consistent_dma_mask(pdev, DMA_BIT_MASK(64))))
#endif
                {
                        printk ("set 64 bit dma\n");
                }
        }
        else
        {
#if LINUX_VERSION_CODE < KERNEL_VERSION (5, 16, 0)
                if (!pci_set_dma_mask(pdev, DMA_BIT_MASK(32)))
#else
                if (!dma_set_mask_and_coherent(&pdev ->dev, DMA_BIT_MASK(32)))
#endif
                {
#if LINUX_VERSION_CODE < KERNEL_VERSION (5, 16, 0)
                        pci_set_consistent_dma_mask (pdev , DMA_BIT_MASK (32));
#endif
                        printk ("set 32 bit dma\n");
                }
                else
                        return -EINVAL ;
        }
```

#### 一致性DMA MAP

```c
dma_addr_t dma_handle ;
cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, gfp);
```

#### 非一致性 DMA MAP

非一致性 DMA MAP 提供了几种类型的 API，若已知虚拟地址连续，则可以使用 dma_map_single。

```c
# include <linux/dma-mapping.h>

dma_addr_t dma_handle;
void *addr = buffer->page;
size_t size = buffer->len;
dma_handle = dma_map_single(dev, addr, size, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_handle))
    dma_unmap_single(dev, dma_handle, size, DMA_TO_DEVICE);
```

要注意 dma_handle 这个 IOMMU 的地址是一次性的经过 DMA 传输后就会自动释放，需要重新进行 map。 但是在传输完成后仍需调用 dma_unmap_single 进行释放，后面的 API 也同样如此。

若已知数据 PAGE 结构，则可以使用 dma_map_page。

```c
dma_addr_t dma_handle ;
struct page *page = buffer->page;
unsigned long offset = buffer->offset;
size_t size = buffer->len;
dma_handle = dma_map_page(dev, page, offset, size, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_handle)) {
    dma_unmap_page(dev, dma_handle, size, DMA_TO_DEVICE);
}
```

若已知数据的多个 PAGE 结构，并且多个PAGE之间可能不连续，则可以使用 dma_map_sg。

```c
struct page **page = buffer->page;
int pages_nr = buffer->pages_nr;
struct sg_table sgt;
if (sg_alloc_table(&sgt, pages_nr, GFP_KERNEL))
    return -ENOMEM;
struct scatterlist *sglist = sgt.sgl;
for (i = 0; i < pages_nr; i++, sglist = sg_next(sglist))
{
    flush_dcache_page(pages[i]);
    sg_set_page(sglist, pages[i], nbytes, offset);
}
int count = dma_map_sg(dev, sglist, pages_nr, DMA_TO_DEVICE);
struct scatterlist *sg;
for_each_sg(sglist, sg, count, i) {
    hw_address[i] = sg_dma_address(sg);
    hw_len[i] = sg_dma_len(sg);
}
```

## DMA 描述符

DMA 描述符在不同设备上有不同的定义，具体要参考对应的手册，下面是一个比较通用的 DMA 描述符该有的内容:

| OFFSET | Fields  | BIT     |
| ------ | ------- | ------- |
| 0x00   | Control | [31:0]  |
| 0x04   | Len     | [31:0]  |
| 0x08   | Src_adr | [31:0]  |
| 0x0C   | Src_adr | [63:32] |
| 0x10   | Dst_adr | [31:0]  |
| 0x14   | Dst_adr | [63:32] |
| 0x18   | Nxt_adr | [31:0]  |
| 0x1C   | Nxt_adr | [63:32] |

在 DMA 描述符中填写的地址都是 DMA 控制器能访问到的物理地址。

Src_adr 和 Dst_adr 中分别填入 DMA 传输的源地址和目的地址。 

Nxt_adr 填写的是下一个描述符的地址。 

Len 填写的是要传输的数据的长度。 

Control 中填写的内容则相对不固定，需要根据具体手册来确定。
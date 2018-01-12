# USB Storage代码流程分析

## 0. 前言
&ensp;&ensp;&ensp;&ensp;当前已从linux kernel剥离出USB storage驱动代码，并做成源码树形式，可通过ctags或source insight等工具进行解读，路径在git-usb目录下，其中代码已添加必要注释，以下会按照函数流程做主要代码的解读。
## 1. 函数入口
### 1.1 注册函数分析
&ensp;&ensp;&ensp;&ensp;函数入口在文件drivers/usb/storage/usb.c中，最后一行代码：

```C
/* USB storage入口函数 */
module_usb_stor_driver(usb_storage_driver, usb_stor_host_template, DRV_NAME);
```

&ensp;&ensp;&ensp;&ensp;这是一个usb storage封装注册函数，其中参数**usb_storage_driver**即usb storage接口驱动struct usb_driver结构体实例，用于在usb核心层识别usb接口驱动，**usb_stor_host_template**为scsi_host_template结构体实例，为一个本地变量，用于与scsi层通信，定义如下：
```C
/* struct scsi_host_template结构体实例usb_stor_host_template */
static struct scsi_host_template usb_stor_host_template;
```

宏定义**DRV_NAME**如下：

```C
/* 驱动名称，后续很多地方用到 */
#define DRV_NAME "usb-storage"
```

解析该封装函数如下：

```C
#define module_usb_stor_driver(__driver, __sht, __name) \
static int __init __driver##_init(void) \
{ \ 
    usb_stor_host_template_init(&(__sht), __name, THIS_MODULE); \
    return usb_register(&(__driver)); \
} \ 
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ \ 
    usb_deregister(&(__driver)); \
} \
module_exit(__driver##_exit)
```

&ensp;&ensp;&ensp;&ensp;这其中介绍一下usb_stor_host_template_init()函数，其是初始化usb.c中定义的struct scsi_host_template实例，代码实现在drivers/usb/storage/scsiglue.c中：

```C
/* 初始化struct scsi_host_template结构体实例 */
void usb_stor_host_template_init(struct scsi_host_template *sht,
                 const char *name, struct module *owner)
{
    *sht = usb_stor_host_template;
    sht->name = name;
    sht->proc_name = name;
    sht->module = owner;
}
EXPORT_SYMBOL_GPL(usb_stor_host_template_init);
```
&ensp;&ensp;&ensp;&ensp;即将传入的参数sht与具体的实现usb_stor_host_template联结，结构体usb_stor_host_template在该段代码实现之上定义：

```C
/*
 * this defines our host template, with which we'll allocate hosts
 */
static const struct scsi_host_template usb_stor_host_template = {
    /* basic userland interface stuff */
    .name =             "usb-storage",
    .proc_name =            "usb-storage",
    .show_info =            show_info,
    .write_info =           write_info,
    .info =             host_info,

    /* command interface -- queued only */
    .queuecommand =         queuecommand,

    /* error and abort handlers */
    .eh_abort_handler =     command_abort,
    .eh_device_reset_handler =  device_reset,
    .eh_bus_reset_handler =     bus_reset,

    /* queue commands only, only one command per LUN */
    .can_queue =            1,

    /* unknown initiator id */
    .this_id =          -1,

    .slave_alloc =          slave_alloc,
    .slave_configure =      slave_configure,
    .target_alloc =         target_alloc,

    /* lots of sg segments can be handled */
    .sg_tablesize =         SG_MAX_SEGMENTS,

    /*
     * Limit the total size of a transfer to 120 KB.
     *
     * Some devices are known to choke with anything larger. It seems like
     * the problem stems from the fact that original IDE controllers had
     * only an 8-bit register to hold the number of sectors in one transfer
     * and even those couldn't handle a full 256 sectors.
     *
     * Because we want to make sure we interoperate with as many devices as
     * possible, we will maintain a 240 sector transfer size limit for USB
     * Mass Storage devices.
     *
     * Tests show that other operating have similar limits with Microsoft
     * Windows 7 limiting transfers to 128 sectors for both USB2 and USB3
     * and Apple Mac OS X 10.11 limiting transfers to 256 sectors for USB2
     * and 2048 for USB3 devices.
     */
    .max_sectors =                  240,

    /*
     * merge commands... this seems to help performance, but
     * periodically someone should test to see which setting is more
     * optimal.
     */
    .use_clustering =       1,

    /* emulated HBA */
    .emulated =         1,

    /* we do our own delay after a device or bus reset */
    .skip_settle_delay =        1,

    /* sysfs device attributes */
    .sdev_attrs =           sysfs_device_attr_list,

    /* module management */
    .module =           THIS_MODULE
};
```
&ensp;&ensp;&ensp;&ensp;这个结构体usb storage在scsi核心层最最关键的接口，结构体内实现了很多函数，scsi核心层会通过通过这些函数下发数据（queuecommand()函数），报异常（command_abort()函数），设备重置（device_reset()处理句柄），总线重置（bus_reset()处理句柄）等，还有scsi控制器信息显示函数相关show_info()，write_info()，host_info()等。此处只关注queuecommand()和command_abort()函数，待后续介绍（第一次调用处）。

### 1.2 匹配函数分析
&ensp;&ensp;&ensp;&ensp;现继续跟踪1.1节介绍过的usb_storage_driver结构体实现。
```C
static struct usb_driver usb_storage_driver = {
    .name =     DRV_NAME,
    .probe =    storage_probe,
    .disconnect =   usb_stor_disconnect,
    .suspend =  usb_stor_suspend,
    .resume =   usb_stor_resume,
    .reset_resume = usb_stor_reset_resume,
    .pre_reset =    usb_stor_pre_reset,
    .post_reset =   usb_stor_post_reset,
    .id_table = usb_storage_usb_ids,
    .supports_autosuspend = 1,
    .soft_unbind =  1,
};
```
&ensp;&ensp;&ensp;&ensp;这其中主要看三个元素实现：.probe()、.disconnect()、.id_table，其它部分暂时没能力和精力分析清楚。现看id_table对应的实现：usb_storage_usb_ids，代码实现如下：

```C
struct usb_device_id usb_storage_usb_ids[] = {
#   include "unusual_devs.h"
    { }     /* Terminating entry */
}; 
MODULE_DEVICE_TABLE(usb, usb_storage_usb_ids);
```
&ensp;&ensp;&ensp;&ensp;unusual_devs，顾名思义，不是通用设备，这个文件不简单，是所有不常用的USB存储设备及常用的USB存储设备的集合，稍后会继续介绍。此处即是对新匹配的设备进行设备，若在设备列表内，则执行probe()匹配函数。进unusual_devs.h文件查看一下，此处该文件在两处使用了。此处作为usb_storage_usb_ids[]结构体内元素使用，另一处在probe()函数中使用：

```C
/* patch submitted by Vivian Bregier <Vivian.Bregier@imag.fr> */
UNUSUAL_DEV(  0x03eb, 0x2002, 0x0100, 0x0100,     /* 不常用设备 */
        "ATMEL",
        "SND1 Storage",
        USB_SC_DEVICE, USB_PR_DEVICE, NULL,
        US_FL_IGNORE_RESIDUE),
        
...
/* 通用设备 */
/* Control/Bulk transport for all SubClass values */
USUAL_DEV(USB_SC_RBC, USB_PR_CB),
USUAL_DEV(USB_SC_8020, USB_PR_CB),
USUAL_DEV(USB_SC_QIC, USB_PR_CB),
USUAL_DEV(USB_SC_UFI, USB_PR_CB),
USUAL_DEV(USB_SC_8070, USB_PR_CB),
USUAL_DEV(USB_SC_SCSI, USB_PR_CB),

/* Control/Bulk/Interrupt transport for all SubClass values */
USUAL_DEV(USB_SC_RBC, USB_PR_CBI),
USUAL_DEV(USB_SC_8020, USB_PR_CBI),
USUAL_DEV(USB_SC_QIC, USB_PR_CBI),
USUAL_DEV(USB_SC_UFI, USB_PR_CBI),
USUAL_DEV(USB_SC_8070, USB_PR_CBI),
USUAL_DEV(USB_SC_SCSI, USB_PR_CBI),

/* Bulk-only transport for all SubClass values */
USUAL_DEV(USB_SC_RBC, USB_PR_BULK),
USUAL_DEV(USB_SC_8020, USB_PR_BULK),
USUAL_DEV(USB_SC_QIC, USB_PR_BULK),
USUAL_DEV(USB_SC_UFI, USB_PR_BULK),
USUAL_DEV(USB_SC_8070, USB_PR_BULK),
USUAL_DEV(USB_SC_SCSI, USB_PR_BULK),  /* U盘使用此宏定义 */
```
其中UNUSUAL_DEV()和USUAL_DEV()宏定义在此处为如下：

```C
/* 
 * The table of devices
 */ 
#define UNUSUAL_DEV(id_vendor, id_product, bcdDeviceMin, bcdDeviceMax, \
            vendorName, productName, useProtocol, useTransport, \
            initFunction, flags) \
{ USB_DEVICE_VER(id_vendor, id_product, bcdDeviceMin, bcdDeviceMax), \
  .driver_info = (flags) }
            
#define COMPLIANT_DEV   UNUSUAL_DEV

#define USUAL_DEV(useProto, useTrans) \ 
{ USB_INTERFACE_INFO(USB_CLASS_MASS_STORAGE, useProto, useTrans) }
...
#undef UNUSUAL_DEV  /* 注销宏定义*/
#undef COMPLIANT_DEV
#undef USUAL_DEV
```

#### 1.2.1 probe()函数
&ensp;&ensp;&ensp;&ensp;现在开始主要介绍probe()函数，先看函数：

```C
/* The main probe routine for standard devices */
static int storage_probe(struct usb_interface *intf,
             const struct usb_device_id *id)
{
    struct us_unusual_dev *unusual_dev;
    struct us_data *us;
    int result;
    int size;

    /* If uas is enabled and this device can do uas then ignore it. */
#if IS_ENABLED(CONFIG_USB_UAS) /* uas: usb attached scsi protocol */
    if (uas_use_uas_driver(intf, id, NULL))
        return -ENXIO;
#endif

    /*
     * If the device isn't standard (is handled by a subdriver
     * module) then don't accept it.
     */
    if (usb_usual_ignore_device(intf))
        return -ENXIO;

    /*
     * Call the general probe procedures.
     *
     * The unusual_dev_list array is parallel to the usb_storage_usb_ids
     * table, so we use the index of the id entry to find the
     * corresponding unusual_devs entry.
     */

    size = ARRAY_SIZE(us_unusual_dev_list);
    if (id >= usb_storage_usb_ids && id < usb_storage_usb_ids + size) {
        unusual_dev = (id - usb_storage_usb_ids) + us_unusual_dev_list;
    } else {
        unusual_dev = &for_dynamic_ids;
    
        dev_dbg(&intf->dev, "Use Bulk-Only transport with the Transparent SCSI protocol for dynamic id: 0x%04x 0x%04x\n",
            id->idVendor, id->idProduct);
    }
    
    result = usb_stor_probe1(&us, intf, id, unusual_dev,
                 &usb_stor_host_template);
    if (result)
        return result;
    
    /* No special transport or protocol settings in the main module */

    result = usb_stor_probe2(us);
    return result;
}
```

&ensp;&ensp;&ensp;&ensp;uas_use_uas_driver()函数，暂时不进行解读，但是UAS/UASP可以简单介绍以下：

- Bulk Only Transport（下简称**[BOT]**）被用于**[USB2]**海量存储类设备，在微控制器中实现是简单而廉价的，因此适合于价格低廉的基于闪存的存储产品；
- **[BOT]**是单线程事务处理，每个由host初始化好的事务，只有在被device处理完成且处理完成状态传回给初始化线程之后才能开始下一个事务处理。这将在整个数据传输中产生显著的开销（约20%）。**[USB3]**将数据传输带宽从480Mb/s增加到5Gb/s，但如果**[BOT]**协议仍然存在，则分析表明，2.4GHz核心Duo<sup>TM</sup>上的实际传输速率将约为250Mb/s，以及12%的CPU使用率；
- UASP(USB Attached SCSI Protocol)将提高**[USB2]**的效率，并利用**[USB3]**全双工功能允许超过400Mb/s的数据传输。新协议将需要新的host软件和device固件，通过支持**[BOT]**和**[UAS]**的设备来实现向后兼容。


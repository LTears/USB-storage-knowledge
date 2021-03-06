# USB Storage代码流程分析

[TOC]

---

## 0. 前言
&ensp;&ensp;&ensp;&ensp;当前已从linux kernel剥离出USB storage驱动代码，并做成源码树形式，可通过ctags或source insight等工具进行解读，路径在git-usb目录下，其中代码已添加必要注释，以下会按照函数流程做主要代码的解读。

## 1. 函数入口
### 1.1 注册函数分析
&ensp;&ensp;&ensp;&ensp;函数入口在文件drivers/usb/storage/usb.c中，为最后一行代码：

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
/* 驱动名，后续很多地方用到 */
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
&ensp;&ensp;&ensp;&ensp;即将传入的参数sht与具体的实现usb_stor_host_template结构体联结，结构体usb_stor_host_template在该段代码实现之上定义：

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
&ensp;&ensp;&ensp;&ensp;这个结构体是usb storage与scsi核心层最最关键的接口，结构体内实现了很多函数，scsi核心层会通过这些函数下发数据（queuecommand()函数），报异常（command_abort()函数），设备重置（device_reset()处理句柄），总线重置（bus_reset()处理句柄）等，还有scsi控制器信息显示相关函数show_info()，write_info()，host_info()等。此处只关注queuecommand()和command_abort()函数，且待后续用到才一一介绍。

### 1.2 驱动结构体分析
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
&ensp;&ensp;&ensp;&ensp;这其中主要看三个元素实现：.probe()、.disconnect()、.id_table，其它部分暂时没能力和精力分析清楚。先看id_table对应的实现：usb_storage_usb_ids，代码实现如下：

```C
struct usb_device_id usb_storage_usb_ids[] = {
#   include "unusual_devs.h"
    { }     /* Terminating entry */
}; 
MODULE_DEVICE_TABLE(usb, usb_storage_usb_ids);
```
&ensp;&ensp;&ensp;&ensp;unusual_devs，顾名思义，不是通用设备，这个文件不简单，是所有不常用的USB存储设备及常用的USB存储设备的集合，稍后会继续介绍。此处即是对新匹配的设备进行设备，若在设备列表内，则执行probe()匹配函数。进unusual_devs.h文件查看一下，此处该文件在两处使用了。此处作为usb_storage_usb_ids[]结构体内元素第一次使用，另一处在probe()函数中使用：

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
&ensp;&ensp;&ensp;&ensp;其中此处的UNUSUAL_DEV()和USUAL_DEV()宏定义在drivers/usb/storage/usual-tables.c中定义，仅用于驱动匹配设备时之用：

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

## 2. probe()函数
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

现在开始从上往下分析：
### 2.1 函数变量分析
&ensp;&ensp;&ensp;&ensp;storage_probe()函数声明了四个函数变量，代码如下：
```C
    struct us_unusual_dev *unusual_dev;
    struct us_data *us;
    int result;
    int size;
```
&ensp;&ensp;&ensp;&ensp;其中变量unusual_dev不常用device列表结构体的实例，struct us_unusual_dev结构体定义如下，在probe函数中用于排查不常用device之用：

```C
/*
 * Unusual device list definitions 
 */
struct us_unusual_dev {
    const char* vendorName;
    const char* productName;
    __u8  useProtocol;
    __u8  useTransport;
    int (*initFunction)(struct us_data *);
};
```
&ensp;&ensp;&ensp;&ensp;struct us_data结构体是USB storage驱动中自定义的贯穿始终的一个非常重要的结构体，相当于一个全局变量了。变量名**us**即usb storage。
&ensp;&ensp;&ensp;&ensp;result变量为函数结果收集，size变量作为不常用device集合的个数变量在用。

### 2.2 uas_use_uas_driver()函数解析
&ensp;&ensp;&ensp;&ensp;uas_use_uas_driver()函数，暂时不进行解读，但是UAS/UASP可以简单介绍以下：

- Bulk Only Transport（以下简称[**BOT**]）被用于[**USB2**]海量存储类设备，在微控制器中实现是简单而廉价的，因此适合于价格低廉的基于闪存的存储产品；
- [**BOT**]是单线程事务处理，每个由host初始化好的事务，只有在被device处理完成且处理完成状态传回给初始化线程之后才能开始下一个事务处理。这将在整个数据传输中产生显著的开销（约20%）。[**USB3**]将数据传输带宽从480Mb/s增加到5Gb/s，但如果[**BOT**]协议仍然存在，则分析表明，2.4GHz核心Duo<sup>TM</sup>上的实际传输速率将约为250Mb/s，以及12%的CPU使用率；
- **UASP**(USB Attached SCSI Protocol)将提高[**USB2**]的效率，并利用[**USB3**]全双工功能允许超过400Mb/s的数据传输。新协议将需要新的host软件和device固件，通过支持[**BOT**]和[**UAS**]的设备来实现向后兼容。

### 2.3 usb_usual_ignore_device()函数解析
&ensp;&ensp;&ensp;&ensp;函数实现在drivers/usb/storage/usual-tables.c中：
```C
#define UNUSUAL_DEV(id_vendor, id_product, bcdDeviceMin, bcdDeviceMax, \
            vendorName, productName, useProtocol, useTransport, \
            initFunction, flags) \
{                   \
    .vid    = id_vendor,        \
    .pid    = id_product,       \
    .bcdmin = bcdDeviceMin,     \
    .bcdmax = bcdDeviceMax,     \
}
    
static struct ignore_entry ignore_ids[] = {
#   include "unusual_alauda.h"
#   include "unusual_cypress.h"
#   include "unusual_datafab.h"
#   include "unusual_ene_ub6250.h"
#   include "unusual_freecom.h"
#   include "unusual_isd200.h"
#   include "unusual_jumpshot.h"
#   include "unusual_karma.h"
#   include "unusual_onetouch.h"
#   include "unusual_realtek.h"
#   include "unusual_sddr09.h"
#   include "unusual_sddr55.h"
#   include "unusual_usbat.h"
    { }     /* Terminating entry */
};

#undef UNUSUAL_DEV

/* Return an error if a device is in the ignore_ids list */
int usb_usual_ignore_device(struct usb_interface *intf)
{
    struct usb_device *udev;
    unsigned vid, pid, bcd;
    struct ignore_entry *p;

    udev = interface_to_usbdev(intf);
    vid = le16_to_cpu(udev->descriptor.idVendor);
    pid = le16_to_cpu(udev->descriptor.idProduct);
    bcd = le16_to_cpu(udev->descriptor.bcdDevice);

    for (p = ignore_ids; p->vid; ++p) {
        if (p->vid == vid && p->pid == pid && 
                p->bcdmin <= bcd && p->bcdmax >= bcd)
            return -ENXIO;
    }
    return 0;
} 
```
&ensp;&ensp;&ensp;&ensp;此处又重新定义了一遍**UNUSUAL_DEV**宏定义，usb_usual_ignore_device()函数的功能即是排查匹配上的设备，如果不是标准device，则退出此通用USB storage驱动，而去匹配对应的子模块驱动。此处也不做深究。

### 2.4 unusual_dev变量赋值
&ensp;&ensp;&ensp;&ensp;接下来是对变量unusual_dev的赋值，代码如下：
```C
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
```
&ensp;&ensp;&ensp;&ensp;其中us_unusual_dev_list为usb.c中定义的静态变量，代码如下：
```C
/*
 * The entries in this table correspond, line for line,
 * with the entries in usb_storage_usb_ids[], defined in usual-tables.c.
 */

/*
 *The vendor name should be kept at eight characters or less, and
 * the product name should be kept at 16 characters or less. If a device
 * has the US_FL_FIX_INQUIRY flag, then the vendor and product names
 * normally generated by a device through the INQUIRY response will be
 * taken from this list, and this is the reason for the above size
 * restriction. However, if the flag is not present, then you
 * are free to use as many characters as you like.
 */

#define UNUSUAL_DEV(idVendor, idProduct, bcdDeviceMin, bcdDeviceMax, \
            vendor_name, product_name, use_protocol, use_transport, \
            init_function, Flags) \
{ \
    .vendorName = vendor_name,  \
    .productName = product_name,    \
    .useProtocol = use_protocol,    \
    .useTransport = use_transport,  \
    .initFunction = init_function,  \
}

#define COMPLIANT_DEV   UNUSUAL_DEV

#define USUAL_DEV(use_protocol, use_transport) \
{ \
    .useProtocol = use_protocol,    \
    .useTransport = use_transport,  \
}

#define UNUSUAL_VENDOR_INTF(idVendor, cl, sc, pr, \
        vendor_name, product_name, use_protocol, use_transport, \
        init_function, Flags) \
{ \
    .vendorName = vendor_name,  \
    .productName = product_name,    \
    .useProtocol = use_protocol,    \
    .useTransport = use_transport,  \
    .initFunction = init_function,  \
}

static struct us_unusual_dev us_unusual_dev_list[] = {
#   include "unusual_devs.h"
    { }     /* Terminating entry */
};

static struct us_unusual_dev for_dynamic_ids =
        USUAL_DEV(USB_SC_SCSI, USB_PR_BULK);

#undef UNUSUAL_DEV
#undef COMPLIANT_DEV
#undef USUAL_DEV
#undef UNUSUAL_VENDOR_INTF
```
&ensp;&ensp;&ensp;&ensp;此处做了以下几件事：

- 将unusual_devs.h文件引入作为结构体元素，在1.2节中介绍过这个.h文件，其中包含了不常用USB存储device和标准USB存储device，此处只是将**UNUSUAL_DEV**和**USUAL_DEV**宏定义重定义了一下；并引入**UNUSUAL_VENDOR_INTF**宏定义，请记住此处的init_function，后续还会用此函数实现来完成某些device特定的软硬件件操作。
- 通过usb_storage_usb_ids变量来定位匹配上的device在unusual_devs.h中的位置，再来找到在us_unusual_dev_list中的位置；
- 定义了静态变量for_dynamic_ids，其也是struct us_unusual_dev结构体实例，即：
   - 将.useProtocol赋值为USB_SC_SCSI；
   - 将.useTransport赋值为USB_PR_BULK。

### 2.5 usb_stor_probe1()函数解析
&ensp;&ensp;&ensp;&ensp;函数实现如下，稍后各小节再逐条进行分析：
```C
/* First part of general USB mass-storage probing */
int usb_stor_probe1(struct us_data **pus,
        struct usb_interface *intf,
        const struct usb_device_id *id,
        struct us_unusual_dev *unusual_dev,
        struct scsi_host_template *sht)
{
    struct Scsi_Host *host;
    struct us_data *us;
    int result;

    dev_info(&intf->dev, "USB Mass Storage device detected\n");

    /*
     * Ask the SCSI layer to allocate a host structure, with extra
     * space at the end for our private us_data structure.
     */
    host = scsi_host_alloc(sht, sizeof(*us));
    if (!host) {
        dev_warn(&intf->dev, "Unable to allocate the scsi host\n");
        return -ENOMEM;
    }
    
    /*
     * Allow 16-byte CDBs and thus > 2TB
     */
    host->max_cmd_len = 16;
    host->sg_tablesize = usb_stor_sg_tablesize(intf);
    *pus = us = host_to_us(host);
    mutex_init(&(us->dev_mutex));
    us_set_lock_class(&us->dev_mutex, intf);
    init_completion(&us->cmnd_ready);
    init_completion(&(us->notify));
    init_waitqueue_head(&us->delay_wait);
    INIT_DELAYED_WORK(&us->scan_dwork, usb_stor_scan_dwork);

    /* Associate the us_data structure with the USB device */
    result = associate_dev(us, intf);
    if (result)
        goto BadDevice;

    /* Get the unusual_devs entries and the descriptors */
    result = get_device_info(us, id, unusual_dev);
    if (result)
        goto BadDevice;

    /* Get standard transport and protocol settings */
    get_transport(us);
    get_protocol(us);

    /*
     * Give the caller a chance to fill in specialized transport
     * or protocol settings.
     */
    return 0;

BadDevice:
    usb_stor_dbg(us, "storage_probe() failed\n");
    release_everything(us);
    return result;
}
EXPORT_SYMBOL_GPL(usb_stor_probe1);
```
#### 2.5.1 Scsi_Host结构体变量分析
&ensp;&ensp;&ensp;&ensp;该函数开头定义了三个变量：
```C
    struct Scsi_Host *host;
    struct us_data *us;
    int result;
```
&ensp;&ensp;&ensp;&ensp;struct Scsi_Host结构体实例host，此处简单分析一下struct scsi_host_template和struct Scsi_Host结构体的关系，在Linux中，每一个scsi主机控制器对应一个数据结构，Scsi_Host（而Linux中将通过使用一个scsi_host_template结构指针为参数的函数来为Scsi_Host初始化，scsi_host_template实现控制器的操作函数以及命令封装，两个结构体包含了很多相同的元素，但又不完全相同，他们协同工作，互相关联，但是各自起的作用不一样，且Scsi_Host有且仅有一个struct scsi_host_template），但有些Scsi_Host对应的并非是真实的scsi卡，虽然硬件是并不存在，但仍然需要一个Scsi_Host，如U盘，因为她被模拟成SCSI设备，所以得为她准备一个SCSI卡，这个可以在插入U盘后，通过cat /proc/scsi/scsi的输出来了解：
```C
$ cat /proc/scsi/scsi
Attached devices:
Host: scsi1 Channel: 00 Id: 00 Lun: 00
  Vendor: ATA      Model: WDC WD10JPVX-22J Rev: 1A01
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi3 Channel: 00 Id: 00 Lun: 00
  Vendor: TSSTcorp Model: CDDVDW SH-224DB  Rev: CM00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi4 Channel: 00 Id: 00 Lun: 00
  Vendor: Netac    Model: OnlyDisk         Rev: 1.0 
  Type:   Direct-Access                    ANSI  SCSI revision: 02
```
&ensp;&ensp;&ensp;&ensp;当前在/proc/scsi目录下还会因为插入U盘后，动态的生成usb-storage目录，里面即是每个新插入的U盘的一些信息，如当前插入的U盘，显示为4，其内信息如下:
```C
$ cat /proc/scsi/usb-storage/4 
   Host scsi4: usb-storage
       Vendor: Netac
      Product: OnlyDisk
Serial Number: 43A02D3FDE34408D
     Protocol: Transparent SCSI
    Transport: Bulk
       Quirks:
```

---
&ensp;&ensp;&ensp;&ensp;struct us_data已经介绍，一个usb storage驱动实现的结构体，非常的重要。int result作为函数结构检查；
#### 2.5.2 host初始化
&ensp;&ensp;&ensp;&ensp;通过将&usb_stor_host_template作为参数传给scsi_host_alloc()函数，分配一个struct Scsi_Host结构体实例，以及相应内存等，此处将us也作为参数传入，是在host的unsigned long hostdata[0]元素中分配了struct us_data结构体内存，作为其私有数据以供后续使用。（此处不继续深究scsi_host_alloc()实现）
```C
    /*
     * Ask the SCSI layer to allocate a host structure, with extra
     * space at the end for our private us_data structure.
     */
    host = scsi_host_alloc(sht, sizeof(*us));
    if (!host) {
        dev_warn(&intf->dev, "Unable to allocate the scsi host\n");
        return -ENOMEM;
    }
```
#### 2.5.3 host填充及us部分初始化
```C
    /*
     * Allow 16-byte CDBs and thus > 2TB
     */
    host->max_cmd_len = 16;
    host->sg_tablesize = usb_stor_sg_tablesize(intf);
    *pus = us = host_to_us(host);
    mutex_init(&(us->dev_mutex));
    us_set_lock_class(&us->dev_mutex, intf);
    init_completion(&us->cmnd_ready);
    init_completion(&(us->notify));
    init_waitqueue_head(&us->delay_wait);
    INIT_DELAYED_WORK(&us->scan_dwork, usb_stor_scan_dwork);
```
&ensp;&ensp;&ensp;&ensp;定义host的元素max_cmd_len为16，以及sg_tablesize大小，通过intf对应的usb设备所在的总线支持的sg_tablesize大小：
```C
static unsigned int usb_stor_sg_tablesize(struct usb_interface *intf)
{
    struct usb_device *usb_dev = interface_to_usbdev(intf);

    if (usb_dev->bus->sg_tablesize) {
        return usb_dev->bus->sg_tablesize;
    }
    return SG_ALL;
}
```
&ensp;&ensp;&ensp;&ensp;将函数传入的struct us_data二级指针参数pus与本函数内定义的us和host相关联：
```C
    *pus = us = host_to_us(host);
```
&ensp;&ensp;&ensp;&ensp;接下来则是各类锁，信号量，等待队列，延迟工作队列的初始化：
```C
    mutex_init(&(us->dev_mutex));
    us_set_lock_class(&us->dev_mutex, intf);
    init_completion(&us->cmnd_ready);
    init_completion(&(us->notify));
    init_waitqueue_head(&us->delay_wait);
    INIT_DELAYED_WORK(&us->scan_dwork, usb_stor_scan_dwork);
```
&ensp;&ensp;&ensp;&ensp;此处需要注意的是us->cmnd_ready和us->notify，其在scsi cmd和abort传递上实现了内核同步机制。此处初始化了一个延迟工作delay_work，稍后在调用的地方再讲解这个实现的函数：usb_stor_scan_dwork()。
#### 2.5.4 associate_dev()实现
```C
    /* Associate the us_data structure with the USB device */
    result = associate_dev(us, intf);
    if (result)
        goto BadDevice;
```
&ensp;&ensp;&ensp;&ensp;正如注释所说，是将USB device和us变量联结起来，代码实现如下：
```C
/* Associate our private data with the USB device */
static int associate_dev(struct us_data *us, struct usb_interface *intf)
{
    /* Fill in the device-related fields */
    us->pusb_dev = interface_to_usbdev(intf);
    us->pusb_intf = intf;
    us->ifnum = intf->cur_altsetting->desc.bInterfaceNumber;
    usb_stor_dbg(us, "Vendor: 0x%04x, Product: 0x%04x, Revision: 0x%04x\n",
             le16_to_cpu(us->pusb_dev->descriptor.idVendor),
             le16_to_cpu(us->pusb_dev->descriptor.idProduct),
             le16_to_cpu(us->pusb_dev->descriptor.bcdDevice));
    usb_stor_dbg(us, "Interface Subclass: 0x%02x, Protocol: 0x%02x\n",
             intf->cur_altsetting->desc.bInterfaceSubClass,
             intf->cur_altsetting->desc.bInterfaceProtocol);
    
    /* Store our private data in the interface */
    usb_set_intfdata(intf, us);
        
    /* Allocate the control/setup and DMA-mapped buffers */
    us->cr = kmalloc(sizeof(*us->cr), GFP_KERNEL);
    if (!us->cr)
        return -ENOMEM;

    us->iobuf = usb_alloc_coherent(us->pusb_dev, US_IOBUF_SIZE,
            GFP_KERNEL, &us->iobuf_dma);
    if (!us->iobuf) {
        usb_stor_dbg(us, "I/O buffer allocation failed\n");
        return -ENOMEM;
    }
    return 0;
}
```
&ensp;&ensp;&ensp;&ensp;这其中做了以下几件事：
- 将us->pusb_dev与intf对应的USB device联结；
- 将us->pusb_intf与intf联结；
- 将us->ifnum赋值为intf对应的当前设置接口下的描述符bInterfaceNumber；
- 将us作为intf私有数据：usb_set_intfdata(intf, us)；
- 分配us下的控制传输buffer：
```C
    /* Allocate the control/setup and DMA-mapped buffers */
    us->cr = kmalloc(sizeof(*us->cr), GFP_KERNEL);
    if (!us->cr)
        return -ENOMEM;
```
- 分配I/O buffer的DMA内存，大小为64byte；

#### 2.5.5 get_device_info()实现
```C
    /* Get the unusual_devs entries and the descriptors */
    result = get_device_info(us, id, unusual_dev);
    if (result)
        goto BadDevice;
```
&ensp;&ensp;&ensp;&ensp;get_device_info()实现如下，用于获取USB device的硬件信息：
```C
/* Get the unusual_devs entries and the string descriptors */
static int get_device_info(struct us_data *us, const struct usb_device_id *id,
        struct us_unusual_dev *unusual_dev)
{
    struct usb_device *dev = us->pusb_dev;
    struct usb_interface_descriptor *idesc =
        &us->pusb_intf->cur_altsetting->desc;
    struct device *pdev = &us->pusb_intf->dev;

    /* Store the entries */
    us->unusual_dev = unusual_dev;
    us->subclass = (unusual_dev->useProtocol == USB_SC_DEVICE) ?
            idesc->bInterfaceSubClass :
            unusual_dev->useProtocol;
    us->protocol = (unusual_dev->useTransport == USB_PR_DEVICE) ?
            idesc->bInterfaceProtocol :
            unusual_dev->useTransport;
    us->fflags = id->driver_info;
    usb_stor_adjust_quirks(us->pusb_dev, &us->fflags);

    if (us->fflags & US_FL_IGNORE_DEVICE) {
        dev_info(pdev, "device ignored\n");
        return -ENODEV;
    }

    /*
     * This flag is only needed when we're in high-speed, so let's
     * disable it if we're in full-speed
     */
    if (dev->speed != USB_SPEED_HIGH)
        us->fflags &= ~US_FL_GO_SLOW;

    if (us->fflags)
        dev_info(pdev, "Quirks match for vid %04x pid %04x: %lx\n",
                le16_to_cpu(dev->descriptor.idVendor),
                le16_to_cpu(dev->descriptor.idProduct),
                us->fflags);

    /*
     * Log a message if a non-generic unusual_dev entry contains an
     * unnecessary subclass or protocol override.  This may stimulate
     * reports from users that will help us remove unneeded entries
     * from the unusual_devs.h table.
     */
    if (id->idVendor || id->idProduct) {
        static const char *msgs[3] = {
            "an unneeded SubClass entry",
            "an unneeded Protocol entry",
            "unneeded SubClass and Protocol entries"};
        struct usb_device_descriptor *ddesc = &dev->descriptor;
        int msg = -1;

        if (unusual_dev->useProtocol != USB_SC_DEVICE &&
            us->subclass == idesc->bInterfaceSubClass)
            msg += 1;
        if (unusual_dev->useTransport != USB_PR_DEVICE &&
            us->protocol == idesc->bInterfaceProtocol)
            msg += 2;
        if (msg >= 0 && !(us->fflags & US_FL_NEED_OVERRIDE))
            dev_notice(pdev, "This device "
                    "(%04x,%04x,%04x S %02x P %02x)"
                    " has %s in unusual_devs.h (kernel"
                    " %s)\n"
                    "   Please send a copy of this message to "
                    "<linux-usb@vger.kernel.org> and "
                    "<usb-storage@lists.one-eyed-alien.net>\n",
                    le16_to_cpu(ddesc->idVendor),
                    le16_to_cpu(ddesc->idProduct),
                    le16_to_cpu(ddesc->bcdDevice),
                    idesc->bInterfaceSubClass,
                    idesc->bInterfaceProtocol,
                    msgs[msg],
                    utsname()->release);
    }

    return 0;
}
```
&ensp;&ensp;&ensp;&ensp;该函数做了以下处理：
- 关联us->unusual_dev和unusual_dev；
- 填充us->subclass、us->protocol、us->fflags，这些信息都和unusual_devs.h有关，其中fflags为某些特定功能标记，算得上对设备做限制定制了，举例如下：
```C
/*
 * Nick Bowler <nbowler@elliptictech.com>
 * SCSI stack spams (otherwise harmless) error messages.
 */
UNUSUAL_DEV(  0xc251, 0x4003, 0x0100, 0x0100,
        "Keil Software, Inc.",
        "V2M MotherBoard",
        USB_SC_DEVICE, USB_PR_DEVICE, NULL,
        US_FL_NOT_LOCKABLE),
USUAL_DEV(USB_SC_SCSI, USB_PR_BULK),
```
&ensp;&ensp;&ensp;&ensp;这其中，当为unusual_dev时，则unusual_dev->useProtocol == USB_SC_DEVICE，所以us->subclass就等于了idesc->bInterfaceSubClass，而当为通用存储设备时，则us->subclass等于了unusual_dev->useProtocol，同理us->protocol也一样，而us->fflags对应的即是US_FL_NOT_LOCKABLE，这宏定义在include/linux/usb_usual.h中，：
```C
#define US_DO_ALL_FLAGS                     \
    US_FLAG(SINGLE_LUN, 0x00000001)         \
        /* allow access to only LUN 0 */        \
    US_FLAG(NEED_OVERRIDE,  0x00000002)         \
        /* unusual_devs entry is necessary */       \
    US_FLAG(SCM_MULT_TARG,  0x00000004)         \
        /* supports multiple targets */         \
    US_FLAG(FIX_INQUIRY,    0x00000008)         \
        /* INQUIRY response needs faking */     \
    US_FLAG(FIX_CAPACITY,   0x00000010)         \
        /* READ CAPACITY response too big */        \
    US_FLAG(IGNORE_RESIDUE, 0x00000020)         \
        /* reported residue is wrong */         \
    US_FLAG(BULK32,     0x00000040)         \
        /* Uses 32-byte CBW length */           \
    US_FLAG(NOT_LOCKABLE,   0x00000080)         \
        /* PREVENT/ALLOW not supported */       \
    US_FLAG(GO_SLOW,    0x00000100)         \
        /* Need delay after Command phase */        \
    US_FLAG(NO_WP_DETECT,   0x00000200)         \
        /* Don't check for write-protect */     \
    US_FLAG(MAX_SECTORS_64, 0x00000400)         \
        /* Sets max_sectors to 64    */         \
    US_FLAG(IGNORE_DEVICE,  0x00000800)         \
        /* Don't claim device */            \
    US_FLAG(CAPACITY_HEURISTICS,    0x00001000)     \
        /* sometimes sizes is too big */        \
    US_FLAG(MAX_SECTORS_MIN,0x00002000)         \
        /* Sets max_sectors to arch min */      \
    US_FLAG(BULK_IGNORE_TAG,0x00004000)         \
        /* Ignore tag mismatch in bulk operations */    \
    US_FLAG(SANE_SENSE,     0x00008000)         \
        /* Sane Sense (> 18 bytes) */           \
    US_FLAG(CAPACITY_OK,    0x00010000)         \
        /* READ CAPACITY response is correct */     \
    US_FLAG(BAD_SENSE,  0x00020000)         \
        /* Bad Sense (never more than 18 bytes) */  \
    US_FLAG(NO_READ_DISC_INFO,  0x00040000)     \
        /* cannot handle READ_DISC_INFO */      \
    US_FLAG(NO_READ_CAPACITY_16,    0x00080000)     \
        /* cannot handle READ_CAPACITY_16 */        \
    US_FLAG(INITIAL_READ10, 0x00100000)         \
        /* Initial READ(10) (and others) must be retried */ \
    US_FLAG(WRITE_CACHE,    0x00200000)         \
        /* Write Cache status is not available */   \
    US_FLAG(NEEDS_CAP16,    0x00400000)         \
        /* cannot handle READ_CAPACITY_10 */        \
    US_FLAG(IGNORE_UAS, 0x00800000)         \
        /* Device advertises UAS but it is broken */    \
    US_FLAG(BROKEN_FUA, 0x01000000)         \
        /* Cannot handle FUA in WRITE or READ CDBs */   \
    US_FLAG(NO_ATA_1X,  0x02000000)         \
        /* Cannot handle ATA_12 or ATA_16 CDBs */   \
    US_FLAG(NO_REPORT_OPCODES,  0x04000000)     \
        /* Cannot handle MI_REPORT_SUPPORTED_OPERATION_CODES */ \
    US_FLAG(MAX_SECTORS_240,    0x08000000)     \
        /* Sets max_sectors to 240 */           \
    US_FLAG(NO_REPORT_LUNS, 0x10000000)         \
        /* Cannot handle REPORT_LUNS */         \
    US_FLAG(ALWAYS_SYNC, 0x20000000)            \
        /* lies about caching, so always sync */    \

#define US_FLAG(name, value)    US_FL_##name = value ,
enum { US_DO_ALL_FLAGS };
#undef US_FLAG
```
##### 2.5.5.1 usb_stor_adjust_quirks()解析
&ensp;&ensp;&ensp;&ensp;该函数的功能是向内核申请了一个/sys接口，路径是/sys/module/usb_storage/parameters/quirks，用户可以通过实时动态的对某些USB mass storage device添加限制标记。先看代码实现，再介绍如何添加特定设备限定标记。
###### 2.5.5.1.1 代码实现
&ensp;&ensp;&ensp;&ensp;代码实现下：
```C
/* Works only for digits and letters, but small and fast */
#define TOLOWER(x) ((x) | 0x20)

/* Adjust device flags based on the "quirks=" module parameter */
void usb_stor_adjust_quirks(struct usb_device *udev, unsigned long *fflags)
{
    char *p;
    u16 vid = le16_to_cpu(udev->descriptor.idVendor);
    u16 pid = le16_to_cpu(udev->descriptor.idProduct);
    unsigned f = 0;
    unsigned int mask = (US_FL_SANE_SENSE | US_FL_BAD_SENSE |
            US_FL_FIX_CAPACITY | US_FL_IGNORE_UAS |
            US_FL_CAPACITY_HEURISTICS | US_FL_IGNORE_DEVICE |
            US_FL_NOT_LOCKABLE | US_FL_MAX_SECTORS_64 |
            US_FL_CAPACITY_OK | US_FL_IGNORE_RESIDUE |
            US_FL_SINGLE_LUN | US_FL_NO_WP_DETECT |
            US_FL_NO_READ_DISC_INFO | US_FL_NO_READ_CAPACITY_16 |
            US_FL_INITIAL_READ10 | US_FL_WRITE_CACHE |
            US_FL_NO_ATA_1X | US_FL_NO_REPORT_OPCODES |
            US_FL_MAX_SECTORS_240 | US_FL_NO_REPORT_LUNS |
            US_FL_ALWAYS_SYNC);

    p = quirks;
    while (*p) {
        /* Each entry consists of VID:PID:flags */
        if (vid == simple_strtoul(p, &p, 16) &&
                *p == ':' &&
                pid == simple_strtoul(p+1, &p, 16) &&
                *p == ':')
            break;
    
        /* Move forward to the next entry */
        while (*p) {
            if (*p++ == ',')
                break;
        }
    }
    if (!*p)    /* No match */
        return;

    /* Collect the flags */
    while (*++p && *p != ',') {
        switch (TOLOWER(*p)) {
        case 'a':
            f |= US_FL_SANE_SENSE;
            break;
        case 'b':
            f |= US_FL_BAD_SENSE;
            break;
        case 'c':
            f |= US_FL_FIX_CAPACITY;
            break;
        case 'd':
            f |= US_FL_NO_READ_DISC_INFO;
            break;
        case 'e':
            f |= US_FL_NO_READ_CAPACITY_16;
            break;
        case 'f':
            f |= US_FL_NO_REPORT_OPCODES;
            break;
        case 'g':
            f |= US_FL_MAX_SECTORS_240;
            break;
        case 'h':
            f |= US_FL_CAPACITY_HEURISTICS;
            break;
        case 'i':
            f |= US_FL_IGNORE_DEVICE;
            break;
        case 'j':
            f |= US_FL_NO_REPORT_LUNS;
            break;
        case 'l':
            f |= US_FL_NOT_LOCKABLE;
            break;
        case 'm':
            f |= US_FL_MAX_SECTORS_64;
            break;
        case 'n':
            f |= US_FL_INITIAL_READ10;
            break;
        case 'o':
            f |= US_FL_CAPACITY_OK;
            break;
        case 'p':
            f |= US_FL_WRITE_CACHE;
            break;
        case 'r':
            f |= US_FL_IGNORE_RESIDUE;
            break;
        case 's':
            f |= US_FL_SINGLE_LUN;
            break;
        case 't':
            f |= US_FL_NO_ATA_1X;
            break;
        case 'u':
            f |= US_FL_IGNORE_UAS;
            break;
        case 'w':
            f |= US_FL_NO_WP_DETECT;
            break;
        case 'y':
            f |= US_FL_ALWAYS_SYNC;
            break;
        /* Ignore unrecognized flag characters */
        }
    }
    *fflags = (*fflags & ~mask) | f;
}
EXPORT_SYMBOL_GPL(usb_stor_adjust_quirks);
```
&ensp;&ensp;&ensp;&ensp;这部分代码实现的流程如下：
- 通过udev->descriptor.idVendor和udev->descriptor.idProduct取得当前匹配上的USB device的vid和pid；
- 通过p = quirks获取/sys接口下quirks接口内数据，内核中quirks变量定义在drivers/usb/storage/usb.c（即本代码的最前面），其是向sys文件系统申请/sys/module/usb_storage/parameters/quirks接口:
```C
static char quirks[128];
module_param_string(quirks, quirks, sizeof(quirks), S_IRUGO | S_IWUSR);
MODULE_PARM_DESC(quirks, "supplemental list of device IDs and their quirks");
```
- 第一个while()循环逐一解析quirks文件内字符串，按顺序匹配当前USB device的vid和pid，若匹配上，则退出循环；若当次循环没匹配上，则在第二个while()循环处继续跳过‘,’字符，进入下一次循环继续匹配vid和pid；
- 当USB device匹配上quirks接口内的vid和pid后，就可以进入第三个while()循环了，该循环通过前期已商定的协议，用户按协议输入对应的字符，以表示对应的宏定义，如‘a’代表US_FL_SANE_SENSE，‘i’代表US_FL_IGNORE_DEVICE等，可以使用多个标记；
- 最后，将获取到的f标记与mask掩码及fflags一起得到最终的fflags；

###### 2.5.5.1.2 USB storage quirks接口操作
&ensp;&ensp;&ensp;&ensp;下面以US_FL_IGNORE_DEVICE标记简单介绍一下如何操作quirks接口：
- 优先获取到需要忽略的USB mass storage device的vid和pid，如下，得到两个U盘的vid和pid分别为0930:6545和058f:6387；
```C
# lsusb
Bus 001 Device 015: ID 0930:6545 Toshiba Corp. Kingston DataTraveler 102/2.0 / HEMA Flash Drive 2 GB / PNY Attache 4GB Stick
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 002: ID 0d8c:0105 C-Media Electronics, Inc. CM108 Audio Controller
Bus 003 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 002 Device 010: ID 058f:6387 Alcor Micro Corp. Flash Drive
Bus 002 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 005 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 004 Device 003: ID 046d:c077 Logitech, Inc. M105 Optical Mouse
Bus 004 Device 002: ID 04f3:0103 Elan Microelectronics Corp. ActiveJet K-2024 Multimedia Keyboard
Bus 004 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
- 将连个U盘的vid和pid以及US_FL_IGNORE_DEVICE标记的代号‘i’按如下规则写入/sys/module/usb_storage/parameters/quirks接口文件中；
```C
~# echo "058f:6387:i,0930:6545:i" > /sys/module/usb_storage/parameters/quirks
~# cat /sys/module/usb_storage/parameters/quirks
058f:6387:i,0930:6545:i

```
- 当前已配置成功，则接下来再插入两个U盘中的随便哪个，都会无法驱动，dmesg大致如下,-19对应内核错误码为-ENODEV：
```C
[97564.445606] usb 2-1: new full-speed USB device number 11 using ohci-pci
[97564.660644] usb 2-1: not running at top speed; connect to a high speed hub
[97564.676651] usb 2-1: New USB device found, idVendor=058f, idProduct=6387
[97564.676659] usb 2-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[97564.676663] usb 2-1: Product: Mass Storage
[97564.676668] usb 2-1: Manufacturer: Generic
[97564.676672] usb 2-1: SerialNumber: 0A0067B7
[97564.676681] device: '2-1': device_add
[97564.676751] bus: 'usb': add device 2-1
[97564.677410] bus: 'usb': driver_probe_device: matched device 2-1 with driver usb
[97564.677416] bus: 'usb': really_probe: probing driver usb with device 2-1
[97564.677427] devices_kset: Moving 2-1 to end of list
[97564.678649] device: '2-1:1.0': device_add
[97564.678683] bus: 'usb': add device 2-1:1.0
[97564.678744] bus: 'usb': driver_probe_device: matched device 2-1:1.0 with driver usb-storage
[97564.678748] bus: 'usb': really_probe: probing driver usb-storage with device 2-1:1.0
[97564.678757] devices_kset: Moving 2-1:1.0 to end of list
[97564.678768] usb-storage 2-1:1.0: USB Mass Storage device detected
[97564.678876] usb-storage 2-1:1.0: device ignored
[97564.678931] usb-storage: probe of 2-1:1.0 rejects match -19
[97564.679008] device: 'ep_01': device_add
[97564.679033] device: 'ep_82': device_add
[97564.679073] driver: 'usb': driver_bound: bound to device '2-1'
[97564.679081] bus: 'usb': really_probe: bound device 2-1 to driver usb
[97564.679094] device: 'ep_00': device_add
```
##### 2.5.5.2 get_device_info()判断部分代码
&ensp;&ensp;&ensp;&ensp;get_device_info()剩下部分代码主要是对udev和unusual_dev做判断：
- 检查US_FL_IGNORE_DEVICE标记位，若设置了该位，则返回-ENODEV，即-19；
```C
    if (us->fflags & US_FL_IGNORE_DEVICE) {
        dev_info(pdev, "device ignored\n");
        return -ENODEV;
    }
```
- 检查是否需要设置US_FL_GO_SLOW标志，如代码解释：
```C
    /*
     * This flag is only needed when we're in high-speed, so let's
     * disable it if we're in full-speed
     */
    if (dev->speed != USB_SPEED_HIGH)
        us->fflags &= ~US_FL_GO_SLOW;
```
- 打印输出vid、pid以及fflags标志值：
```C
    if (us->fflags)
        dev_info(pdev, "Quirks match for vid %04x pid %04x: %lx\n",
                le16_to_cpu(dev->descriptor.idVendor),
                le16_to_cpu(dev->descriptor.idProduct),
                us->fflags);
```
- 用于判断当前匹配的非通用device若使用了通用的us->subclass或us->protocol时，则需要修改对应的unusual_devs.h文件；

&ensp;&ensp;&ensp;&ensp;至此，get_device_info()函数分析完成。
#### 2.5.6 get_transport()及get_protocol()函数介绍
```C
    /* Get standard transport and protocol settings */
    get_transport(us);
    get_protocol(us);
```
&ensp;&ensp;&ensp;&ensp;先简单介绍下这两个函数，后续在USB storage内核线程的地方再详细介绍；
##### 2.5.6.1 get_transport()函数一览
&ensp;&ensp;&ensp;&ensp;该函数为us->transport及us->transport_reset填充了处理句柄：
```C
/* Get the transport settings */
static void get_transport(struct us_data *us)
{
    switch (us->protocol) {
    case USB_PR_CB:
        us->transport_name = "Control/Bulk";
        us->transport = usb_stor_CB_transport;
        us->transport_reset = usb_stor_CB_reset;
        us->max_lun = 7;
        break;

    case USB_PR_CBI:
        us->transport_name = "Control/Bulk/Interrupt";
        us->transport = usb_stor_CB_transport;
        us->transport_reset = usb_stor_CB_reset;
        us->max_lun = 7;
        break;
    
    case USB_PR_BULK:  /* USB mass storage调用 */
        us->transport_name = "Bulk";
        us->transport = usb_stor_Bulk_transport;
        us->transport_reset = usb_stor_Bulk_reset;
        break;
    }
}
```
##### 2.5.6.2 get_protocol()函数一览
&ensp;&ensp;&ensp;&ensp;该函数为us->proto_handler填充了处理句柄：
```C
/* Get the protocol settings */
static void get_protocol(struct us_data *us)
{
    switch (us->subclass) {
    case USB_SC_RBC:
        us->protocol_name = "Reduced Block Commands (RBC)";
        us->proto_handler = usb_stor_transparent_scsi_command;
        break;
    
    case USB_SC_8020:
        us->protocol_name = "8020i";
        us->proto_handler = usb_stor_pad12_command;
        us->max_lun = 0;
        break;

    case USB_SC_QIC:
        us->protocol_name = "QIC-157";
        us->proto_handler = usb_stor_pad12_command;
        us->max_lun = 0;
        break;
    
    case USB_SC_8070:
        us->protocol_name = "8070i";
        us->proto_handler = usb_stor_pad12_command;
        us->max_lun = 0;
        break; 

    case USB_SC_SCSI:  /* USB mass storage遵循scsi传输协议 */
        us->protocol_name = "Transparent SCSI";
        us->proto_handler = usb_stor_transparent_scsi_command;
        break;

    case USB_SC_UFI:
        us->protocol_name = "Uniform Floppy Interface (UFI)";
        us->proto_handler = usb_stor_ufi_command;
        break;
    }
}
```
&ensp;&ensp;&ensp;&ensp;至此usb_stor_probe1()函数介绍完全，返回其上级调用函数。

### 2.6 usb_stor_probe2()函数解析
```C
    /* No special transport or protocol settings in the main module */
    result = usb_stor_probe2(us);
```
&ensp;&ensp;&ensp;&ensp;函数实现如下，其后各节开始逐步解析该函数实现：
```C
/* Second part of general USB mass-storage probing */
int usb_stor_probe2(struct us_data *us)
{
    int result;
    struct device *dev = &us->pusb_intf->dev;

    /* Make sure the transport and protocol have both been set */
    if (!us->transport || !us->proto_handler) {
        result = -ENXIO;
        goto BadDevice;
    }
    usb_stor_dbg(us, "Transport: %s\n", us->transport_name);
    usb_stor_dbg(us, "Protocol: %s\n", us->protocol_name);

    if (us->fflags & US_FL_SCM_MULT_TARG) {
        /* 
         * SCM eUSCSI bridge devices can have different numbers
         * of LUNs on different targets; allow all to be probed.
         */ 
        us->max_lun = 7;
        /* The eUSCSI itself has ID 7, so avoid scanning that */
        us_to_host(us)->this_id = 7;
        /* max_id is 8 initially, so no need to set it here */
    } else {
        /* In the normal case there is only a single target */
        us_to_host(us)->max_id = 1;
        /*
         * Like Windows, we won't store the LUN bits in CDB[1] for 
         * SCSI-2 devices using the Bulk-Only transport (even though
         * this violates the SCSI spec).
         */
        if (us->transport == usb_stor_Bulk_transport)
            us_to_host(us)->no_scsi2_lun_in_cdb = 1;
    }

    /* fix for single-lun devices */
    if (us->fflags & US_FL_SINGLE_LUN)
        us->max_lun = 0;

    /* Find the endpoints and calculate pipe values */
    result = get_pipes(us);
    if (result)
        goto BadDevice;

    /*
     * If the device returns invalid data for the first READ(10)
     * command, indicate the command should be retried.
     */
    if (us->fflags & US_FL_INITIAL_READ10)
        set_bit(US_FLIDX_REDO_READ10, &us->dflags);

    /* Acquire all the other resources and add the host */
    result = usb_stor_acquire_resources(us);
    if (result)
        goto BadDevice;
    usb_autopm_get_interface_no_resume(us->pusb_intf);
    snprintf(us->scsi_name, sizeof(us->scsi_name), "usb-storage %s",
                    dev_name(&us->pusb_intf->dev));
    result = scsi_add_host(us_to_host(us), dev);
    if (result) {
        dev_warn(dev,
                "Unable to add the scsi host\n");
        goto HostAddErr;
    }

    /* Submit the delayed_work for SCSI-device scanning */
    set_bit(US_FLIDX_SCAN_PENDING, &us->dflags);

    if (delay_use > 0)
        dev_dbg(dev, "waiting for device to settle before scanning\n");
    queue_delayed_work(system_freezable_wq, &us->scan_dwork,
            delay_use * HZ);
    return 0;

    /* We come here if there are any problems */
HostAddErr:
    usb_autopm_put_interface_no_suspend(us->pusb_intf);
BadDevice:
    usb_stor_dbg(us, "storage_probe() failed\n");
    release_everything(us);
    return result;
}
EXPORT_SYMBOL_GPL(usb_stor_probe2);
```

#### 2.6.1  变量解析
&ensp;&ensp;&ensp;&ensp;该函数只声明了两个变量，result为结果收集，dev为接口device；
#### 2.6.2 代码检验部分解析
```C
    /* Make sure the transport and protocol have both been set */
    if (!us->transport || !us->proto_handler) {
        result = -ENXIO;
        goto BadDevice;
    }
    usb_stor_dbg(us, "Transport: %s\n", us->transport_name);
    usb_stor_dbg(us, "Protocol: %s\n", us->protocol_name);

    if (us->fflags & US_FL_SCM_MULT_TARG) {
        /*
         * SCM eUSCSI bridge devices can have different numbers
         * of LUNs on different targets; allow all to be probed.
         */
        us->max_lun = 7;
        /* The eUSCSI itself has ID 7, so avoid scanning that */
        us_to_host(us)->this_id = 7;
        /* max_id is 8 initially, so no need to set it here */
    } else {
        /* In the normal case there is only a single target */
        us_to_host(us)->max_id = 1;
        /*
         * Like Windows, we won't store the LUN bits in CDB[1] for
         * SCSI-2 devices using the Bulk-Only transport (even though
         * this violates the SCSI spec).
         */
        if (us->transport == usb_stor_Bulk_transport)
            us_to_host(us)->no_scsi2_lun_in_cdb = 1;
    }

    /* fix for single-lun devices */
    if (us->fflags & US_FL_SINGLE_LUN)
        us->max_lun = 0;
```
&ensp;&ensp;&ensp;&ensp;此部分做了以下几件事：

- 检查us->transport和us->proto_handler是否被赋值，这个在2.5节usb_stor_probe1()函数中的get_transport(us)和get_protocol(us)已经被赋值了；
- 添加调试打印，输出Transport:和Protocol:的结果；
- 根据us->fflags设置us->max_lun的值，像通用的U盘max_lun的值是0；
- 以及其它一些相关的设置，主要是设置scsi host主机控制器的；
- 还有对US_FL_INITIAL_READ10标记的设置；

#### 2.6.3 get_pipes()函数分析
```C
    /* Find the endpoints and calculate pipe values */
    result = get_pipes(us);
    if (result)
        goto BadDevice;
```
&ensp;&ensp;&ensp;&ensp;找到USB mass storage device所在接口上的的端点，并依此计算出pipe值；代码实现如下，以下各小节对其中各部分进行细致分析：
```C
/* Get the pipe settings */
static int get_pipes(struct us_data *us)
{
    struct usb_host_interface *alt = us->pusb_intf->cur_altsetting;
    struct usb_endpoint_descriptor *ep_in;
    struct usb_endpoint_descriptor *ep_out;
    struct usb_endpoint_descriptor *ep_int;
    int res;

    /*
     * Find the first endpoint of each type we need.
     * We are expecting a minimum of 2 endpoints - in and out (bulk).
     * An optional interrupt-in is OK (necessary for CBI protocol).
     * We will ignore any others.
     */
    res = usb_find_common_endpoints(alt, &ep_in, &ep_out, NULL, NULL);
    if (res) {
        usb_stor_dbg(us, "bulk endpoints not found\n");
        return res;
    }

    res = usb_find_int_in_endpoint(alt, &ep_int);
    if (res && us->protocol == USB_PR_CBI) {
        usb_stor_dbg(us, "interrupt endpoint not found\n");
        return res;
    }

    /* Calculate and store the pipe values */
    us->send_ctrl_pipe = usb_sndctrlpipe(us->pusb_dev, 0);
    us->recv_ctrl_pipe = usb_rcvctrlpipe(us->pusb_dev, 0);
    us->send_bulk_pipe = usb_sndbulkpipe(us->pusb_dev,
        usb_endpoint_num(ep_out));
    us->recv_bulk_pipe = usb_rcvbulkpipe(us->pusb_dev,
        usb_endpoint_num(ep_in));
    if (ep_int) {
        us->recv_intr_pipe = usb_rcvintpipe(us->pusb_dev,
            usb_endpoint_num(ep_int));
        us->ep_bInterval = ep_int->bInterval;
    }
    return 0;
}
```

##### 2.6.3.1 变量部分分析
```C
    struct usb_host_interface *alt = us->pusb_intf->cur_altsetting;
    struct usb_endpoint_descriptor *ep_in;
    struct usb_endpoint_descriptor *ep_out;
    struct usb_endpoint_descriptor *ep_int;
    int res;
```
&ensp;&ensp;&ensp;&ensp;alt变量为匹配上的USB接口下的当前接口设定结构体实例，可通过该变量找到端点及端点描述符；ep_in、ep_out、ep_int是端点描述符实例；res为结果收集。

##### 2.6.3.2 usb_find_common_endpoints()函数解析
```C
    /*
     * Find the first endpoint of each type we need.
     * We are expecting a minimum of 2 endpoints - in and out (bulk).
     * An optional interrupt-in is OK (necessary for CBI protocol).
     * We will ignore any others.
     */
    res = usb_find_common_endpoints(alt, &ep_in, &ep_out, NULL, NULL);
    if (res) {
        usb_stor_dbg(us, "bulk endpoints not found\n");
        return res;
    }
```
&ensp;&ensp;&ensp;&ensp;这个函数的目的是找到bulk传输类型的ep_in和ep_out端点，因为USB mass storage主要就是bulk传输（一般只会实现bulk端点）。函数usb_find_common_endpoints()是USB core层实现的通用函数，主要是寻找alt接口下bulk端点和int端点；代码实现如下：
```C
/**
 * usb_find_common_endpoints() -- look up common endpoint descriptors
 * @alt:    alternate setting to search
 * @bulk_in:    pointer to descriptor pointer, or NULL
 * @bulk_out:   pointer to descriptor pointer, or NULL
 * @int_in: pointer to descriptor pointer, or NULL
 * @int_out:    pointer to descriptor pointer, or NULL
 *
 * Search the alternate setting's endpoint descriptors for the first bulk-in,
 * bulk-out, interrupt-in and interrupt-out endpoints and return them in the
 * provided pointers (unless they are NULL).
 *
 * If a requested endpoint is not found, the corresponding pointer is set to
 * NULL.
 *
 * Return: Zero if all requested descriptors were found, or -ENXIO otherwise.
 */
int usb_find_common_endpoints(struct usb_host_interface *alt,
        struct usb_endpoint_descriptor **bulk_in,
        struct usb_endpoint_descriptor **bulk_out,
        struct usb_endpoint_descriptor **int_in,
        struct usb_endpoint_descriptor **int_out)
{
    struct usb_endpoint_descriptor *epd;
    int i;

    if (bulk_in)
        *bulk_in = NULL;
    if (bulk_out)
        *bulk_out = NULL;
    if (int_in)
        *int_in = NULL;
    if (int_out)
        *int_out = NULL;

    for (i = 0; i < alt->desc.bNumEndpoints; ++i) {
        epd = &alt->endpoint[i].desc;

        if (match_endpoint(epd, bulk_in, bulk_out, int_in, int_out))
            return 0;
    }

    return -ENXIO;
}
EXPORT_SYMBOL_GPL(usb_find_common_endpoints);
```
&ensp;&ensp;&ensp;&ensp;此处传入的参数为alt，ep_out，ep_in；对应函数内的alt、bulk_in和bulk_out；继续分析：

- 声明epd端点描述符变量；
- 检查传入的bulk_in、bulk_out、int_in、int_out参数，并做初始化为NULL操作；
- for循环查找alt->desc.bNumEndpoints个端点号，取出对应的&alt->endpoint[i].desc端点描述符结构，赋值给epd，再调用match_endpoint()来匹配所需的端点；
###### 2.6.3.2.1 match_endpoint()函数分析
```C
static bool match_endpoint(struct usb_endpoint_descriptor *epd,
        struct usb_endpoint_descriptor **bulk_in,
        struct usb_endpoint_descriptor **bulk_out,
        struct usb_endpoint_descriptor **int_in,
        struct usb_endpoint_descriptor **int_out)
{
    switch (usb_endpoint_type(epd)) {
    case USB_ENDPOINT_XFER_BULK:
        if (usb_endpoint_dir_in(epd)) {
            if (bulk_in && !*bulk_in) {
                *bulk_in = epd;
                break;
            }
        } else {
            if (bulk_out && !*bulk_out) {
                *bulk_out = epd;
                break;
            } 
        }
        
        return false;
    case USB_ENDPOINT_XFER_INT:
        if (usb_endpoint_dir_in(epd)) {
            if (int_in && !*int_in) {
                *int_in = epd;
                break;
            }
        } else {
            if (int_out && !*int_out) {
                *int_out = epd;
                break;
            }
        }

        return false;
    default:
        return false;
    }

    return (!bulk_in || *bulk_in) && (!bulk_out || *bulk_out) &&
            (!int_in || *int_in) && (!int_out || *int_out);
}
```
&ensp;&ensp;&ensp;&ensp;该函数的功能如下：

- 通过函数usb_endpoint_type(epd)来确认该端点的传输类型，此处只做两种传输类型的匹配：USB_ENDPOINT_XFER_BULK和USB_ENDPOINT_XFER_INT，usb_endpoint_type()函数及各宏定义的实现如下：
```C
#define USB_ENDPOINT_XFERTYPE_MASK  0x03    /* in bmAttributes */
#define USB_ENDPOINT_XFER_CONTROL   0
#define USB_ENDPOINT_XFER_ISOC      1
#define USB_ENDPOINT_XFER_BULK      2
#define USB_ENDPOINT_XFER_INT       3

/**
 * usb_endpoint_type - get the endpoint's transfer type
 * @epd: endpoint to be checked
 *  
 * Returns one of USB_ENDPOINT_XFER_{CONTROL, ISOC, BULK, INT} according
 * to @epd's transfer type.
 */
static inline int usb_endpoint_type(const struct usb_endpoint_descriptor *epd)
{
    return epd->bmAttributes & USB_ENDPOINT_XFERTYPE_MASK;
}
```
&ensp;&ensp;&ensp;&ensp;即通过端点描述符下的bmAttributes元素进行匹配。该值也可以通过lsusb命令查看得到；

- bulk传输模式下，再通过usb_endpoint_dir_in(epd)函数确认传输方向是否为IN，函数实现及相关宏定义如下：
```C
#define USB_ENDPOINT_DIR_MASK       0x80
#define USB_DIR_IN          0x80        /* to host */

/**
 * usb_endpoint_dir_in - check if the endpoint has IN direction
 * @epd: endpoint to be checked
 *      
 * Returns true if the endpoint is of type IN, otherwise it returns false.
 */
static inline int usb_endpoint_dir_in(const struct usb_endpoint_descriptor *epd)
{
    return ((epd->bEndpointAddress & USB_ENDPOINT_DIR_MASK) == USB_DIR_IN);
}
```
&ensp;&ensp;&ensp;&ensp;如果是bulk_in，则将该端点epd赋值给*bulk_in变量；否则就给*bulk_out变量；

- int传输模式下，通过usb_endpoint_dir_in(epd)确定epd方向，再赋值给对应的*int_in或*int_out；

- 若符合以上条件，则返回true，否则返回false；

##### 2.6.3.3 usb_find_int_in_endpoint()函数分析
&ensp;&ensp;&ensp;&ensp;函数usb_find_int_in_endpoint()其实就是封装了usb_find_common_endpoints()，只是传给该函数的参数是给int_in，代码实现如下：
```C
    res = usb_find_int_in_endpoint(alt, &ep_int);
    if (res && us->protocol == USB_PR_CBI) {
        usb_stor_dbg(us, "interrupt endpoint not found\n");
        return res;
    }
```
```C
static inline int __must_check
usb_find_int_in_endpoint(struct usb_host_interface *alt,
        struct usb_endpoint_descriptor **int_in)
{
    return usb_find_common_endpoints(alt, NULL, NULL, int_in, NULL);
}
```
&ensp;&ensp;&ensp;&ensp;此部分是给CBI协议类型设备提供的，BOT协议类型设备一般是不实现中断端点的；

##### 2.6.3.4 us赋值部分分析
```C
    /* Calculate and store the pipe values */
    us->send_ctrl_pipe = usb_sndctrlpipe(us->pusb_dev, 0);
    us->recv_ctrl_pipe = usb_rcvctrlpipe(us->pusb_dev, 0);
    us->send_bulk_pipe = usb_sndbulkpipe(us->pusb_dev,
        usb_endpoint_num(ep_out));
    us->recv_bulk_pipe = usb_rcvbulkpipe(us->pusb_dev,
        usb_endpoint_num(ep_in));
    if (ep_int) { 
        us->recv_intr_pipe = usb_rcvintpipe(us->pusb_dev,
            usb_endpoint_num(ep_int)); 
        us->ep_bInterval = ep_int->bInterval;
    }
```
&ensp;&ensp;&ensp;&ensp;代码实现如下：

- 通过usb_sndctrlpipe()计算端点0的发送control pipe，并赋值给us->send_ctrl_pipe。代码实现如下：
```C
#define PIPE_CONTROL            2

static inline unsigned int __create_pipe(struct usb_device *dev,
        unsigned int endpoint) 
{
    return (dev->devnum << 8) | (endpoint << 15);
}

#define usb_sndctrlpipe(dev, endpoint)  \
    ((PIPE_CONTROL << 30) | __create_pipe(dev, endpoint))
```

- 通过usb_rcvctrlpipe()计算端点0的接收control pipe，并赋值给us->recv_ctrl_pipe。代码实现如下：
```C
#define usb_rcvctrlpipe(dev, endpoint)  \
    ((PIPE_CONTROL << 30) | __create_pipe(dev, endpoint) | USB_DIR_IN)
```

- 通过usb_sndbulkpipe()计算发送bulk pipe，并赋值给us->send_bulk_pipe。代码实现如下：
```C
#define PIPE_BULK           3
#define USB_ENDPOINT_NUMBER_MASK    0x0f    /* in bEndpointAddress */

/** 
 * usb_endpoint_num - get the endpoint's number
 * @epd: endpoint to be checked
 *
 * Returns @epd's number: 0 to 15.
 */
static inline int usb_endpoint_num(const struct usb_endpoint_descriptor *epd)
{
    return epd->bEndpointAddress & USB_ENDPOINT_NUMBER_MASK;
}

#define usb_sndbulkpipe(dev, endpoint)  \
    ((PIPE_BULK << 30) | __create_pipe(dev, endpoint))
```
&ensp;&ensp;&ensp;&ensp;通过ep_out和ep_in端点描述符的epd->bEndpointAddress获取到端点号，再计算出发送/接收bulk pipe值；

- 通过usb_rcvbulkpipe()计算接收bulk  pipe，并赋值给us->recv_bulk_pipe。代码实现如下：
```C
#define usb_rcvbulkpipe(dev, endpoint)  \
    ((PIPE_BULK << 30) | __create_pipe(dev, endpoint) | USB_DIR_IN)
```

- 如果有中断端点ep_int的话（此部分USB mass storage不会走），则通过usb_rcvintpipe()获取接收int pipe，并赋值给us->recv_intr_pipe，并设置us->ep_bInterval为端点中的ep_int->bInterval，usb_rcvintpipe()代码实现如下：
```C
#define usb_rcvintpipe(dev, endpoint)   \
    ((PIPE_INTERRUPT << 30) | __create_pipe(dev, endpoint) | USB_DIR_IN)
```

#### 2.6.4 usb_stor_acquire_resources()函数解析
&ensp;&ensp;&ensp;&ensp;该函数是USB storage驱动中的重头戏。里面实现了一个内核线程usb_stor_control_thread，就是该线程在操作数据的传输。代码实现如下：
```C
/* Initialize all the dynamic resources we need */
static int usb_stor_acquire_resources(struct us_data *us)
{
    int p;
    struct task_struct *th;

    us->current_urb = usb_alloc_urb(0, GFP_KERNEL);
    if (!us->current_urb)
        return -ENOMEM;
    
    /*
     * Just before we start our control thread, initialize
     * the device if it needs initialization
     */
    if (us->unusual_dev->initFunction) {
        p = us->unusual_dev->initFunction(us);
        if (p)
            return p;
    }

    /* Start up our control thread */
    th = kthread_run(usb_stor_control_thread, us, "usb-storage");
    if (IS_ERR(th)) {
        dev_warn(&us->pusb_intf->dev,
                "Unable to start control thread\n");
        return PTR_ERR(th);
    }
    us->ctl_thread = th;

    return 0;
}
```
&ensp;&ensp;&ensp;&ensp;代码逻辑如下：

- 首先利用usb_alloc_urb()申请一个urb：us->current_urb，usb_alloc_urb()是USB core层核心函数，此处不介绍；
- 判断是否实现了us->unusual_dev->initFunction，即unusual dev的私有函数，实现了的话就在此处调用；
- 创建一个内核线程usb_stor_control_thread，取名“usb-storage”，并将该线程号赋值给us->ctl_thread；后面再详细讲解该线程实现。

#### 2.6.5 usb_autopm_get_interface_no_resume()介绍
&ensp;&ensp;&ensp;&ensp;增加USB接口电源管理计数，代码实现如下：
```C
/**
 * usb_autopm_get_interface_no_resume - increment a USB interface's PM-usage counter
 * @intf: the usb_interface whose counter should be incremented
 *      
 * This routine increments @intf's usage counter but does not carry out an
 * autoresume.
 *  
 * This routine can run in atomic context.
 */
void usb_autopm_get_interface_no_resume(struct usb_interface *intf)
{
    struct usb_device   *udev = interface_to_usbdev(intf);
    
    usb_mark_last_busy(udev);
    atomic_inc(&intf->pm_usage_cnt);
    pm_runtime_get_noresume(&intf->dev);
}
EXPORT_SYMBOL_GPL(usb_autopm_get_interface_no_resume);
```

#### 2.6.6 scsi_add_host()介绍
&ensp;&ensp;&ensp;&ensp;相对于scsi控制器host的注册及使用，此处简单介绍一下。前面使用scsi_host_alloc()函数，它的作用就是给struct Scsi_Host结构体申请了空间，而真正要想模拟一个scsi的情景，需要三个函数：scsi_host_alloc()、scsi_add_host()、scsi_scan_host()；只有调用了第二个函数之后，scsi核心层才知道有这么一个host的存在，而在第三个函数被调用之后，真正的设备才被发现。

&ensp;&ensp;&ensp;&ensp;所以，此处调用scsi_add_host()，是将USB storage所在的虚拟host通报给SCSI核心层，代码实现如下：
```C
    result = scsi_add_host(us_to_host(us), dev);
    if (result) {
        dev_warn(dev,
                "Unable to add the scsi host\n");
        goto HostAddErr;
    }
```

#### 2.6.7 queue_delayed_work()调用分析
```C
    /* Submit the delayed_work for SCSI-device scanning */
    set_bit(US_FLIDX_SCAN_PENDING, &us->dflags);

    if (delay_use > 0)
        dev_dbg(dev, "waiting for device to settle before scanning\n");
    queue_delayed_work(system_freezable_wq, &us->scan_dwork,
            delay_use * HZ);
```
&ensp;&ensp;&ensp;&ensp;这里需要分两个部分来解释，一个是us->dflags标记，一个是us->scan_dwork；下面分小节来讲解。

##### 2.6.7.1 us->dflags标记作用
&ensp;&ensp;&ensp;&ensp;经过分析，该标记主要是用来检测scsi控制器的状态之用，其在struct us_data结构体内的解释如下：

```C
    unsigned long       dflags;      /* dynamic atomic bitflags */
```
&ensp;&ensp;&ensp;&ensp;此处是在scsi_scan_host()函数调用前设置，标志马上进入scsi scan状态，该标志还在内核线程中做scsi cmd timeout状态检测，以及在scsi abort函数实现中做设置scsi cmd timeout状态等；稍后就会分析到；

##### 2.6.7.2 queue_delayed_work()调用
&ensp;&ensp;&ensp;&ensp;前面在2.5.3小节已经介绍在函数usb_stor_probe1()中已调用INIT_DELAYED_WORK()初始化了struct delayed_work us->scan_dwork，并赋予了执行函数：usb_stor_scan_dwork()，此处即将该延迟工作加入到的工作队列system_freezable_wq中；

**==【注1】==**

---
&ensp;&ensp;&ensp;&ensp;以个人理解来分析一下system_freezable_wq工作队列，代码实现如下(本USB storage代码树中没有该函数实现副本，请参照主线内核)：
```C
struct workqueue_struct *system_freezable_wq __read_mostly;
EXPORT_SYMBOL_GPL(system_freezable_wq);

    system_freezable_wq = alloc_workqueue("events_freezable",
                          WQ_FREEZABLE, 0);
```
&ensp;&ensp;&ensp;&ensp;WQ_FREEZABLE是一个和电源管理相关的内容。在系统Hibernation或者suspend的时候，有一个步骤就是冻结用户空间的进程以及部分（标注freezable的）内核线程（包括workqueue的worker thread）。标记WQ_FREEZABLE的workqueue需要参与到进程冻结的过程中，worker thread被冻结的时候，会处理完当前所有的work，一旦冻结完成，那么就不会启动新的work的执行，直到进程被解冻[^sample_1]。

&ensp;&ensp;&ensp;&ensp;所以，当内核进入suspend状态时，会冻结该workqueue，导致，插拔U盘无反应？

---

&ensp;&ensp;&ensp;&ensp;此处的delay_use是和之前的quirks一样的，通过在sys文件系统下创建接口，可供用户层修改，默认值为1，代码实现及对应接口路径如下：
```C
static unsigned int delay_use = 1;
module_param(delay_use, uint, S_IRUGO | S_IWUSR);
MODULE_PARM_DESC(delay_use, "seconds to delay before using a new device");
```
```C
~$ cat /sys/module/usb_storage/parameters/delay_use 
1
```

##### 2.6.7.3 usb_stor_scan_dwork()函数实现
&ensp;&ensp;&ensp;&ensp;此刻，我们该进入之前INIT_DELAYED_WORK(&us->scan_dwork, usb_stor_scan_dwork)中的延迟工作函数usb_stor_scan_dwork()分析了，代码如下：
```C
/* Delayed-work routine to carry out SCSI-device scanning */
static void usb_stor_scan_dwork(struct work_struct *work)
{
    struct us_data *us = container_of(work, struct us_data,
            scan_dwork.work);
    struct device *dev = &us->pusb_intf->dev;

    dev_dbg(dev, "starting scan\n");
    
    /* For bulk-only devices, determine the max LUN value */
    if (us->protocol == USB_PR_BULK &&
        !(us->fflags & US_FL_SINGLE_LUN) &&
        !(us->fflags & US_FL_SCM_MULT_TARG)) {
        mutex_lock(&us->dev_mutex);
        us->max_lun = usb_stor_Bulk_max_lun(us);
        /*
         * Allow proper scanning of devices that present more than 8 LUNs
         * While not affecting other devices that may need the previous
         * behavior
         */
        if (us->max_lun >= 8)
            us_to_host(us)->max_lun = us->max_lun+1;
        mutex_unlock(&us->dev_mutex);
    }
    scsi_scan_host(us_to_host(us));
    dev_dbg(dev, "scan complete\n");

    /* Should we unbind if no devices were detected? */

    usb_autopm_put_interface(us->pusb_intf);
    clear_bit(US_FLIDX_SCAN_PENDING, &us->dflags);
}
```
&ensp;&ensp;&ensp;&ensp;该函数的主要功能有两个：

- 一个是获取host的max_lun，该值是从USB mass storage device中获得，由硬件提供，此处实现了一套通过control pipe向device发送control命令的流程；
- 另一个是通过scsi_scan_host()，激活host主机控制器，完成struct Scsi_Host的注册及使用；
&ensp;&ensp;&ensp;&ensp;下面再分解逐一讲解。

###### 2.6.7.3.1 usb_stor_Bulk_max_lun()函数实现
&ensp;&ensp;&ensp;&ensp;对于bulk-only devices，需要向device获取max LUN值，该部分实现如下：
```C
    struct us_data *us = container_of(work, struct us_data,
            scan_dwork.work);
    struct device *dev = &us->pusb_intf->dev;

    /* For bulk-only devices, determine the max LUN value */
    if (us->protocol == USB_PR_BULK &&
        !(us->fflags & US_FL_SINGLE_LUN) &&
        !(us->fflags & US_FL_SCM_MULT_TARG)) {
        mutex_lock(&us->dev_mutex);
        us->max_lun = usb_stor_Bulk_max_lun(us);
        /*
         * Allow proper scanning of devices that present more than 8 LUNs
         * While not affecting other devices that may need the previous
         * behavior
         */
        if (us->max_lun >= 8)
            us_to_host(us)->max_lun = us->max_lun+1;
        mutex_unlock(&us->dev_mutex);
    }
```
&ensp;&ensp;&ensp;&ensp;已针对USB_PR_BULK协议以及US_FL_SINGLE_LUN和US_FL_SCM_MULT_TARG标记做了限制，通过调用usb_stor_Bulk_max_lun()函数来获取max lun，函数实现如下：
```C
/* Determine what the maximum LUN supported is */
int usb_stor_Bulk_max_lun(struct us_data *us)
{
    int result;
    
    /* issue the command */
    us->iobuf[0] = 0;
    result = usb_stor_control_msg(us, us->recv_ctrl_pipe,
                 US_BULK_GET_MAX_LUN, 
                 USB_DIR_IN | USB_TYPE_CLASS |
                 USB_RECIP_INTERFACE,
                 0, us->ifnum, us->iobuf, 1, 10*HZ);

    usb_stor_dbg(us, "GetMaxLUN command result is %d, data is %d\n",
             result, us->iobuf[0]);
            
    /*
     * If we have a successful request, return the result if valid. The
     * CBW LUN field is 4 bits wide, so the value reported by the device
     * should fit into that.
     */
    if (result > 0) {
        if (us->iobuf[0] < 16) {
            return us->iobuf[0];
        } else {
            dev_info(&us->pusb_intf->dev,
                 "Max LUN %d is not valid, using 0 instead",
                 us->iobuf[0]);
        }
    }

    /*
     * Some devices don't like GetMaxLUN.  They may STALL the control
     * pipe, they may return a zero-length result, they may do nothing at
     * all and timeout, or they may fail in even more bizarrely creative
     * ways.  In these cases the best approach is to use the default
     * value: only one LUN.
     */
    return 0;
}
```
&ensp;&ensp;&ensp;&ensp;通过调用usb_stor_control_msg函数下发控制命令US_BULK_GET_MAX_LUN，该函数的参数需要分析一下，参见本github中的手册文档USB_Storage_spec.md[^sample_2]中对US_BULK_GET_MAX_LUN命令的说明，即下表命令规则：

bmRequestType|bRequest|wValue|wIndex|wLength|Data
--|--|--|--|--|--
10100001b|11111110b|0000h|Interface|0001h|1 byte

&ensp;&ensp;&ensp;&ensp;而函数参数对应关系如下：
![](images/US_BULK_GET_MAX_LUN_cmd_parameter1.png)

&ensp;&ensp;&ensp;&ensp;显然，此处是完全按照硬件协议规定来定义的，usb_stor_control_msg()函数实现如下：
```C
/*
 * Transfer one control message, with timeouts, and allowing early
 * termination.  Return codes are usual -Exxx, *not* USB_STOR_XFER_xxx.
 */ 
int usb_stor_control_msg(struct us_data *us, unsigned int pipe,
         u8 request, u8 requesttype, u16 value, u16 index,
         void *data, u16 size, int timeout)
{
    int status;

    usb_stor_dbg(us, "rq=%02x rqtype=%02x value=%04x index=%02x len=%u\n",
             request, requesttype, value, index, size);

    /* fill in the devrequest structure */
    us->cr->bRequestType = requesttype;
    us->cr->bRequest = request;
    us->cr->wValue = cpu_to_le16(value);
    us->cr->wIndex = cpu_to_le16(index);
    us->cr->wLength = cpu_to_le16(size);

    /* fill and submit the URB */
    usb_fill_control_urb(us->current_urb, us->pusb_dev, pipe,
             (unsigned char*) us->cr, data, size,
             usb_stor_blocking_completion, NULL);
    status = usb_stor_msg_common(us, timeout);

    /* return the actual length of the data transferred if no error */
    if (status == 0)
        status = us->current_urb->actual_length;
    return status;
}
EXPORT_SYMBOL_GPL(usb_stor_control_msg);
```
&ensp;&ensp;&ensp;&ensp;us->cr在之前已经申请了内存空间，此处直接填充该结构，接着调用USB core层函数usb_fill_control_urb()来填充us->current_urb结构，并将urb->complete赋值为usb_stor_blocking_completion()，再调用usb_stor_msg_common()函数进一步处理并下发给USB HCD，usb_stor_blocking_completion()和usb_stor_msg_common()代码实现如下：

---
```C
/*
 * This is the completion handler which will wake us up when an URB
 * completes.
 */ 
static void usb_stor_blocking_completion(struct urb *urb)
{
    struct completion *urb_done_ptr = urb->context;
    
    complete(urb_done_ptr);
}
```
```C
/*
 * This is the common part of the URB message submission code
 *
 * All URBs from the usb-storage driver involved in handling a queued scsi
 * command _must_ pass through this function (or something like it) for the
 * abort mechanisms to work properly. 
 */
static int usb_stor_msg_common(struct us_data *us, int timeout)
{
    struct completion urb_done;
    long timeleft;
    int status;
    
    /* don't submit URBs during abort processing */
    if (test_bit(US_FLIDX_ABORTING, &us->dflags))
        return -EIO;

    /* set up data structures for the wakeup system */
    init_completion(&urb_done);

    /* fill the common fields in the URB */
    us->current_urb->context = &urb_done;
    us->current_urb->transfer_flags = 0;

    /*
     * we assume that if transfer_buffer isn't us->iobuf then it
     * hasn't been mapped for DMA.  Yes, this is clunky, but it's
     * easier than always having the caller tell us whether the
     * transfer buffer has already been mapped.
     */
    if (us->current_urb->transfer_buffer == us->iobuf)
        us->current_urb->transfer_flags |= URB_NO_TRANSFER_DMA_MAP;
    us->current_urb->transfer_dma = us->iobuf_dma;

    /* submit the URB */
    status = usb_submit_urb(us->current_urb, GFP_NOIO);
    if (status) {
        /* something went wrong */
        return status;
    }

    /*
     * since the URB has been submitted successfully, it's now okay
     * to cancel it
     */
    set_bit(US_FLIDX_URB_ACTIVE, &us->dflags);

    /* did an abort occur during the submission? */
    if (test_bit(US_FLIDX_ABORTING, &us->dflags)) {

        /* cancel the URB, if it hasn't been cancelled already */
        if (test_and_clear_bit(US_FLIDX_URB_ACTIVE, &us->dflags)) {
            usb_stor_dbg(us, "-- cancelling URB\n");
            usb_unlink_urb(us->current_urb);
        }
    }

    /* wait for the completion of the URB */
    timeleft = wait_for_completion_interruptible_timeout(
            &urb_done, timeout ? : MAX_SCHEDULE_TIMEOUT);

    clear_bit(US_FLIDX_URB_ACTIVE, &us->dflags);

    if (timeleft <= 0) {
        usb_stor_dbg(us, "%s -- cancelling URB\n",
                 timeleft == 0 ? "Timeout" : "Signal");
        usb_kill_urb(us->current_urb);
    }

    /* return the URB status */
    return us->current_urb->status;
}
```
&ensp;&ensp;&ensp;&ensp;该函数工作流程如下：

- 先设置us->dflags成US_FLIDX_ABORTING，以便处理urb；
- 初始化完成量urb_done，并将该完成量赋值给us->current_urb->context，这在usb_stor_blocking_completion()中会调用complete()完成urb处理；
- 接着是对传输buffer的设定；
- 调用usb_submit_urb()上报urb给USB主机控制器；
- 设置us->dflags为US_FLIDX_URB_ACTIVE，以及检测us->dflags；
- 接着便是通过wait_for_completion_interruptible_timeout()等待urb_done被处理，或者超时；
- 清除us->dflags位，表示urb已结束；
- 检测timeleft确认urb是否下发有效，超时的话则调用usb_kill_urb(us->current_urb)清除urb；
- 返回urb传输状态；

---
&ensp;&ensp;&ensp;&ensp;usb_stor_control_msg()函数接着检查usb_stor_msg_common()函数返回值，即status，如果无错误，则返回的status被赋值为us->current_urb->actual_length；

---
&ensp;&ensp;&ensp;&ensp;回到usb_stor_Bulk_max_lun()函数，再对usb_stor_control_msg()返回值result进行检查设置：

- 如果命令下发成功，则device将返回一个1byte数据，只是LUNs值，所以返回了us->iobuf[0]值；
- 而有些device则可能会对该命令STALL，则直接设置为0；
- 简而言之，这个max LUN对通用USB mass storage device来说没啥意义，尽管如此，这也算是一次完整的host和device通信示例；

###### 2.6.7.3.2 scsi_scan_host()函数介绍
&ensp;&ensp;&ensp;&ensp;Scsi_Host主机控制器最后调用函数：扫描scsi主机控制器；至此USB模拟scsi控制器完成，U盘即可被应用层发现。应用层可通过scsi接口进行查看USB mass storage的状态。

---
&ensp;&ensp;&ensp;&ensp;函数最后两行：
```C
    usb_autopm_put_interface(us->pusb_intf);
    clear_bit(US_FLIDX_SCAN_PENDING, &us->dflags);
```

- USB PM电源管理相关代码：usb_autopm_put_interface()，即此处准许USB接口设备autosuspend了；
- 清除us->dflags的US_FLIDX_SCAN_PENDING标记，毕竟scan已经结束。


### 2.7 内核线程usb_stor_control_thread()数据传输解析
&ensp;&ensp;&ensp;&ensp;终于进入USB storage传输的主体，也是USB storage最重要的一部分。现看函数实现：
```C
static int usb_stor_control_thread(void * __us)
{
    struct us_data *us = (struct us_data *)__us;
    struct Scsi_Host *host = us_to_host(us);
    struct scsi_cmnd *srb;
        
    for (;;) {
        usb_stor_dbg(us, "*** thread sleeping\n");
        if (wait_for_completion_interruptible(&us->cmnd_ready))
            break;

        usb_stor_dbg(us, "*** thread awakened\n");

        /* lock the device pointers */
        mutex_lock(&(us->dev_mutex));

        /* lock access to the state */
        scsi_lock(host);

        /* When we are called with no command pending, we're done */
        srb = us->srb;
        if (srb == NULL) {
            scsi_unlock(host);
            mutex_unlock(&us->dev_mutex);
            usb_stor_dbg(us, "-- exiting\n");
            break;
        }

        /* has the command timed out *already* ? */
        if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
            srb->result = DID_ABORT << 16;
            goto SkipForAbort;
        }

        scsi_unlock(host);

        /*
         * reject the command if the direction indicator
         * is UNKNOWN
         */
        if (srb->sc_data_direction == DMA_BIDIRECTIONAL) {
            usb_stor_dbg(us, "UNKNOWN data direction\n");
            srb->result = DID_ERROR << 16;
        }

        /*
         * reject if target != 0 or if LUN is higher than
         * the maximum known LUN
         */
        else if (srb->device->id &&
                !(us->fflags & US_FL_SCM_MULT_TARG)) {
            usb_stor_dbg(us, "Bad target number (%d:%llu)\n",
                       srb->device->id,
                     srb->device->lun);
            srb->result = DID_BAD_TARGET << 16;
        }

        else if (srb->device->lun > us->max_lun) {
            usb_stor_dbg(us, "Bad LUN (%d:%llu)\n",
                     srb->device->id,
                     srb->device->lun);
            srb->result = DID_BAD_TARGET << 16;
        }

        /*
         * Handle those devices which need us to fake
         * their inquiry data
         */
        else if ((srb->cmnd[0] == INQUIRY) &&
                (us->fflags & US_FL_FIX_INQUIRY)) {
            unsigned char data_ptr[36] = {
                0x00, 0x80, 0x02, 0x02,
                0x1F, 0x00, 0x00, 0x00};

            usb_stor_dbg(us, "Faking INQUIRY command\n");
            fill_inquiry_response(us, data_ptr, 36);
            srb->result = SAM_STAT_GOOD;
        }

        /* we've got a command, let's do it! */
        else {
            US_DEBUG(usb_stor_show_command(us, srb));
            us->proto_handler(srb, us);
            usb_mark_last_busy(us->pusb_dev);
        }

        /* lock access to the state */
        scsi_lock(host);

        /* was the command aborted? */
        if (srb->result == DID_ABORT << 16) {
SkipForAbort:
            usb_stor_dbg(us, "scsi command aborted\n");
            srb = NULL; /* Don't call srb->scsi_done() */
        }

        /*
         * If an abort request was received we need to signal that
         * the abort has finished.  The proper test for this is
         * the TIMED_OUT flag, not srb->result == DID_ABORT, because
         * the timeout might have occurred after the command had
         * already completed with a different result code.
         */
        if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
            complete(&(us->notify));

            /* Allow USB transfers to resume */
            clear_bit(US_FLIDX_ABORTING, &us->dflags);
            clear_bit(US_FLIDX_TIMED_OUT, &us->dflags);
        }

        /* finished working on this command */
        us->srb = NULL;
        scsi_unlock(host);

        /* unlock the device pointers */
        mutex_unlock(&us->dev_mutex);

        /* now that the locks are released, notify the SCSI core */
        if (srb) {
            usb_stor_dbg(us, "scsi cmd done, result=0x%x\n",
                    srb->result);
            srb->scsi_done(srb);
        }
    } /* for (;;) */

    /* Wait until we are told to stop */
    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE);
        if (kthread_should_stop())
            break;
        schedule();
    }
    __set_current_state(TASK_RUNNING);
    return 0;
}
```
&ensp;&ensp;&ensp;&ensp;直接进第一个for循环，第一部分，us->cmnd_ready完成量的调用，这个用于同步功能的完成量。

#### 2.7.1  us->cmnd_ready解析
us->cmnd_ready在本驱动中主要用于该内核线程和scsi中间层命令下发的同步，此处实现为：
```C
        if (wait_for_completion_interruptible(&us->cmnd_ready))
            break;
```
&ensp;&ensp;&ensp;&ensp;调用complete(&us->cmnd_ready)的地方有两处，一个是在释放内核资源时调用，此处不解释；另一处则是在drivers/usb/storage/scsiglue.c中struct scsi_host_template usb_stor_host_template结构体实例中的queuecommand()函数中，即scsi中间层下发scsi cmd给USB storage驱动；

---

queuecommand()实现如下：
```C
/* queue a command */
/* This is always called with scsi_lock(host) held */
static int queuecommand_lck(struct scsi_cmnd *srb,
            void (*done)(struct scsi_cmnd *))
{
    struct us_data *us = host_to_us(srb->device->host);

    /* check for state-transition errors */
    if (us->srb != NULL) {
        printk(KERN_ERR USB_STORAGE "Error in %s: us->srb = %p\n",
            __func__, us->srb);
        return SCSI_MLQUEUE_HOST_BUSY;
    }

    /* fail the command if we are disconnecting */
    if (test_bit(US_FLIDX_DISCONNECTING, &us->dflags)) {
        usb_stor_dbg(us, "Fail command during disconnect\n");
        srb->result = DID_NO_CONNECT << 16;
        done(srb);
        return 0;
    }

    /* enqueue the command and wake up the control thread */
    srb->scsi_done = done;
    us->srb = srb;
    complete(&us->cmnd_ready);

    return 0;
}

static DEF_SCSI_QCMD(queuecommand)
```
&ensp;&ensp;&ensp;&ensp;srb为scsi cmd的current srb，为struct scsi_cmnd结构体；该函数流程如下：

- 检查之前的us->srb是否有效；
- 检测us->dflags的US_FLIDX_DISCONNECTING标记；
- 赋值；
- 调用complete(&us->cmnd_ready)，实现与内核线程usb_stor_control_thread()同步；

---

#### 2.7.2 srb及us->dflags初步判断
&ensp;&ensp;&ensp;&ensp;继续usb_stor_control_thread()函数；接下来对us->srb和us->dflags的US_FLIDX_TIMED_OUT做检测判断，此处使用了自旋锁及scsi_lock/unlock()，代码实现如下：
```C
        /* lock the device pointers */
        mutex_lock(&(us->dev_mutex));

        /* lock access to the state */
        scsi_lock(host);

        /* When we are called with no command pending, we're done */
        srb = us->srb;
        if (srb == NULL) {
            scsi_unlock(host);
            mutex_unlock(&us->dev_mutex);
            usb_stor_dbg(us, "-- exiting\n");
            break;
        }

        /* has the command timed out *already* ? */
        if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
            srb->result = DID_ABORT << 16;
            goto SkipForAbort;
        }

        scsi_unlock(host);
```

#### 2.7.3 对srb结构进一步检测
&ensp;&ensp;&ensp;&ensp;接下来将对由queuecommand()赋值的srb进行细致的检测，代码如下：
```C
        /*
         * reject the command if the direction indicator
         * is UNKNOWN
         */
        if (srb->sc_data_direction == DMA_BIDIRECTIONAL) {
            usb_stor_dbg(us, "UNKNOWN data direction\n");
            srb->result = DID_ERROR << 16;
        }

        /*
         * reject if target != 0 or if LUN is higher than
         * the maximum known LUN
         */
        else if (srb->device->id &&
                !(us->fflags & US_FL_SCM_MULT_TARG)) {
            usb_stor_dbg(us, "Bad target number (%d:%llu)\n",
                     srb->device->id,
                     srb->device->lun);
            srb->result = DID_BAD_TARGET << 16;
        }

        else if (srb->device->lun > us->max_lun) {
            usb_stor_dbg(us, "Bad LUN (%d:%llu)\n",
                     srb->device->id,
                     srb->device->lun);
            srb->result = DID_BAD_TARGET << 16;
        }

        /*
         * Handle those devices which need us to fake
         * their inquiry data
         */
        else if ((srb->cmnd[0] == INQUIRY) &&
                (us->fflags & US_FL_FIX_INQUIRY)) {
            unsigned char data_ptr[36] = {
                0x00, 0x80, 0x02, 0x02,
                0x1F, 0x00, 0x00, 0x00};

            usb_stor_dbg(us, "Faking INQUIRY command\n");
            fill_inquiry_response(us, data_ptr, 36);
            srb->result = SAM_STAT_GOOD;
        }

        /* we've got a command, let's do it! */
        else {
            US_DEBUG(usb_stor_show_command(us, srb));
            us->proto_handler(srb, us);
            usb_mark_last_busy(us->pusb_dev);
        }
```
&ensp;&ensp;&ensp;&ensp;代码流程如下：

- 检查数据传输方向srb->sc_data_direction，若未指定，则表示错误，赋值错误码；
- 检查硬件没有设置US_FL_SCM_MULT_TARG标记，但是获得的srb->device->id却大于0的情况；
- 检查srb->device->lun指定的值大于从硬件获取的max_lun值的情况；
- 硬件不支持US_FL_FIX_INQUIRY标记，即SCSI INQUIRY命令的情况，这部分在2.7.3.1小节介绍；
- 所有的判断都没问题的话，则可调用us->proto_handler(srb, us)了，这部分在2.7.4节介绍；

#####  2.7.3.1 USB storage device不支持INQUIRY情况分析
&ensp;&ensp;&ensp;&ensp;先看代码实现：
```C
        /*
         * Handle those devices which need us to fake
         * their inquiry data
         */
        else if ((srb->cmnd[0] == INQUIRY) &&
                (us->fflags & US_FL_FIX_INQUIRY)) {
            unsigned char data_ptr[36] = {
                0x00, 0x80, 0x02, 0x02,
                0x1F, 0x00, 0x00, 0x00};

            usb_stor_dbg(us, "Faking INQUIRY command\n");
            fill_inquiry_response(us, data_ptr, 36);
            srb->result = SAM_STAT_GOOD;
        }
```
&ensp;&ensp;&ensp;&ensp;scsi 子系统里的设备使用 scsi 命令来通信,scsi spec 定义了一大堆的命令,spec 里称这个为命令集,即所谓的 command set.其中一些命令是每一个 scsi 设备都必须支持的,另一些命令则是可选的.而作为 U盘,它所支持的是 scsi transparent command set,所以它基本上就是支持所有的 scsi 命令了,不过我们其实并不关心任何一个具体的命令,只需要了解一些最基本的命令就是了.比如我们需要知道,所有的 scsi设备都至少需要支持以下这四个 scsi 命令:INQUIRY,REQUEST SENSE,SEND DIAGNOSTIC,TEST UNIT READY。

&ensp;&ensp;&ensp;&ensp;事实上，INQUIRY命令是最最基本的一个SCSI命令，比如主机第一次探测device的时候就要用INQUIRY命令来了解这是一个什么设备。通常大多数设备的vendor name 和 product name 是通过 INQUIRY 命令来获得的,而这个 us->fflag 表明,这些设备的 vendor name 和 product name 不需要查询,或者根本就不支持查询,她们的 vendor name 和 product name直接就定义好了,在 unusal_devs.h 中就设好了。

&ensp;&ensp;&ensp;&ensp;此处的srb->cmnd[0]包含的就是一个scsi命令，这个判断的意思是，如果device不支持scsi的INQUIRY命令，则USB storage直接在此处封装一个INQUIRY命令的返回结果给scsi中间层，这样就做到两边都满意了。先继续看fill_inquiry_response()函数：
```C
/*
 * fill_inquiry_response takes an unsigned char array (which must
 * be at least 36 characters) and populates the vendor name,
 * product name, and revision fields. Then the array is copied
 * into the SCSI command's response buffer (oddly enough
 * called request_buffer). data_len contains the length of the
 * data array, which again must be at least 36.
 */

void fill_inquiry_response(struct us_data *us, unsigned char *data,
        unsigned int data_len)
{
    if (data_len < 36) /* You lose. */
        return;

    memset(data+8, ' ', 28);
    if (data[0]&0x20) { /*
                 * USB device currently not connected. Return
                 * peripheral qualifier 001b ("...however, the
                 * physical device is not currently connected
                 * to this logical unit") and leave vendor and
                 * product identification empty. ("If the target
                 * does store some of the INQUIRY data on the
                 * device, it may return zeros or ASCII spaces
                 * (20h) in those fields until the data is
                 * available from the device.").
                 */
    } else {
        u16 bcdDevice = le16_to_cpu(us->pusb_dev->descriptor.bcdDevice);
        int n;

        n = strlen(us->unusual_dev->vendorName);
        memcpy(data+8, us->unusual_dev->vendorName, min(8, n));
        n = strlen(us->unusual_dev->productName);
        memcpy(data+16, us->unusual_dev->productName, min(16, n));

        data[32] = 0x30 + ((bcdDevice>>12) & 0x0F);
        data[33] = 0x30 + ((bcdDevice>>8) & 0x0F);
        data[34] = 0x30 + ((bcdDevice>>4) & 0x0F);
        data[35] = 0x30 + ((bcdDevice) & 0x0F);
    }

    usb_stor_set_xfer_buf(data, data_len, us->srb);
}
EXPORT_SYMBOL_GPL(fill_inquiry_response);
```
&ensp;&ensp;&ensp;&ensp;每个SCSI命令，在device应答时，都是依据SCSI协议里规定的格式。所以相对INQUIRY命令，规范中规定，响应数据必须至少包含36各字节，所以函数开头就直接检测这个data_len必须至少36字节。

######  2.7.3.1.1 sg_inq工具查询INQUIRY命令
&ensp;&ensp;&ensp;&ensp;此处推荐一个可以查询SCSI命令的工具，安装包sg3-utils，然后就可以看到一个sg_inq命令了，其它sg_*没有研究过，此处只介绍下sg_inq命令，插入U盘，运行命令：
```C
$ sudo sg_inq /dev/sdb
standard INQUIRY:
  PQual=0  Device_type=0  RMB=1  LU_CONG=0  version=0x06  [SPC-4]
  [AERC=0]  [TrmTsk=0]  NormACA=0  HiSUP=0  Resp_data_format=2
  SCCS=0  ACC=0  TPGS=0  3PC=0  Protect=0  [BQue=0]
  EncServ=0  MultiP=0  [MChngr=0]  [ACKREQQ=0]  Addr16=0
  [RelAdr=0]  WBus16=0  Sync=0  [Linked=0]  [TranDis=0]  CmdQue=0
    length=36 (0x24)   Peripheral device type: disk
 Vendor identification: Kingston
 Product identification: DataTraveler 2.0
 Product revision level: PMAP
 Unit serial number: CB806100040A 
```
&ensp;&ensp;&ensp;&ensp;这里面看得懂的信息并不多，但是length=36，显示了命令响应的长度，以及Vendor ID，Product ID，Product revision等；

######  2.7.3.1.2 fill_inquiry_response()函数解析 
==【data数据说明及填充】==

&ensp;&ensp;&ensp;&ensp;继续fill_inquiry_response()，接下来的两句：
```C
memset(data+8, ' ', 28);
    if (data[0]&0x20) {
```
&ensp;&ensp;&ensp;&ensp;SCSI协议规定了，标准的INQUIRY命令的data[0]，总共8个bit位，其中bit[7:5]被称为peripheral qualifier，bit[4:0]被称为perpheral device type；此处的0x20就表示peripheral qualifier 这个外围设备限定符为 001b，而 peripheral device type这个外围设备类型则为 00h。查询SCSI协议可知，后者代表的是设备类型为磁盘，或者说直接访问设备；前者代表的是目标设备的当前LUN支持这种类型，然而，实际的物理设备并没有连接在当前LUN上。

&ensp;&ensp;&ensp;&ensp;在data[36]中，从data[8]一直到data[35]这28个字节都是保存的vendor和product的信息，SCSI协议里面写了，如果设备里有保存这些信息，那么它可以暂时先返回0x20h，所以可以先把data[8]到data[35]都给设置成空，等到保存在设备上的这些信息可以读了再去读[^sample_3]。

&ensp;&ensp;&ensp;&ensp;此处fill_inquiry_response()传入的data的data[0]不是0x20，所以继续分析：
```C
        u16 bcdDevice = le16_to_cpu(us->pusb_dev->descriptor.bcdDevice);
        int n;

        n = strlen(us->unusual_dev->vendorName);
        memcpy(data+8, us->unusual_dev->vendorName, min(8, n));
        n = strlen(us->unusual_dev->productName);
        memcpy(data+16, us->unusual_dev->productName, min(16, n));

        data[32] = 0x30 + ((bcdDevice>>12) & 0x0F);
        data[33] = 0x30 + ((bcdDevice>>8) & 0x0F);
        data[34] = 0x30 + ((bcdDevice>>4) & 0x0F);
        data[35] = 0x30 + ((bcdDevice) & 0x0F);
```

- 将bcdDevice、us->unusual_dev->vendorName的前8位，us->unusual_dev->productName的前16位分别填充到data[8] - data[35]；
- 当然，此处都是针对那些unusual devs来说的。像遵守通用协议的设备来说，则不需要如此。
- 在继续之前，再解释下fill_inquiry_response()参数data_ptr数组的前8字节的含义：
    - data_ptr[0]已经介绍过；
    - data_ptr[1]被赋值为0x80，表示这个设备是可移除的；
    - data_ptr[2]被赋值为0x02，表示设备遵循SCSI-2；
    - data_ptr[3]被赋值为0x02，表示数据格式遵循国际标准化组织所规定的格式；
    - data_ptr[4]被赋值为0x1F，该字节被称为additional length，附加参数的长度，即除了用这么一个标准格式的数据响应之外，可能还会返回更多的一些信息；
    
==【usb_stor_set_xfer_buf()函数解析】==

&ensp;&ensp;&ensp;&ensp;函数实现如下，正如英文注释，即存储该buffer内容到srb的transfer buffer中，并设置SCSI residue。

~~~C
/*
 * Store the contents of buffer into srb's transfer buffer and set the
 * SCSI residue.
 */
void usb_stor_set_xfer_buf(unsigned char *buffer,
    unsigned int buflen, struct scsi_cmnd *srb)
{
    unsigned int offset = 0;
    struct scatterlist *sg = NULL;

    buflen = min(buflen, scsi_bufflen(srb));
    buflen = usb_stor_access_xfer_buf(buffer, buflen, srb, &sg, &offset,
            TO_XFER_BUF);
    if (buflen < scsi_bufflen(srb))
        scsi_set_resid(srb, scsi_bufflen(srb) - buflen);
}
EXPORT_SYMBOL_GPL(usb_stor_set_xfer_buf);
~~~

&ensp;&ensp;&ensp;&ensp;取buflen和scsi_bufflen(srb)之间最小值，继续跟踪usb_stor_access_xfer_buf()函数，此函数设计到内存管理；代码实现如下：
```C
/*
 * Copy a buffer of length buflen to/from the srb's transfer buffer.
 * Update the **sgptr and *offset variables so that the next copy will
 * pick up from where this one left off.
 */
unsigned int usb_stor_access_xfer_buf(unsigned char *buffer,
    unsigned int buflen, struct scsi_cmnd *srb, struct scatterlist **sgptr,
    unsigned int *offset, enum xfer_buf_dir dir)
{
    unsigned int cnt = 0;
    struct scatterlist *sg = *sgptr;
    struct sg_mapping_iter miter;
    unsigned int nents = scsi_sg_count(srb); 

    if (sg)
        nents = sg_nents(sg);
    else
        sg = scsi_sglist(srb);

    sg_miter_start(&miter, sg, nents, dir == FROM_XFER_BUF ?
        SG_MITER_FROM_SG: SG_MITER_TO_SG);

    if (!sg_miter_skip(&miter, *offset))
        return cnt;

    while (sg_miter_next(&miter) && cnt < buflen) {
        unsigned int len = min_t(unsigned int, miter.length,
                buflen - cnt);

        if (dir == FROM_XFER_BUF)
            memcpy(buffer + cnt, miter.addr, len);
        else
            memcpy(miter.addr, buffer + cnt, len);

        if (*offset + len < miter.piter.sg->length) {
            *offset += len;
            *sgptr = miter.piter.sg;
        } else {
            *offset = 0;
            *sgptr = sg_next(miter.piter.sg);
        }
        cnt += len;
    }
    sg_miter_stop(&miter);

    return cnt;
}
EXPORT_SYMBOL_GPL(usb_stor_access_xfer_buf);
```
&ensp;&ensp;&ensp;&ensp;先介绍下scatter/gather，这是一种用于高性能IO的标准技术，通常意味着一种DMA传输方式，对于一个给定的数据块，可能在内存中存在于一些离散的缓冲区，换言之，就是说一些不连续的内存缓冲区一起保存一个数据块，如果没有 scatter/gather呢，那么当我们要建立一个从内存到磁盘的传输，那么操作系统通常会为每一个buffer做一次传输，或者干脆就是把这些不连续的buffer里边的冬冬全都移动到另一个很大的buffer里边，然后再开始传输。那么这两种方法显然都是效率不高的。毫无疑问，如果【操作系统/驱动程序/硬件】能够把这些来自内存中离散位置的数据收集起来(gather up)并转移她们到适当位置整个这个步骤是一个单一的操作的话，效率肯定就会更高。反之，如果要从磁盘向内存中传输，而有一个单一的操作能够把数据块直接分散开来(scatter)到达内存中需要的位置，而不再需要中间的那个块移动，或者别的方法，那么显然，效率总会更高。

&ensp;&ensp;&ensp;&ensp;尽管如此，此处的sg_miter_start()、sg_miter_skip()、sg_miter_next()、sg_miter_stop()函数仍然很难理解，暂且不介绍，等以后学精了再补充；

&ensp;&ensp;&ensp;&ensp;回到usb_stor_set_xfer_buf()函数，有关SCSI residue的设定，residue是指在一次传输中，数据并未传输完，则先记下还剩未传输的部分。

---

&ensp;&ensp;&ensp;&ensp;总之，usb_stor_set_xfer_buf()函数的目的，就是将填充好的data数据结构，拷贝到us->srb的transfer buffer中，以反馈回SCSI中间层；

&ensp;&ensp;&ensp;&ensp;回到usb_stor_control_thread()内核线程，srb->result = SAM_STAT_GOOD，填充完srb后，再设置命令处理结果标记，标志命令已被device"处理"。

#### 2.7.4 us->proto_handler处理
&ensp;&ensp;&ensp;&ensp;如果所有的检测都已通过，则USB storage驱动层已获取到srb命令，现在是调用协议层和传输层来完成命令的下发。

&ensp;&ensp;&ensp;&ensp;us->proto_handler(srb, us)函数在usb_stor_probe1()的get_protocol()函数中指定，因为U盘遵循USB_SC_SCSI协议，所以直接调用usb_stor_transparent_scsi_command()。

##### 2.7.4.1 usb_stor_transparent_scsi_command()函数一览
&ensp;&ensp;&ensp;&ensp;函数实现如下：
```C
void usb_stor_transparent_scsi_command(struct scsi_cmnd *srb,
                       struct us_data *us)
{
    /* send the command to the transport layer */
    usb_stor_invoke_transport(srb, us);
}
EXPORT_SYMBOL_GPL(usb_stor_transparent_scsi_command);
```
```C
/*
 * Invoke the transport and basic error-handling/recovery methods
 *  
 * This is used by the protocol layers to actually send the message to
 * the device and receive the response.
 */
void usb_stor_invoke_transport(struct scsi_cmnd *srb, struct us_data *us)
{
    int need_auto_sense;
    int result;
 
    /* send the command to the transport layer */
    scsi_set_resid(srb, 0);
    result = us->transport(srb, us);

    /*
     * if the command gets aborted by the higher layers, we need to
     * short-circuit all other processing
     */
    if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
        usb_stor_dbg(us, "-- command was aborted\n");
        srb->result = DID_ABORT << 16;
        goto Handle_Errors;
    }
    
    /* if there is a transport error, reset and don't auto-sense */
    if (result == USB_STOR_TRANSPORT_ERROR) {
        usb_stor_dbg(us, "-- transport indicates error, resetting\n");
        srb->result = DID_ERROR << 16;
        goto Handle_Errors;
    }

    /* if the transport provided its own sense data, don't auto-sense */
    if (result == USB_STOR_TRANSPORT_NO_SENSE) {
        srb->result = SAM_STAT_CHECK_CONDITION;
        last_sector_hacks(us, srb);
        return;
    }

    srb->result = SAM_STAT_GOOD;

    /*
     * Determine if we need to auto-sense
     *
     * I normally don't use a flag like this, but it's almost impossible
     * to understand what's going on here if I don't.
     */
    need_auto_sense = 0;

    /*
     * If we're running the CB transport, which is incapable
     * of determining status on its own, we will auto-sense
     * unless the operation involved a data-in transfer.  Devices
     * can signal most data-in errors by stalling the bulk-in pipe.
     */
    if ((us->protocol == USB_PR_CB || us->protocol == USB_PR_DPCM_USB) &&
            srb->sc_data_direction != DMA_FROM_DEVICE) {
        usb_stor_dbg(us, "-- CB transport device requiring auto-sense\n");
        need_auto_sense = 1;
    }

    /*
     * If we have a failure, we're going to do a REQUEST_SENSE 
     * automatically.  Note that we differentiate between a command
     * "failure" and an "error" in the transport mechanism.
     */
    if (result == USB_STOR_TRANSPORT_FAILED) {
        usb_stor_dbg(us, "-- transport indicates command failure\n");
        need_auto_sense = 1;
    }

    /*
     * Determine if this device is SAT by seeing if the
     * command executed successfully.  Otherwise we'll have
     * to wait for at least one CHECK_CONDITION to determine
     * SANE_SENSE support
     */
    if (unlikely((srb->cmnd[0] == ATA_16 || srb->cmnd[0] == ATA_12) &&
        result == USB_STOR_TRANSPORT_GOOD &&
        !(us->fflags & US_FL_SANE_SENSE) &&
        !(us->fflags & US_FL_BAD_SENSE) &&
        !(srb->cmnd[2] & 0x20))) {
        usb_stor_dbg(us, "-- SAT supported, increasing auto-sense\n");
        us->fflags |= US_FL_SANE_SENSE;
    }

    /*
     * A short transfer on a command where we don't expect it
     * is unusual, but it doesn't mean we need to auto-sense.
     */
    if ((scsi_get_resid(srb) > 0) &&
        !((srb->cmnd[0] == REQUEST_SENSE) ||
          (srb->cmnd[0] == INQUIRY) ||
          (srb->cmnd[0] == MODE_SENSE) ||
          (srb->cmnd[0] == LOG_SENSE) ||
          (srb->cmnd[0] == MODE_SENSE_10))) {
        usb_stor_dbg(us, "-- unexpectedly short transfer\n");
    }

    /* Now, if we need to do the auto-sense, let's do it */
    if (need_auto_sense) {
        int temp_result;
        struct scsi_eh_save ses;
        int sense_size = US_SENSE_SIZE;
        struct scsi_sense_hdr sshdr;
        const u8 *scdd;
        u8 fm_ili;

        /* device supports and needs bigger sense buffer */
        if (us->fflags & US_FL_SANE_SENSE)
            sense_size = ~0;
Retry_Sense:
        usb_stor_dbg(us, "Issuing auto-REQUEST_SENSE\n");

        scsi_eh_prep_cmnd(srb, &ses, NULL, 0, sense_size);

        /* FIXME: we must do the protocol translation here */
        if (us->subclass == USB_SC_RBC || us->subclass == USB_SC_SCSI ||
                us->subclass == USB_SC_CYP_ATACB)
            srb->cmd_len = 6;
        else
            srb->cmd_len = 12;

        /* issue the auto-sense command */
        scsi_set_resid(srb, 0);
        temp_result = us->transport(us->srb, us);

        /* let's clean up right away */
        scsi_eh_restore_cmnd(srb, &ses);

        if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
            usb_stor_dbg(us, "-- auto-sense aborted\n");
            srb->result = DID_ABORT << 16;

            /* If SANE_SENSE caused this problem, disable it */
            if (sense_size != US_SENSE_SIZE) {
                us->fflags &= ~US_FL_SANE_SENSE;
                us->fflags |= US_FL_BAD_SENSE;
            }
            goto Handle_Errors;
        }

        /*
         * Some devices claim to support larger sense but fail when
         * trying to request it. When a transport failure happens
         * using US_FS_SANE_SENSE, we always retry with a standard
         * (small) sense request. This fixes some USB GSM modems
         */
        if (temp_result == USB_STOR_TRANSPORT_FAILED &&
                sense_size != US_SENSE_SIZE) {
            usb_stor_dbg(us, "-- auto-sense failure, retry small sense\n");
            sense_size = US_SENSE_SIZE;
            us->fflags &= ~US_FL_SANE_SENSE;
            us->fflags |= US_FL_BAD_SENSE;
            goto Retry_Sense;
        }

        /* Other failures */
        if (temp_result != USB_STOR_TRANSPORT_GOOD) {
            usb_stor_dbg(us, "-- auto-sense failure\n");

            /*
             * we skip the reset if this happens to be a
             * multi-target device, since failure of an
             * auto-sense is perfectly valid
             */
            srb->result = DID_ERROR << 16;
            if (!(us->fflags & US_FL_SCM_MULT_TARG))
                goto Handle_Errors;
            return;
        }

        /*
         * If the sense data returned is larger than 18-bytes then we
         * assume this device supports requesting more in the future.
         * The response code must be 70h through 73h inclusive.
         */
        if (srb->sense_buffer[7] > (US_SENSE_SIZE - 8) &&
            !(us->fflags & US_FL_SANE_SENSE) &&
            !(us->fflags & US_FL_BAD_SENSE) &&
            (srb->sense_buffer[0] & 0x7C) == 0x70) {
            usb_stor_dbg(us, "-- SANE_SENSE support enabled\n");
            us->fflags |= US_FL_SANE_SENSE;

            /*
             * Indicate to the user that we truncated their sense
             * because we didn't know it supported larger sense.
             */
            usb_stor_dbg(us, "-- Sense data truncated to %i from %i\n",
                     US_SENSE_SIZE,
                     srb->sense_buffer[7] + 8);
            srb->sense_buffer[7] = (US_SENSE_SIZE - 8);
        }

        scsi_normalize_sense(srb->sense_buffer, SCSI_SENSE_BUFFERSIZE,
                     &sshdr);

        usb_stor_dbg(us, "-- Result from auto-sense is %d\n",
                 temp_result);
        usb_stor_dbg(us, "-- code: 0x%x, key: 0x%x, ASC: 0x%x, ASCQ: 0x%x\n",
                 sshdr.response_code, sshdr.sense_key,
                 sshdr.asc, sshdr.ascq);
#ifdef CONFIG_USB_STORAGE_DEBUG
        usb_stor_show_sense(us, sshdr.sense_key, sshdr.asc, sshdr.ascq);
#endif

        /* set the result so the higher layers expect this data */
        srb->result = SAM_STAT_CHECK_CONDITION;

        scdd = scsi_sense_desc_find(srb->sense_buffer,
                        SCSI_SENSE_BUFFERSIZE, 4);
        fm_ili = (scdd ? scdd[3] : srb->sense_buffer[2]) & 0xA0;

        /*
         * We often get empty sense data.  This could indicate that
         * everything worked or that there was an unspecified
         * problem.  We have to decide which.
         */
        if (sshdr.sense_key == 0 && sshdr.asc == 0 && sshdr.ascq == 0 &&
            fm_ili == 0) {
            /*
             * If things are really okay, then let's show that.
             * Zero out the sense buffer so the higher layers
             * won't realize we did an unsolicited auto-sense.
             */
            if (result == USB_STOR_TRANSPORT_GOOD) {
                srb->result = SAM_STAT_GOOD;
                srb->sense_buffer[0] = 0x0;
            }

            /*
             * ATA-passthru commands use sense data to report
             * the command completion status, and often devices
             * return Check Condition status when nothing is
             * wrong.
             */
            else if (srb->cmnd[0] == ATA_16 ||
                    srb->cmnd[0] == ATA_12) {
                /* leave the data alone */
            }

            /*
             * If there was a problem, report an unspecified
             * hardware error to prevent the higher layers from
             * entering an infinite retry loop.
             */
            else {
                srb->result = DID_ERROR << 16;
                if ((sshdr.response_code & 0x72) == 0x72)
                    srb->sense_buffer[1] = HARDWARE_ERROR;
                else
                    srb->sense_buffer[2] = HARDWARE_ERROR;
            }
        }
    }

    /*
     * Some devices don't work or return incorrect data the first
     * time they get a READ(10) command, or for the first READ(10)
     * after a media change.  If the INITIAL_READ10 flag is set,
     * keep track of whether READ(10) commands succeed.  If the
     * previous one succeeded and this one failed, set the REDO_READ10
     * flag to force a retry.
     */
    if (unlikely((us->fflags & US_FL_INITIAL_READ10) &&
            srb->cmnd[0] == READ_10)) {
        if (srb->result == SAM_STAT_GOOD) {
            set_bit(US_FLIDX_READ10_WORKED, &us->dflags);
        } else if (test_bit(US_FLIDX_READ10_WORKED, &us->dflags)) {
            clear_bit(US_FLIDX_READ10_WORKED, &us->dflags);
            set_bit(US_FLIDX_REDO_READ10, &us->dflags);
        }

        /*
         * Next, if the REDO_READ10 flag is set, return a result
         * code that will cause the SCSI core to retry the READ(10)
         * command immediately.
         */
        if (test_bit(US_FLIDX_REDO_READ10, &us->dflags)) {
            clear_bit(US_FLIDX_REDO_READ10, &us->dflags);
            srb->result = DID_IMM_RETRY << 16;
            srb->sense_buffer[0] = 0;
        }
    }

    /* Did we transfer less than the minimum amount required? */
    if ((srb->result == SAM_STAT_GOOD || srb->sense_buffer[2] == 0) &&
            scsi_bufflen(srb) - scsi_get_resid(srb) < srb->underflow)
        srb->result = DID_ERROR << 16;

    last_sector_hacks(us, srb);
    return;

    /*
     * Error and abort processing: try to resynchronize with the device
     * by issuing a port reset.  If that fails, try a class-specific
     * device reset.
     */
  Handle_Errors:

    /*
     * Set the RESETTING bit, and clear the ABORTING bit so that
     * the reset may proceed.
     */
    scsi_lock(us_to_host(us));
    set_bit(US_FLIDX_RESETTING, &us->dflags);
    clear_bit(US_FLIDX_ABORTING, &us->dflags);
    scsi_unlock(us_to_host(us));

    /*
     * We must release the device lock because the pre_reset routine
     * will want to acquire it.
     */
    mutex_unlock(&us->dev_mutex);
    result = usb_stor_port_reset(us);
    mutex_lock(&us->dev_mutex);

    if (result < 0) {
        scsi_lock(us_to_host(us));
        usb_stor_report_device_reset(us);
        scsi_unlock(us_to_host(us));
        us->transport_reset(us);
    }
    clear_bit(US_FLIDX_RESETTING, &us->dflags);
    last_sector_hacks(us, srb);
}
```
&ensp;&ensp;&ensp;&ensp;实测，usb_stor_invoke_transport()函数当前长达319行，所以要解析起来还是蛮复杂的。但是在讲解之前，还得先介绍us->transport(srb, us)。

##### 2.7.4.2 usb_stor_Bulk_transport()函数解析
&ensp;&ensp;&ensp;&ensp; 函数的实现如下：
```C
int usb_stor_Bulk_transport(struct scsi_cmnd *srb, struct us_data *us)
{
    struct bulk_cb_wrap *bcb = (struct bulk_cb_wrap *) us->iobuf;
    struct bulk_cs_wrap *bcs = (struct bulk_cs_wrap *) us->iobuf;
    unsigned int transfer_length = scsi_bufflen(srb);
    unsigned int residue;
    int result;
    int fake_sense = 0;
    unsigned int cswlen;
    unsigned int cbwlen = US_BULK_CB_WRAP_LEN;

    /* Take care of BULK32 devices; set extra byte to 0 */
    if (unlikely(us->fflags & US_FL_BULK32)) {
        cbwlen = 32;
        us->iobuf[31] = 0;
    }

    /* set up the command wrapper */
    bcb->Signature = cpu_to_le32(US_BULK_CB_SIGN);
    bcb->DataTransferLength = cpu_to_le32(transfer_length);
    bcb->Flags = srb->sc_data_direction == DMA_FROM_DEVICE ?
        US_BULK_FLAG_IN : 0;
    bcb->Tag = ++us->tag;
    bcb->Lun = srb->device->lun;
    if (us->fflags & US_FL_SCM_MULT_TARG)
        bcb->Lun |= srb->device->id << 4;
    bcb->Length = srb->cmd_len; 

    /* copy the command payload */
    memset(bcb->CDB, 0, sizeof(bcb->CDB));
    memcpy(bcb->CDB, srb->cmnd, bcb->Length);

    /* send it to out endpoint */
    usb_stor_dbg(us, "Bulk Command S 0x%x T 0x%x L %d F %d Trg %d LUN %d CL %d\n",
             le32_to_cpu(bcb->Signature), bcb->Tag,
             le32_to_cpu(bcb->DataTransferLength), bcb->Flags,
             (bcb->Lun >> 4), (bcb->Lun & 0x0F),
             bcb->Length);
    result = usb_stor_bulk_transfer_buf(us, us->send_bulk_pipe,
                bcb, cbwlen, NULL);
    usb_stor_dbg(us, "Bulk command transfer result=%d\n", result);
    if (result != USB_STOR_XFER_GOOD)
        return USB_STOR_TRANSPORT_ERROR;

    /* DATA STAGE */
    /* send/receive data payload, if there is any */

    /*
     * Some USB-IDE converter chips need a 100us delay between the
     * command phase and the data phase.  Some devices need a little
     * more than that, probably because of clock rate inaccuracies.
     */
    if (unlikely(us->fflags & US_FL_GO_SLOW))
        usleep_range(125, 150);

    if (transfer_length) {
        unsigned int pipe = srb->sc_data_direction == DMA_FROM_DEVICE ?
                us->recv_bulk_pipe : us->send_bulk_pipe;
        result = usb_stor_bulk_srb(us, pipe, srb);
        usb_stor_dbg(us, "Bulk data transfer result 0x%x\n", result);
        if (result == USB_STOR_XFER_ERROR)
            return USB_STOR_TRANSPORT_ERROR;

        /*
         * If the device tried to send back more data than the
         * amount requested, the spec requires us to transfer
         * the CSW anyway.  Since there's no point retrying the
         * the command, we'll return fake sense data indicating
         * Illegal Request, Invalid Field in CDB.
         */
        if (result == USB_STOR_XFER_LONG)
            fake_sense = 1;

        /*
         * Sometimes a device will mistakenly skip the data phase
         * and go directly to the status phase without sending a
         * zero-length packet.  If we get a 13-byte response here,
         * check whether it really is a CSW.
         */
        if (result == USB_STOR_XFER_SHORT &&
                srb->sc_data_direction == DMA_FROM_DEVICE &&
                transfer_length - scsi_get_resid(srb) ==
                    US_BULK_CS_WRAP_LEN) {
            struct scatterlist *sg = NULL;
            unsigned int offset = 0;

            if (usb_stor_access_xfer_buf((unsigned char *) bcs,
                    US_BULK_CS_WRAP_LEN, srb, &sg,
                    &offset, FROM_XFER_BUF) ==
                        US_BULK_CS_WRAP_LEN &&
                    bcs->Signature ==
                        cpu_to_le32(US_BULK_CS_SIGN)) {
                usb_stor_dbg(us, "Device skipped data phase\n");
                scsi_set_resid(srb, transfer_length);
                goto skipped_data_phase;
            }
        }
    }

    /*
     * See flow chart on pg 15 of the Bulk Only Transport spec for
     * an explanation of how this code works.
     */

    /* get CSW for device status */
    usb_stor_dbg(us, "Attempting to get CSW...\n");
    result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, &cswlen);

    /*
     * Some broken devices add unnecessary zero-length packets to the
     * end of their data transfers.  Such packets show up as 0-length
     * CSWs.  If we encounter such a thing, try to read the CSW again.
     */
    if (result == USB_STOR_XFER_SHORT && cswlen == 0) {
        usb_stor_dbg(us, "Received 0-length CSW; retrying...\n");
        result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, &cswlen);
    }

    /* did the attempt to read the CSW fail? */
    if (result == USB_STOR_XFER_STALLED) {

        /* get the status again */
        usb_stor_dbg(us, "Attempting to get CSW (2nd try)...\n");
        result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, NULL);
    }

    /* if we still have a failure at this point, we're in trouble */
    usb_stor_dbg(us, "Bulk status result = %d\n", result);
    if (result != USB_STOR_XFER_GOOD)
        return USB_STOR_TRANSPORT_ERROR;

 skipped_data_phase:
    /* check bulk status */
    residue = le32_to_cpu(bcs->Residue);
    usb_stor_dbg(us, "Bulk Status S 0x%x T 0x%x R %u Stat 0x%x\n",
             le32_to_cpu(bcs->Signature), bcs->Tag,
             residue, bcs->Status);
    if (!(bcs->Tag == us->tag || (us->fflags & US_FL_BULK_IGNORE_TAG)) ||
        bcs->Status > US_BULK_STAT_PHASE) {
        usb_stor_dbg(us, "Bulk logical error\n");
        return USB_STOR_TRANSPORT_ERROR;
    }

    /*
     * Some broken devices report odd signatures, so we do not check them
     * for validity against the spec. We store the first one we see,
     * and check subsequent transfers for validity against this signature.
     */
    if (!us->bcs_signature) {
        us->bcs_signature = bcs->Signature;
        if (us->bcs_signature != cpu_to_le32(US_BULK_CS_SIGN))
            usb_stor_dbg(us, "Learnt BCS signature 0x%08X\n",
                     le32_to_cpu(us->bcs_signature));
    } else if (bcs->Signature != us->bcs_signature) {
        usb_stor_dbg(us, "Signature mismatch: got %08X, expecting %08X\n",
                 le32_to_cpu(bcs->Signature),
                 le32_to_cpu(us->bcs_signature));
        return USB_STOR_TRANSPORT_ERROR;
    }

    /*
     * try to compute the actual residue, based on how much data
     * was really transferred and what the device tells us
     */
    if (residue && !(us->fflags & US_FL_IGNORE_RESIDUE)) {

        /*
         * Heuristically detect devices that generate bogus residues
         * by seeing what happens with INQUIRY and READ CAPACITY
         * commands.
         */
        if (bcs->Status == US_BULK_STAT_OK &&
                scsi_get_resid(srb) == 0 &&
                    ((srb->cmnd[0] == INQUIRY &&
                        transfer_length == 36) ||
                    (srb->cmnd[0] == READ_CAPACITY &&
                        transfer_length == 8))) {
            us->fflags |= US_FL_IGNORE_RESIDUE;

        } else {
            residue = min(residue, transfer_length);
            scsi_set_resid(srb, max(scsi_get_resid(srb),
                                                   (int) residue));
        }
    }

    /* based on the status code, we report good or bad */
    switch (bcs->Status) {
        case US_BULK_STAT_OK:
            /* device babbled -- return fake sense data */
            if (fake_sense) {
                memcpy(srb->sense_buffer,
                       usb_stor_sense_invalidCDB,
                       sizeof(usb_stor_sense_invalidCDB));
                return USB_STOR_TRANSPORT_NO_SENSE;
            }

            /* command good -- note that data could be short */
            return USB_STOR_TRANSPORT_GOOD;

        case US_BULK_STAT_FAIL:
            /* command failed */
            return USB_STOR_TRANSPORT_FAILED;

        case US_BULK_STAT_PHASE:
            /*
             * phase error -- note that a transport reset will be
             * invoked by the invoke_transport() function
             */
            return USB_STOR_TRANSPORT_ERROR;
    }

    /* we should never get here, but if we do, we're in trouble */
    return USB_STOR_TRANSPORT_ERROR;
}
EXPORT_SYMBOL_GPL(usb_stor_Bulk_transport);
```
&ensp;&ensp;&ensp;&ensp;实测代码长218行，非常显然，usb_stor_Bulk_transport()和usb_stor_transparent_scsi_command()才是USB storage的主体核心；下面逐一进行分析；

###### 2.7.4.2.1 函数变量分析
&ensp;&ensp;&ensp;&ensp;函数实现了很多变量，代码如下：
```C
    struct bulk_cb_wrap *bcb = (struct bulk_cb_wrap *) us->iobuf;
    struct bulk_cs_wrap *bcs = (struct bulk_cs_wrap *) us->iobuf;
    unsigned int transfer_length = scsi_bufflen(srb);
    unsigned int residue;
    int result;
    int fake_sense = 0;
    unsigned int cswlen;
    unsigned int cbwlen = US_BULK_CB_WRAP_LEN;
```

- struct bulk_cb_wrap结构体实例bcb；是BOT协议中对scsi cmd/data的封装结构，先看下该结构体：
```C
/* command block wrapper */
struct bulk_cb_wrap {
    __le32  Signature;      /* contains 'USBC' */
    __u32   Tag;            /* unique per command id */
    __le32  DataTransferLength; /* size of data */
    __u8    Flags;          /* direction in bit 0 */
    __u8    Lun;            /* LUN normally 0 */
    __u8    Length;         /* length of the CDB */
    __u8    CDB[16];        /* max command */
};
```
&ensp;&ensp;&ensp;&ensp;这个结构体是和手册上定义的Command Block Wrapper (CBW)一模一样的；而us->iobuf是已经被申请了的DMA内存；

- struct bulk_cs_wrap结构体实例bcs；是BOT协议中device传回的状态封装结构，现看下该结构体：
```C
/* command status wrapper */
struct bulk_cs_wrap {
    __le32  Signature;  /* contains 'USBS' */
    __u32   Tag;        /* same as original command */
    __le32  Residue;    /* amount not transferred */
    __u8    Status;     /* see below */
};
```

- transfer_length为本次传输cmd/data数据的长度；
- residue为scsi剩余传输的数据；
- fake_sense，某些错误处理时需要用到；
- cswlen，device返回时的数据长度；
- cbwlen，host下发的bcb长度，此处US_BULK_CB_WRAP_LEN宏定义为31字节；

&ensp;&ensp;&ensp;&ensp;usb_stor_Bulk_transport()函数主要分了四个部分，摘取代码英文注释分为如下：

- set up the command wrapper；
- DATA STAGE
- get CSW for device status；
- check bulk status。

&ensp;&ensp;&ensp;&ensp;接下来会逐一分小节介绍，在这之前，usb_stor_Bulk_transport()还有个unlikely的判断：
```C
    /* Take care of BULK32 devices; set extra byte to 0 */
    if (unlikely(us->fflags & US_FL_BULK32)) {
        cbwlen = 32;
        us->iobuf[31] = 0;
    }
```
&ensp;&ensp;&ensp;&ensp;即用于根据us->fflags的US_FL_BULK32标记，做出限定CBW的长度，这也是个特例。

###### 2.7.4.2.2 set up the command wrapper
==【填充CBW】==

&ensp;&ensp;&ensp;&ensp;此部分，即优先发送一个SCSI命令，通过封装成CBW结构，即由bcb携带并发送给device：
```C
    /* set up the command wrapper */
    bcb->Signature = cpu_to_le32(US_BULK_CB_SIGN);
    bcb->DataTransferLength = cpu_to_le32(transfer_length);
    bcb->Flags = srb->sc_data_direction == DMA_FROM_DEVICE ?
        US_BULK_FLAG_IN : 0;
    bcb->Tag = ++us->tag;
    bcb->Lun = srb->device->lun;
    if (us->fflags & US_FL_SCM_MULT_TARG)
        bcb->Lun |= srb->device->id << 4;
    bcb->Length = srb->cmd_len;

    /* copy the command payload */
    memset(bcb->CDB, 0, sizeof(bcb->CDB));
    memcpy(bcb->CDB, srb->cmnd, bcb->Length);

    /* send it to out endpoint */
    usb_stor_dbg(us, "Bulk Command S 0x%x T 0x%x L %d F %d Trg %d LUN %d CL %d\n",
             le32_to_cpu(bcb->Signature), bcb->Tag,
             le32_to_cpu(bcb->DataTransferLength), bcb->Flags,
             (bcb->Lun >> 4), (bcb->Lun & 0x0F),
             bcb->Length);
    result = usb_stor_bulk_transfer_buf(us, us->send_bulk_pipe,
                bcb, cbwlen, NULL);
    usb_stor_dbg(us, "Bulk command transfer result=%d\n", result);
    if (result != USB_STOR_XFER_GOOD)
        return USB_STOR_TRANSPORT_ERROR;
```

- 首先填充bcb结构体元素bcb->Signature、bcb->DataTransferLength、bcb->Flags、bcb->Tag、bcb->Lun、bcb->Length；
    - bcb->Signature：US_BULK_CB_SIGN宏定义值为0x43425355，代表CBW；
    - bcb->DataTransferLength：host希望传输的data字节长度，根据手册，所有的CBW和CSW都是little endian，所以此处得用cpu_to_le32()做了封装；
    - bcb->Flags：此处是设置接下来的数据传输方向，即bit[[7] - 1表示从device到host；
    - bcb->Tag：此处直接由us->tag赋值，Tag元素的释义是“unique per command id”，该值是用来识别匹配CBW与CSW之用；
    - bcb->Lun：指定与哪一个lun通信，此处对于U盘来说意义不大；
    - bcb->Length：cmd的长度，只能是1到16之间的数。
    
&ensp;&ensp;&ensp;&ensp;以上所涉及到的宏定义代码实现如下（包含上面部分宏定义）：
```C
#define US_BULK_CB_WRAP_LEN 31
#define US_BULK_CB_SIGN     0x43425355  /* spells out 'USBC' */
#define US_BULK_FLAG_IN     (1 << 7)
#define US_BULK_FLAG_OUT    0
```

==【拷贝cmd命令数据】==

&ensp;&ensp;&ensp;&ensp;将srb内携带的cmd数据填充到bcb封装中，此部分也是填充CBW结构：
```C
    /* copy the command payload */
    memset(bcb->CDB, 0, sizeof(bcb->CDB));
    memcpy(bcb->CDB, srb->cmnd, bcb->Length);
```

==【发送CBW】==
&ensp;&ensp;&ensp;&ensp;使用usb_stor_bulk_transfer_buf()函数发送携带SCSI smd的CBW，代码实现如下：
```C
/*
 * Transfer one buffer via bulk pipe, without timeouts, but allowing early
 * termination.  Return codes are USB_STOR_XFER_xxx.  If the bulk pipe
 * stalls during the transfer, the halt is automatically cleared.
 */
int usb_stor_bulk_transfer_buf(struct us_data *us, unsigned int pipe,
    void *buf, unsigned int length, unsigned int *act_len)
{
    int result;
    
    usb_stor_dbg(us, "xfer %u bytes\n", length);
    
    /* fill and submit the URB */
    usb_fill_bulk_urb(us->current_urb, us->pusb_dev, pipe, buf, length,
              usb_stor_blocking_completion, NULL);
    result = usb_stor_msg_common(us, 0);

    /* store the actual length of the data transferred */
    if (act_len)
        *act_len = us->current_urb->actual_length;
    return interpret_urb_result(us, pipe, length, result,
            us->current_urb->actual_length);
}
EXPORT_SYMBOL_GPL(usb_stor_bulk_transfer_buf);
```
&ensp;&ensp;&ensp;&ensp;代码流程如下：

- 使用usb_fill_bulk_urb()填充urb，赋值us->current_urb->complete = complete_fn；
- 调用usb_stor_msg_common()初始化完成量urb_done，及使用usb_submit_urb()上报urb给USB HCD，并利用wait_for_completion_interruptible_timeout()等待处理结果；返回urb状态（该函数之前解析过，此处不继续）；
- 利用interpret_urb_result()解析urb状态结果，该函数实现如下：

---
```C
    return interpret_urb_result(us, pipe, length, result,
            us->current_urb->actual_length);
```
```C
/*
 * Interpret the results of a URB transfer
 *  
 * This function prints appropriate debugging messages, clears halts on
 * non-control endpoints, and translates the status to the corresponding
 * USB_STOR_XFER_xxx return code.
 */
static int interpret_urb_result(struct us_data *us, unsigned int pipe,
        unsigned int length, int result, unsigned int partial)
{
    usb_stor_dbg(us, "Status code %d; transferred %u/%u\n",
             result, partial, length);
    switch (result) {

    /* no error code; did we send all the data? */
    case 0:
        if (partial != length) {
            usb_stor_dbg(us, "-- short transfer\n");
            return USB_STOR_XFER_SHORT;
        }

        usb_stor_dbg(us, "-- transfer complete\n");
        return USB_STOR_XFER_GOOD;

    /* stalled */
    case -EPIPE:
        /*
         * for control endpoints, (used by CB[I]) a stall indicates
         * a failed command
         */
        if (usb_pipecontrol(pipe)) {
            usb_stor_dbg(us, "-- stall on control pipe\n");
            return USB_STOR_XFER_STALLED;
        }

        /* for other sorts of endpoint, clear the stall */
        usb_stor_dbg(us, "clearing endpoint halt for pipe 0x%x\n",
                 pipe);
        if (usb_stor_clear_halt(us, pipe) < 0)
            return USB_STOR_XFER_ERROR;
        return USB_STOR_XFER_STALLED;

    /* babble - the device tried to send more than we wanted to read */
    case -EOVERFLOW:
        usb_stor_dbg(us, "-- babble\n");
        return USB_STOR_XFER_LONG;

    /* the transfer was cancelled by abort, disconnect, or timeout */
    case -ECONNRESET:
        usb_stor_dbg(us, "-- transfer cancelled\n");
        return USB_STOR_XFER_ERROR;

    /* short scatter-gather read transfer */
    case -EREMOTEIO:
        usb_stor_dbg(us, "-- short read transfer\n");
        return USB_STOR_XFER_SHORT;

    /* abort or disconnect in progress */
    case -EIO:
        usb_stor_dbg(us, "-- abort or disconnect in progress\n");
        return USB_STOR_XFER_ERROR;

    /* the catch-all error case */
    default:
        usb_stor_dbg(us, "-- unknown error\n");
        return USB_STOR_XFER_ERROR;
    }
}
```
&ensp;&ensp;&ensp;&ensp;针对不同的urb返回状态，返回不同的宏定义标记，宏定义实现如下：
```C
/*
 * usb_stor_bulk_transfer_xxx() return codes, in order of severity
 */

#define USB_STOR_XFER_GOOD  0   /* good transfer                 */
#define USB_STOR_XFER_SHORT 1   /* transferred less than expected */
#define USB_STOR_XFER_STALLED   2   /* endpoint stalled              */
#define USB_STOR_XFER_LONG  3   /* device tried to send too much */
#define USB_STOR_XFER_ERROR 4   /* transfer died in the middle   */
```

---

==【结果检查】==

&ensp;&ensp;&ensp;&ensp;发送完CBW后，此处检查发送urb的状态，即本次USB传输是否成功，若不成功，也就没必须继续了。
```C
    usb_stor_dbg(us, "Bulk command transfer result=%d\n", result);
    if (result != USB_STOR_XFER_GOOD)
        return USB_STOR_TRANSPORT_ERROR;
```
###### 2.7.4.2.3 DATA STAGE
&ensp;&ensp;&ensp;&ensp;先对us->fflags的US_FL_GO_SLOW进行检测，函数解析如下：
```C
    /*
     * Some USB-IDE converter chips need a 100us delay between the
     * command phase and the data phase.  Some devices need a little
     * more than that, probably because of clock rate inaccuracies.
     */
    if (unlikely(us->fflags & US_FL_GO_SLOW))
        usleep_range(125, 150);
```
&ensp;&ensp;&ensp;&ensp;然后针对有中间数据传输的情况进行数据的传送，代码实现如下：
```C
    if (transfer_length) {
        unsigned int pipe = srb->sc_data_direction == DMA_FROM_DEVICE ?
                us->recv_bulk_pipe : us->send_bulk_pipe;
        result = usb_stor_bulk_srb(us, pipe, srb);
        usb_stor_dbg(us, "Bulk data transfer result 0x%x\n", result);
        if (result == USB_STOR_XFER_ERROR)
            return USB_STOR_TRANSPORT_ERROR;

        /*
         * If the device tried to send back more data than the
         * amount requested, the spec requires us to transfer
         * the CSW anyway.  Since there's no point retrying the
         * the command, we'll return fake sense data indicating
         * Illegal Request, Invalid Field in CDB.
         */
        if (result == USB_STOR_XFER_LONG)
            fake_sense = 1;

        /*
         * Sometimes a device will mistakenly skip the data phase
         * and go directly to the status phase without sending a
         * zero-length packet.  If we get a 13-byte response here,
         * check whether it really is a CSW.
         */
        if (result == USB_STOR_XFER_SHORT &&
                srb->sc_data_direction == DMA_FROM_DEVICE &&
                transfer_length - scsi_get_resid(srb) ==
                    US_BULK_CS_WRAP_LEN) {
            struct scatterlist *sg = NULL;
            unsigned int offset = 0;

            if (usb_stor_access_xfer_buf((unsigned char *) bcs,
                    US_BULK_CS_WRAP_LEN, srb, &sg,
                    &offset, FROM_XFER_BUF) ==
                        US_BULK_CS_WRAP_LEN &&
                    bcs->Signature ==
                        cpu_to_le32(US_BULK_CS_SIGN)) {
                usb_stor_dbg(us, "Device skipped data phase\n");
                scsi_set_resid(srb, transfer_length);
                goto skipped_data_phase;
            }
        }
    }
```
&ensp;&ensp;&ensp;&ensp;下面解析代码流程：

- 通过srb->sc_data_direction获取对应方向bulk pipe；
- 使用usb_stor_bulk_srb()做数据传输，此部分在2.7.4.2.4小节解析：

###### 2.7.4.2.4 usb_stor_bulk_transfer_sglist()函数解析
```C
/*
 * Common used function. Transfer a complete command
 * via usb_stor_bulk_transfer_sglist() above. Set cmnd resid
 */
int usb_stor_bulk_srb(struct us_data* us, unsigned int pipe,
              struct scsi_cmnd* srb)
{
    unsigned int partial;
    int result = usb_stor_bulk_transfer_sglist(us, pipe, scsi_sglist(srb),
                      scsi_sg_count(srb), scsi_bufflen(srb),
                      &partial);

    scsi_set_resid(srb, scsi_bufflen(srb) - partial);
    return result; 
}
EXPORT_SYMBOL_GPL(usb_stor_bulk_srb);
```
```C
/*
 * Transfer a scatter-gather list via bulk transfer
 *
 * This function does basically the same thing as usb_stor_bulk_transfer_buf()
 * above, but it uses the usbcore scatter-gather library.
 */
static int usb_stor_bulk_transfer_sglist(struct us_data *us, unsigned int pipe,
        struct scatterlist *sg, int num_sg, unsigned int length,
        unsigned int *act_len)
{
    int result;

    /* don't submit s-g requests during abort processing */
    if (test_bit(US_FLIDX_ABORTING, &us->dflags))
        return USB_STOR_XFER_ERROR;

    /* initialize the scatter-gather request block */
    usb_stor_dbg(us, "xfer %u bytes, %d entries\n", length, num_sg);
    result = usb_sg_init(&us->current_sg, us->pusb_dev, pipe, 0,
            sg, num_sg, length, GFP_NOIO);
    if (result) {
        usb_stor_dbg(us, "usb_sg_init returned %d\n", result);
        return USB_STOR_XFER_ERROR;
    }
    
    /*
     * since the block has been initialized successfully, it's now
     * okay to cancel it
     */
    set_bit(US_FLIDX_SG_ACTIVE, &us->dflags);

    /* did an abort occur during the submission? */
    if (test_bit(US_FLIDX_ABORTING, &us->dflags)) {

        /* cancel the request, if it hasn't been cancelled already */
        if (test_and_clear_bit(US_FLIDX_SG_ACTIVE, &us->dflags)) {
            usb_stor_dbg(us, "-- cancelling sg request\n");
            usb_sg_cancel(&us->current_sg);
        }
    }

    /* wait for the completion of the transfer */
    usb_sg_wait(&us->current_sg);
    clear_bit(US_FLIDX_SG_ACTIVE, &us->dflags);

    result = us->current_sg.status;
    if (act_len)
        *act_len = us->current_sg.bytes;
    return interpret_urb_result(us, pipe, length, result,
            us->current_sg.bytes);
}
```
&ensp;&ensp;&ensp;&ensp;直接看usb_stor_bulk_transfer_sglist()函数实现，此处注释已讲解明了，通过bulk pipe传输scatter-gather列表；此处调用了USB core层的scatter-gather库实现；

&ensp;&ensp;&ensp;&ensp;进函数即检测us->dflags的US_FLIDX_ABORTING标记，确认当前USB  storage未出错；

&ensp;&ensp;&ensp;&ensp;下面得着重介绍下scatter-gather库的实现，现看函数usb_sg_init()，查看该函数第一个参数：us->current_sg，这是一个struct usb_sg_request结构体元素，长得和us->current_urb差不多，摘抄struct us_data结构体内这部分元素定义：
```C
    /* control and bulk communications data */
    struct urb      *current_urb;    /* USB requests     */
    struct usb_ctrlrequest  *cr;         /* control requests     */
    struct usb_sg_request   current_sg;  /* scatter-gather req.  */
```
&ensp;&ensp;&ensp;&ensp;此部分即，当传输scsi cmd时，因为数据量少，所以使用了urb request；此处会做大数据量传输，所以使用scatter-gather request；对于每次 urb 请求，我们所作的只是申请一个结构体变量或者说申请指针然后申请内存，第二步就是提交 urb，即调用 usb_submit_urb()，剩下的事情 usb core 就会去帮我们处理了，Linux 中的模块机制酷就酷在这里，每个模块都给别人服务，也同时享受着别人提供的服务。同样对于 sg request,usb core 也实现了这些，我们只需要申请并初始化一个 struct usb_sg_request 的结构体，然后提交，然后 usb core 那边自然就知道该怎么处理了。查看struct usb_sg_request结构体实现：
```C
/** 
 * struct usb_sg_request - support for scatter/gather I/O
 * @status: zero indicates success, else negative errno
 * @bytes: counts bytes transferred.
 *
 * These requests are initialized using usb_sg_init(), and then are used
 * as request handles passed to usb_sg_wait() or usb_sg_cancel().  Most
 * members of the request object aren't for driver access.
 *  
 * The status and bytecount values are valid only after usb_sg_wait()
 * returns.  If the status is zero, then the bytecount matches the total
 * from the request.
 *  
 * After an error completion, drivers may need to clear a halt condition
 * on the endpoint.
 */
struct usb_sg_request {
    int         status;
    size_t          bytes;

    /* private:
     * members below are private to usbcore,
     * and are not provided for driver access!
     */
    spinlock_t      lock;

    struct usb_device   *dev;
    int         pipe;

    int         entries;
    struct urb      **urbs;

    int         count;
    struct completion   complete;
};
```
&ensp;&ensp;&ensp;&ensp;整个 usb 系统都会使用这个数据结构，如果我们希望使用 scatter-gather 方式的话。usb core 已经为我们准备好了数据结构和相应的函数，我们只需要调用即可。一共有三个函数，她们是usb_sg_init()，usb_sg_wait()，usb_sg_cancel()。我们要提交一个 sg 请求，需要做的是，先用 usb_sg_init 来初始化请求，然后 usb_sg_wait()正式提交，然后我们该做的就都做了。如果想撤销一个 sg 请求，那么调用usb_sg_cancel 即可。

&ensp;&ensp;&ensp;&ensp;分析一下usb_sg_init()的参数：

- us->current_sg：使用us->current_sg指示一个scatter-gather request；
- us->pusb_dev：指示是USB device要发送/接收数据；
- pipe：传输pipe，此处是send/recv bulk pipe；
- 0：查看usb_sg_init()参数定义为interrupt端点的轮询速率，即interval，因为BOT用的是bulk传输，所以此处直接指定为0；
- sg：需要传输的sg数组；
- num_sg：sg数组个数；
- length：本次希望传输的数据长度；
- GFP_NOIO：意思就是不能在申请内存的时候进行IO操作。这个和usb_submit_urb(us->current_urb, GFP_NOIO)的标志一样。

&ensp;&ensp;&ensp;&ensp;继续分析：

- 在提交us->current_sg后，设置us->dflags标记US_FLIDX_SG_ACTIVE；这和调用usb_submit_urb()之后一样的设定；
- 再次检测us->dflags的US_FLIDX_ABORTING标记；如果以报错，则检测并清除us->dflags的US_FLIDX_SG_ACTIVE标记；
- 调用usb_sg_wait(&us->current_sg)等待传输处理完成；
- 清除us->dflags的US_FLIDX_SG_ACTIVE标记，标示已用完该功能；
- 获取本次的传输状态us->current_sg.status，如果定义了act_len变量，即实际传输数据长度，则赋值us->current_sg.bytes，这里面保存了实际传输的长度；
- 调用interpret_urb_result()对结果做检测判定，（前面已经解析过）；

&ensp;&ensp;&ensp;&ensp;回到usb_stor_bulk_srb()函数，再用scsi_set_resid设置未传输完的字节数；

&ensp;&ensp;&ensp;&ensp;回到usb_stor_Bulk_transport()，检测本次的传输结果状态：

- 如果result为USB_STOR_XFER_ERROR，则直接返回错误，无法继续了；
- 如果result为USB_STOR_XFER_LONG，则设定fake_sense为1，此处可以在interpret_urb_result()中获得解释：
```C
    /* babble - the device tried to send more than we wanted to read */
    case -EOVERFLOW:
        usb_stor_dbg(us, "-- babble\n");
        return USB_STOR_XFER_LONG;
```
&ensp;&ensp;&ensp;&ensp;即在从device读模式时，device发的数据大于host想要读取的，后面再解析fake_sense不为0的处理；
- 还有一个特例，即如果result为USB_STOR_XFER_SHORT时，需要做进一步判断，代码实现如下：

---
==【skipped_data_phase判断】==
```C
        /*
         * Sometimes a device will mistakenly skip the data phase
         * and go directly to the status phase without sending a
         * zero-length packet.  If we get a 13-byte response here,
         * check whether it really is a CSW.
         */
        if (result == USB_STOR_XFER_SHORT &&
                srb->sc_data_direction == DMA_FROM_DEVICE &&
                transfer_length - scsi_get_resid(srb) ==
                    US_BULK_CS_WRAP_LEN) {
            struct scatterlist *sg = NULL;
            unsigned int offset = 0;

            if (usb_stor_access_xfer_buf((unsigned char *) bcs,
                    US_BULK_CS_WRAP_LEN, srb, &sg,
                    &offset, FROM_XFER_BUF) ==
                        US_BULK_CS_WRAP_LEN &&
                    bcs->Signature ==
                        cpu_to_le32(US_BULK_CS_SIGN)) {
                usb_stor_dbg(us, "Device skipped data phase\n");
                scsi_set_resid(srb, transfer_length);
                goto skipped_data_phase;
            }
        }

```
&ensp;&ensp;&ensp;&ensp;注释上说，有些device会犯错误，直接跳过了data环节，直接传回CSW状态，这个时候USB storage驱动就得判断分析以下了。

- result == USB_STOR_XFER_SHORT的地方就两个，这个可从interpret_urb_result()得到，一个是实际数据传输长度不等于host预期传输的数据长度；另一个是-EREMOTEIO错误；
- 判断srb->sc_data_direction方向，若是从device到host，则继续下一步；
- transfer_length - scsi_get_resid(srb) == US_BULK_CS_WRAP_LEN：这一步看了半天，才看懂，给出下列代码：
```C
#define US_BULK_CS_WRAP_LEN 13

unsigned int transfer_length = scsi_bufflen(srb);

static inline int scsi_get_resid(struct scsi_cmnd *cmd)
{
    return cmd->sdb.resid;
}

static inline void scsi_set_resid(struct scsi_cmnd *cmd, int resid)
{
    cmd->sdb.resid = resid;
}

scsi_set_resid(srb, scsi_bufflen(srb) - partial);
```
&ensp;&ensp;&ensp;&ensp;主要还是各个实现太分散，顾此忘彼，USB storage驱动在传输完一次sg后，partial为实际的传输长度；transfer_length则是host希望传输/接收的数据长度；所以对于DMA_FROM_DEVICE这种情况，此处的transfer_length - scsi_get_resid(srb)即是这个partial的值。而如果partial等于了US_BULK_CS_WRAP_LEN，即13字节，则需要再做下判断，是data数据就是13字节，还是device误操作把CSW给提前传上来了，而没有穿个0长度包。

&ensp;&ensp;&ensp;&ensp;继续代码

- 使用usb_stor_access_xfer_buf()函数从srb的transfer buffer中取出US_BULK_CS_WRAP_LEN长度的数据（bcs本来就只有13字节的数据嘛），并赋值到bcs中；
- 如果取到的数据确实等于US_BULK_CS_WRAP_LEN，则继续；
- 确认bcs->Signature的数值是不是US_BULK_CS_SIGN，根据手册，得用cpu_to_le32()转换一下，如果正确，则铁定CSW无疑；
- 重新调用scsi_set_resid()设置，并使用goto语言跳到后面skipped_data_phase段，后续再讲解；

---

###### 2.7.4.2.5 get CSW for device status
&ensp;&ensp;&ensp;&ensp;手册第15页的流程图对此部分代码做了解析，再次贴出该流程和接下来的代码部分如下：

![](images/status_Transport_Flow.png)

```C
    /*
     * See flow chart on pg 15 of the Bulk Only Transport spec for
     * an explanation of how this code works.
     */

    /* get CSW for device status */
    usb_stor_dbg(us, "Attempting to get CSW...\n");
    result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, &cswlen);

    /*
     * Some broken devices add unnecessary zero-length packets to the
     * end of their data transfers.  Such packets show up as 0-length
     * CSWs.  If we encounter such a thing, try to read the CSW again.
     */
    if (result == USB_STOR_XFER_SHORT && cswlen == 0) {
        usb_stor_dbg(us, "Received 0-length CSW; retrying...\n");
        result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, &cswlen);
    }

    /* did the attempt to read the CSW fail? */
    if (result == USB_STOR_XFER_STALLED) {

        /* get the status again */
        usb_stor_dbg(us, "Attempting to get CSW (2nd try)...\n");
        result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, NULL);
    }

    /* if we still have a failure at this point, we're in trouble */
    usb_stor_dbg(us, "Bulk status result = %d\n", result);
    if (result != USB_STOR_XFER_GOOD)
        return USB_STOR_TRANSPORT_ERROR;
...
```
下面根据流程图来逐步解析代码：

- 使用usb_stor_bulk_transfer_buf()获取CSW，此处用的是urb request，各函数之前已解析过，代码实现如下：
```C
/*
 * Transfer one buffer via bulk pipe, without timeouts, but allowing early
 * termination.  Return codes are USB_STOR_XFER_xxx.  If the bulk pipe
 * stalls during the transfer, the halt is automatically cleared.
 */
int usb_stor_bulk_transfer_buf(struct us_data *us, unsigned int pipe,
    void *buf, unsigned int length, unsigned int *act_len)
{
    int result;

    usb_stor_dbg(us, "xfer %u bytes\n", length);

    /* fill and submit the URB */
    usb_fill_bulk_urb(us->current_urb, us->pusb_dev, pipe, buf, length,
              usb_stor_blocking_completion, NULL);
    result = usb_stor_msg_common(us, 0);

    /* store the actual length of the data transferred */
    if (act_len)
        *act_len = us->current_urb->actual_length;
    return interpret_urb_result(us, pipe, length, result,
            us->current_urb->actual_length);
}
EXPORT_SYMBOL_GPL(usb_stor_bulk_transfer_buf);
```
&ensp;&ensp;&ensp;&ensp;继续将传输结果赋给result；

- 一种特例。某些device会在data传输阶段完成后，还发一个多余的0长度包，如果是，则再重新读取一次CSW；
```C
    /*
     * Some broken devices add unnecessary zero-length packets to the
     * end of their data transfers.  Such packets show up as 0-length
     * CSWs.  If we encounter such a thing, try to read the CSW again.
     */
    if (result == USB_STOR_XFER_SHORT && cswlen == 0) {
        usb_stor_dbg(us, "Received 0-length CSW; retrying...\n");
        result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, &cswlen);
    }
```

- 如果result为USB_STOR_XFER_STALLED，继续重读一次CSW；
```C
    /* did the attempt to read the CSW fail? */
    if (result == USB_STOR_XFER_STALLED) {

        /* get the status again */
        usb_stor_dbg(us, "Attempting to get CSW (2nd try)...\n");
        result = usb_stor_bulk_transfer_buf(us, us->recv_bulk_pipe,
                bcs, US_BULK_CS_WRAP_LEN, NULL);
    }
```

- 如果二次读CSW还是出错，则该开始检测错误了；
```C
    /* if we still have a failure at this point, we're in trouble */
    usb_stor_dbg(us, "Bulk status result = %d\n", result);
    if (result != USB_STOR_XFER_GOOD)
        return USB_STOR_TRANSPORT_ERROR;
```

###### 2.7.4.2.6 check bulk status
&ensp;&ensp;&ensp;&ensp;特别将状态检测做一小节来解析；

- 先拿到bcs的residue：le32_to_cpu(bcs->Residue)；
- 判断一下有没有逻辑错误：即bcs->Tag与之前的bcb->Tag是否相等；us->fflags有没有设置US_FL_BULK_IGNORE_TAG标记；bcs->Status的值是否大于US_BULK_STAT_PHASE，即0x02；
```C
    if (!(bcs->Tag == us->tag || (us->fflags & US_FL_BULK_IGNORE_TAG)) ||
        bcs->Status > US_BULK_STAT_PHASE) {
        usb_stor_dbg(us, "Bulk logical error\n");
        return USB_STOR_TRANSPORT_ERROR;
    }
```
&ensp;&ensp;&ensp;&ensp;此处只有在bcs->Tag等于bcb->Tag；us->fflags没有设置US_FL_BULK_IGNORE_TAG标记；bcs->Status小于等于US_BULK_STAT_PHASE时才不会进if语句；

- 对于某些broken devices会上报一个odd（古怪的）signatures，所以此处没有硬性按照spec来规划代码，而是在第一次传输时记下bcs->Signature，赋值给us->bcs_signature，然后在之后的传输中就只和us->bcs_signature比较即可，不同，则表示Signature mismatch；
```C
    /*
     * Some broken devices report odd signatures, so we do not check them
     * for validity against the spec. We store the first one we see,
     * and check subsequent transfers for validity against this signature.
     */
    if (!us->bcs_signature) {
        us->bcs_signature = bcs->Signature;
        if (us->bcs_signature != cpu_to_le32(US_BULK_CS_SIGN))
            usb_stor_dbg(us, "Learnt BCS signature 0x%08X\n",
                     le32_to_cpu(us->bcs_signature));
    } else if (bcs->Signature != us->bcs_signature) {
        usb_stor_dbg(us, "Signature mismatch: got %08X, expecting %08X\n",
                 le32_to_cpu(bcs->Signature),
                 le32_to_cpu(us->bcs_signature));
        return USB_STOR_TRANSPORT_ERROR;
    }
```

- 计算精确的residue值；（此处不是很理解，后续再更新；）
```C
    /*
     * try to compute the actual residue, based on how much data
     * was really transferred and what the device tells us
     */
    if (residue && !(us->fflags & US_FL_IGNORE_RESIDUE)) {

        /*
         * Heuristically detect devices that generate bogus residues
         * by seeing what happens with INQUIRY and READ CAPACITY
         * commands.
         */
        if (bcs->Status == US_BULK_STAT_OK &&
                scsi_get_resid(srb) == 0 &&
                    ((srb->cmnd[0] == INQUIRY &&
                        transfer_length == 36) ||
                    (srb->cmnd[0] == READ_CAPACITY &&
                        transfer_length == 8))) {
            us->fflags |= US_FL_IGNORE_RESIDUE;

        } else {
            residue = min(residue, transfer_length);
            scsi_set_resid(srb, max(scsi_get_resid(srb),
                                                   (int) residue));
        }
    }
```
- 通过bcs->Status值，返回不同的处理结果；
```C
    /* based on the status code, we report good or bad */
    switch (bcs->Status) {
        case US_BULK_STAT_OK:
            /* device babbled -- return fake sense data */
            if (fake_sense) {
                memcpy(srb->sense_buffer,
                       usb_stor_sense_invalidCDB,
                       sizeof(usb_stor_sense_invalidCDB));
                return USB_STOR_TRANSPORT_NO_SENSE;
            }

            /* command good -- note that data could be short */
            return USB_STOR_TRANSPORT_GOOD;

        case US_BULK_STAT_FAIL:
            /* command failed */
            return USB_STOR_TRANSPORT_FAILED;

        case US_BULK_STAT_PHASE:
            /*
             * phase error -- note that a transport reset will be
             * invoked by the invoke_transport() function
             */
            return USB_STOR_TRANSPORT_ERROR;
    }
```
&ensp;&ensp;&ensp;&ensp;CSW返回的bCSWStatus段，只会有四种状态：

Value | Description
--|--
00h|Command Passed ("good status")
01h|Command Failed
02h|Phase Error
03h and 04h|Reserved (Obsolete)
05h to FFh|Reserved

&ensp;&ensp;&ensp;&ensp;其中大于02h的情况已经被拦截，所以此处只需要讨论前面三种状态了；另外在bcs->Status的US_BULK_STAT_OK下，还存在一种fake_sense=1的可能，即在data阶段从device读数据时，device多发了数据给host，此处定义device babbled，用fake_sense表示，当是这种情况时，需要返回给host一个封装好的无效cdb命令结果。所以就在此处捏造一个处理结果：
```C
        case US_BULK_STAT_OK:
            /* device babbled -- return fake sense data */
            if (fake_sense) {
                memcpy(srb->sense_buffer,
                       usb_stor_sense_invalidCDB,
                       sizeof(usb_stor_sense_invalidCDB));
                return USB_STOR_TRANSPORT_NO_SENSE;
            }
```
&ensp;&ensp;&ensp;&ensp;其中usb_stor_sense_invalidCDB()结构定义在drivers/usb/storage/scsiglue.c中，实现如下：
```C
/* To Report "Illegal Request: Invalid Field in CDB */
unsigned char usb_stor_sense_invalidCDB[18] = {
    [0] = 0x70,             /* current error */
    [2] = ILLEGAL_REQUEST,      /* Illegal Request = 0x05 */
    [7] = 0x0a,             /* additional length */
    [12]    = 0x24              /* Invalid Field in CDB */
};
EXPORT_SYMBOL_GPL(usb_stor_sense_invalidCDB);
```
&ensp;&ensp;&ensp;&ensp;这是一个字符数组，共18个元素，初始化的时候其中4个元素被赋值。这个结构是按照SCSI协议的sense data的格式来定义的，原理则是如果一个设备接收到一个Request Sense命令，那么它将按规则返回一个sense data。

&ensp;&ensp;&ensp;&ensp;此处的意思是将usb_stor_sense_invalidCDB数组拷贝给srb->sense_buffer，然后返回USB_STOR_TRANSPORT_NO_SENSE结果，struct scsi_cmnd结构体内对sense_buffer元素的定义及解释如下：
```C
#define SCSI_SENSE_BUFFERSIZE   96
    unsigned char *sense_buffer;
                /* obtained by REQUEST SENSE when
                 * CHECK CONDITION is received on original
                 * command (auto-sense) */
```
&ensp;&ensp;&ensp;&ensp;SCSI协议里面规定了，当一个SCSI命令执行出了错，可以再发送一个REQUEST SENSE命令给目标device，然后它会返回一些信息，即sense data，而这部分逻辑被放在了底层驱动中来实现，因为某些scsi host卡可以自动发送REQUEST SENSE命令去了解详情，所以为了统一，就让内核底层驱动来判断了。这个即后续的need_auto_sense变量。

- 最后，为了代码的严谨，在usb_stor_Bulk_transport()函数最末尾还加了一句无条件返回，当然，除非特殊中的特殊，代码一般不会执行到那里。
```C
    /* we should never get here, but if we do, we're in trouble */
    return USB_STOR_TRANSPORT_ERROR;
```

---
&ensp;&ensp;&ensp;&ensp;至此，usb_stor_Bulk_transport()函数解析完成；回到usb_stor_transparent_scsi_command()继续跟进。

##### 2.7.4.3 usb_stor_invoke_transport()函数分析
&ensp;&ensp;&ensp;&ensp;usb_stor_transparent_scsi_command()直接封装了usb_stor_invoke_transport()；上一小节介绍了usb_stor_invoke_transport()函数中的us->transport(srb, us)实现过程；现在继续分小段来跟踪代码：

###### 2.7.4.3.1 result结果分析
```C
    result = us->transport(srb, us);

    /*
     * if the command gets aborted by the higher layers, we need to
     * short-circuit all other processing
     */
    if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
        usb_stor_dbg(us, "-- command was aborted\n");
        srb->result = DID_ABORT << 16;
        goto Handle_Errors;
    }

    /* if there is a transport error, reset and don't auto-sense */
    if (result == USB_STOR_TRANSPORT_ERROR) {
        usb_stor_dbg(us, "-- transport indicates error, resetting\n");
        srb->result = DID_ERROR << 16;
        goto Handle_Errors;
    }

    /* if the transport provided its own sense data, don't auto-sense */
    if (result == USB_STOR_TRANSPORT_NO_SENSE) {
        srb->result = SAM_STAT_CHECK_CONDITION;
        last_sector_hacks(us, srb);
        return;
    }

    srb->result = SAM_STAT_GOOD;
```
&ensp;&ensp;&ensp;&ensp;result结果是从us->transport(srb, us)返回的，它只会返回四个结果：USB_STOR_TRANSPORT_NO_SENSE、USB_STOR_TRANSPORT_GOOD、USB_STOR_TRANSPORT_FAILED、USB_STOR_TRANSPORT_ERROR。

- 首先检查us->dflags的US_FLIDX_TIMED_OUT标记；
- 继而检查result的USB_STOR_TRANSPORT_ERROR情况，如果传输错误，则进入Handle_Errors流程，直接reset；
- result的USB_STOR_TRANSPORT_NO_SENSE情况，设置srb->result为SAM_STAT_CHECK_CONDITION，SCSI规范中定义的状态码，表明有sense data被放置在相应的buffer里，SCSI core层就会去读sense buffer。last_sector_hacks()函数是解决最后扇页相关BUG的，与此处的USB storage无关（主要是目前结合USB storage还看不懂该函数实现）；
- 先默认给srb->result赋值SAM_STAT_GOOD；

###### 2.7.4.3.2 result和need_auto_sense联合分析
&ensp;&ensp;&ensp;&ensp;need_auto_sense变量是用来进一步确认是否需要发送REQUEST SENSE命令给device。此处是在判断是否需要设置该参数：
```C
    /*
     * Determine if we need to auto-sense
     *
     * I normally don't use a flag like this, but it's almost impossible
     * to understand what's going on here if I don't.
     */
    need_auto_sense = 0;

    /*
     * If we're running the CB transport, which is incapable
     * of determining status on its own, we will auto-sense
     * unless the operation involved a data-in transfer.  Devices
     * can signal most data-in errors by stalling the bulk-in pipe.
     */
    if ((us->protocol == USB_PR_CB || us->protocol == USB_PR_DPCM_USB) &&
            srb->sc_data_direction != DMA_FROM_DEVICE) {
        usb_stor_dbg(us, "-- CB transport device requiring auto-sense\n");
        need_auto_sense = 1;
    }

    /*
     * If we have a failure, we're going to do a REQUEST_SENSE 
     * automatically.  Note that we differentiate between a command
     * "failure" and an "error" in the transport mechanism.
     */
    if (result == USB_STOR_TRANSPORT_FAILED) {
        usb_stor_dbg(us, "-- transport indicates command failure\n");
        need_auto_sense = 1;
    }
```
&ensp;&ensp;&ensp;&ensp;里面的注释也是非常有趣的，先初始化need_auto_sense为0，然后分别对各种情况做判断：

- 第一种情况是在CB传输时，设置need_auto_sense为1，注释部分已解释得很清楚了，目前先忽略掉该部分代码；
- 另一种则是在传输FAILED时，设置need_auto_sense为1；

###### 2.7.4.3.3 处理need_auto_sense前判断
```C
    /*
     * Determine if this device is SAT by seeing if the
     * command executed successfully.  Otherwise we'll have
     * to wait for at least one CHECK_CONDITION to determine
     * SANE_SENSE support
     */
    if (unlikely((srb->cmnd[0] == ATA_16 || srb->cmnd[0] == ATA_12) &&
        result == USB_STOR_TRANSPORT_GOOD &&
        !(us->fflags & US_FL_SANE_SENSE) &&
        !(us->fflags & US_FL_BAD_SENSE) &&
        !(srb->cmnd[2] & 0x20))) {
        usb_stor_dbg(us, "-- SAT supported, increasing auto-sense\n");
        us->fflags |= US_FL_SANE_SENSE;
    }

    /*
     * A short transfer on a command where we don't expect it
     * is unusual, but it doesn't mean we need to auto-sense.
     */
    if ((scsi_get_resid(srb) > 0) &&
        !((srb->cmnd[0] == REQUEST_SENSE) ||
          (srb->cmnd[0] == INQUIRY) || 
          (srb->cmnd[0] == MODE_SENSE) ||
          (srb->cmnd[0] == LOG_SENSE) ||
          (srb->cmnd[0] == MODE_SENSE_10))) {
        usb_stor_dbg(us, "-- unexpectedly short transfer\n");
    }
```

- 如果device为SAT设备，且result为USB_STOR_TRANSPORT_GOOD，而us->fflags没有设置US_FL_SANE_SENSE和US_FL_BAD_SENSE等，设置us->fflags增加US_FL_SANE_SENSE标记（此处这个0x20尚未完全理解）；
- 某些命令，返回short transfer结果时，即命令希望传输n个字节，结果device反馈回n-m各字节，但是对于以下五种命令来说，也无伤大雅，host暂时忍让，不需要auto-sense，继续代码；

###### 2.7.4.3.4 sense data结构简介
&ensp;&ensp;&ensp;&ensp;在讲解need_auto_sense情况之前，我们还是先简单学习一下SENSE DATA数据结构，该结构由SCSI手册定义，手册名：SCSI Primary Commands，当前最新为第5版，全称：spc5r18.pdf[^sample_4]

&ensp;&ensp;&ensp;&ensp;spc5r18.pdf手册的4.4节详细介绍了sense data结构，分为两类：fixed format和descriptor format。两种结构存在共同元素，但是所在结构中的位置不一致，需要分别取出来进行处理。先贴出两个结构的格式图：

- Descriptor format sense data
![](images/descriptor_format_sense_data.png)

- Fixed format sense data
![](images/fixed_format_sense_data.png)

&ensp;&ensp;&ensp;&ensp;内核里面用struct scsi_sense_hdr结构来对应sense data内通用元素，该结构定义如下：
```C
/*
 * This is a slightly modified SCSI sense "descriptor" format header.
 * The addition is to allow the 0x70 and 0x71 response codes. The idea
 * is to place the salient data from either "fixed" or "descriptor" sense
 * format into one structure to ease application processing.
 *
 * The original sense buffer should be kept around for those cases
 * in which more information is required (e.g. the LBA of a MEDIUM ERROR).
 */
struct scsi_sense_hdr {     /* See SPC-3 section 4.5 */
    u8 response_code;   /* permit: 0x0, 0x70, 0x71, 0x72, 0x73 */
    u8 sense_key;
    u8 asc;
    u8 ascq;
    u8 byte4;
    u8 byte5;
    u8 byte6;
    u8 additional_length;   /* always 0 for fixed sense format */
};
```

&ensp;&ensp;&ensp;&ensp;因为后续代码会用到sense_key、asc、ascq三个字段，分别对应手册中的SENSE KEY、ADDITIONAL SENSE CODE、ADDITIONAL SENSE CODE QUALIFIER字段，这三个字段提供了一个分层的信息，这个分层结构为应用程序客户端提供一个自顶向下的方法，以决定信息的报告状态。下面简单介绍下这三个元素的作用：

- sense_key：标示描述报告状态的通用信息；
- asc：标示sense_key字段的（描述报告状态的）附加信息，为可选字段；
- ascq：标示asc字段中（描述报告状态信息的）的详细信息，为可选字段；


###### 2.7.4.3.5 auto_sense处理
&ensp;&ensp;&ensp;&ensp;判断了各类情况后，下面开始对need_auto_sense为1的情况进行处理；
```C
    /* Now, if we need to do the auto-sense, let's do it */
    if (need_auto_sense) {
        int temp_result;
        struct scsi_eh_save ses;
        int sense_size = US_SENSE_SIZE;
        struct scsi_sense_hdr sshdr;
        const u8 *scdd;
        u8 fm_ili;

        /* device supports and needs bigger sense buffer */
        if (us->fflags & US_FL_SANE_SENSE)
            sense_size = ~0;
Retry_Sense:
        usb_stor_dbg(us, "Issuing auto-REQUEST_SENSE\n");

        scsi_eh_prep_cmnd(srb, &ses, NULL, 0, sense_size);

        /* FIXME: we must do the protocol translation here */
        if (us->subclass == USB_SC_RBC || us->subclass == USB_SC_SCSI ||
                us->subclass == USB_SC_CYP_ATACB)
            srb->cmd_len = 6;
        else
            srb->cmd_len = 12;

        /* issue the auto-sense command */
        scsi_set_resid(srb, 0);
        temp_result = us->transport(us->srb, us);

        /* let's clean up right away */
        scsi_eh_restore_cmnd(srb, &ses);

        if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
            usb_stor_dbg(us, "-- auto-sense aborted\n");
            srb->result = DID_ABORT << 16;

            /* If SANE_SENSE caused this problem, disable it */
            if (sense_size != US_SENSE_SIZE) {
                us->fflags &= ~US_FL_SANE_SENSE;
                us->fflags |= US_FL_BAD_SENSE;
            }
            goto Handle_Errors;
        }

        /*
         * Some devices claim to support larger sense but fail when
         * trying to request it. When a transport failure happens
         * using US_FS_SANE_SENSE, we always retry with a standard
         * (small) sense request. This fixes some USB GSM modems
         */
        if (temp_result == USB_STOR_TRANSPORT_FAILED &&
                sense_size != US_SENSE_SIZE) {
            usb_stor_dbg(us, "-- auto-sense failure, retry small sense\n");
            sense_size = US_SENSE_SIZE;
            us->fflags &= ~US_FL_SANE_SENSE;
            us->fflags |= US_FL_BAD_SENSE;
            goto Retry_Sense;
        }

        /* Other failures */
        if (temp_result != USB_STOR_TRANSPORT_GOOD) {
            usb_stor_dbg(us, "-- auto-sense failure\n");

            /*
             * we skip the reset if this happens to be a
             * multi-target device, since failure of an
             * auto-sense is perfectly valid
             */
            srb->result = DID_ERROR << 16;
            if (!(us->fflags & US_FL_SCM_MULT_TARG))
                goto Handle_Errors;
            return;
        }

        /*
         * If the sense data returned is larger than 18-bytes then we
         * assume this device supports requesting more in the future.
         * The response code must be 70h through 73h inclusive.
         */
        if (srb->sense_buffer[7] > (US_SENSE_SIZE - 8) &&
            !(us->fflags & US_FL_SANE_SENSE) &&
            !(us->fflags & US_FL_BAD_SENSE) &&
            (srb->sense_buffer[0] & 0x7C) == 0x70) {
            usb_stor_dbg(us, "-- SANE_SENSE support enabled\n");
            us->fflags |= US_FL_SANE_SENSE;

            /*
             * Indicate to the user that we truncated their sense
             * because we didn't know it supported larger sense.
             */
            usb_stor_dbg(us, "-- Sense data truncated to %i from %i\n",
                     US_SENSE_SIZE,
                     srb->sense_buffer[7] + 8);
            srb->sense_buffer[7] = (US_SENSE_SIZE - 8);
        }

        scsi_normalize_sense(srb->sense_buffer, SCSI_SENSE_BUFFERSIZE,
                     &sshdr);

        usb_stor_dbg(us, "-- Result from auto-sense is %d\n",
                 temp_result);
        usb_stor_dbg(us, "-- code: 0x%x, key: 0x%x, ASC: 0x%x, ASCQ: 0x%x\n",
                 sshdr.response_code, sshdr.sense_key,
                 sshdr.asc, sshdr.ascq);
#ifdef CONFIG_USB_STORAGE_DEBUG
        usb_stor_show_sense(us, sshdr.sense_key, sshdr.asc, sshdr.ascq);
#endif

        /* set the result so the higher layers expect this data */
        srb->result = SAM_STAT_CHECK_CONDITION;

        scdd = scsi_sense_desc_find(srb->sense_buffer,
                        SCSI_SENSE_BUFFERSIZE, 4);
        fm_ili = (scdd ? scdd[3] : srb->sense_buffer[2]) & 0xA0;

        /*
         * We often get empty sense data.  This could indicate that
         * everything worked or that there was an unspecified
         * problem.  We have to decide which.
         */
        if (sshdr.sense_key == 0 && sshdr.asc == 0 && sshdr.ascq == 0 &&
            fm_ili == 0) {
            /*
             * If things are really okay, then let's show that.
             * Zero out the sense buffer so the higher layers
             * won't realize we did an unsolicited auto-sense.
             */
            if (result == USB_STOR_TRANSPORT_GOOD) {
                srb->result = SAM_STAT_GOOD;
                srb->sense_buffer[0] = 0x0;
            }

            /*
             * ATA-passthru commands use sense data to report
             * the command completion status, and often devices
             * return Check Condition status when nothing is
             * wrong.
             */
            else if (srb->cmnd[0] == ATA_16 ||
                    srb->cmnd[0] == ATA_12) {
                /* leave the data alone */
            }

            /*
             * If there was a problem, report an unspecified
             * hardware error to prevent the higher layers from
             * entering an infinite retry loop.
             */
            else {
                srb->result = DID_ERROR << 16;
                if ((sshdr.response_code & 0x72) == 0x72)
                    srb->sense_buffer[1] = HARDWARE_ERROR;
                else
                    srb->sense_buffer[2] = HARDWARE_ERROR;
            }
        }
    }
```
&ensp;&ensp;&ensp;&ensp;此处的主要工作就是发送REQUEST SENSE命令，这个逻辑和之前一样的，即再进行一次Bulk传输，还是三个阶段；现准备一个struct scsi_cmnd，调用us->transport(us->srb, us)，然后结束后检查结果。但是此处并没有再申请一个新的struct scsi_cmnd结构，而是沿用了之前的us->srb，只是在用之前调用scsi_eh_prep_cmnd()函数，将srb的一些关键信息保存在struct scsi_eh_save结构实例ses中，等调用完us->transport(us->srb, us)时候，再调用scsi_eh_restore_cmnd()恢复回来。

&ensp;&ensp;&ensp;&ensp;中间还对USB_SC_RBC/USB_SC_SCSI/USB_SC_CYP_ATACB类设备做了限定，即对于该类设备，REQUEST SENSE长度只需要6字节，而其它的设备该命令长度为12字节。

&ensp;&ensp;&ensp;&ensp;在分析temp_result结果时，先确认下us->dflags的US_FLIDX_TIMED_OUT标记，并且对存在US_FL_SANE_SENSE的情况进行了分析；即是否是因为Sane Sense大于18字节，才导致了此超时，如果是，那么就没必要继续分析了，直接跳到Handle_Errors流程。

&ensp;&ensp;&ensp;&ensp;对于大于18字节的sense导致的failed的情况，如注释所说，驱动可以再重新发送一次标准的sense request命令，这能修复部分USB GSM模型；
```C
        /*
         * Some devices claim to support larger sense but fail when
         * trying to request it. When a transport failure happens
         * using US_FS_SANE_SENSE, we always retry with a standard
         * (small) sense request. This fixes some USB GSM modems
         */
        if (temp_result == USB_STOR_TRANSPORT_FAILED &&
                sense_size != US_SENSE_SIZE) {
            usb_stor_dbg(us, "-- auto-sense failure, retry small sense\n");
            sense_size = US_SENSE_SIZE;
            us->fflags &= ~US_FL_SANE_SENSE;
            us->fflags |= US_FL_BAD_SENSE;
            goto Retry_Sense;
        }
```

&ensp;&ensp;&ensp;&ensp;对于其它错误的分析，即说明连REQUEST SENSE命令也failed了，于是设置srb->result = DID_ERROR << 16；此处需要做个进一步判断了，如果device没有设置US_FL_SCM_MULT_TARG标记，即没有多个target的话，那可以直接reset，但是如果是多个target的情况，则就不能这么简单的reset了。
```C
        /* Other failures */
        if (temp_result != USB_STOR_TRANSPORT_GOOD) {
            usb_stor_dbg(us, "-- auto-sense failure\n");

            /*
             * we skip the reset if this happens to be a
             * multi-target device, since failure of an
             * auto-sense is perfectly valid
             */
            srb->result = DID_ERROR << 16;
            if (!(us->fflags & US_FL_SCM_MULT_TARG))
                goto Handle_Errors;
            return;
        }
```

&ensp;&ensp;&ensp;&ensp;如果sense data大于18字节，则驱动认为在将来该device还需更大的sense data支持，即此处给它US_FL_SANE_SENSE标记。但是此次只能做截断sense data操作了，所以强迫将srb->sense_buffer[7] = (US_SENSE_SIZE - 8)，此处的减8操作，是因为sense_buffer[7]的值就是n - 7的来的，n为返回的数据。
```C
        /*
         * If the sense data returned is larger than 18-bytes then we
         * assume this device supports requesting more in the future.
         * The response code must be 70h through 73h inclusive.
         */
        if (srb->sense_buffer[7] > (US_SENSE_SIZE - 8) &&
            !(us->fflags & US_FL_SANE_SENSE) &&
            !(us->fflags & US_FL_BAD_SENSE) &&
            (srb->sense_buffer[0] & 0x7C) == 0x70) {
            usb_stor_dbg(us, "-- SANE_SENSE support enabled\n");
            us->fflags |= US_FL_SANE_SENSE;

            /*
             * Indicate to the user that we truncated their sense
             * because we didn't know it supported larger sense.
             */
            usb_stor_dbg(us, "-- Sense data truncated to %i from %i\n",
                     US_SENSE_SIZE,
                     srb->sense_buffer[7] + 8);
            srb->sense_buffer[7] = (US_SENSE_SIZE - 8);
        }
```

&ensp;&ensp;&ensp;&ensp;scsi_normalize_sense()函数，即将srb->sense_buffer中通用sense data元素合成到sshdr变量中。代码实现如下：
```C
/**
 * scsi_normalize_sense - normalize main elements from either fixed or
 *          descriptor sense data format into a common format.
 *          
 * @sense_buffer:   byte array containing sense data returned by device
 * @sb_len:     number of valid bytes in sense_buffer
 * @sshdr:      pointer to instance of structure that common
 *          elements are written to.
 *
 * Notes:
 *  The "main elements" from sense data are: response_code, sense_key,
 *  asc, ascq and additional_length (only for descriptor format).
 *
 *  Typically this function can be called after a device has
 *  responded to a SCSI command with the CHECK_CONDITION status.
 *                   
 * Return value:
 *  true if valid sense data information found, else false;
 */
bool scsi_normalize_sense(const u8 *sense_buffer, int sb_len,
              struct scsi_sense_hdr *sshdr)
{
    memset(sshdr, 0, sizeof(struct scsi_sense_hdr));
          
    if (!sense_buffer || !sb_len)
        return false;
                 
    sshdr->response_code = (sense_buffer[0] & 0x7f);
        
    if (!scsi_sense_valid(sshdr))
        return false;
  
    if (sshdr->response_code >= 0x72) {
        /*  
         * descriptor format
         */
        if (sb_len > 1)
            sshdr->sense_key = (sense_buffer[1] & 0xf);
        if (sb_len > 2)
            sshdr->asc = sense_buffer[2];
        if (sb_len > 3)
            sshdr->ascq = sense_buffer[3];
        if (sb_len > 7)
            sshdr->additional_length = sense_buffer[7];
    } else {
        /*  
         * fixed format
         */
        if (sb_len > 2)
            sshdr->sense_key = (sense_buffer[2] & 0xf);
        if (sb_len > 7) {
            sb_len = (sb_len < (sense_buffer[7] + 8)) ?
                     sb_len : (sense_buffer[7] + 8);
            if (sb_len > 12)
                sshdr->asc = sense_buffer[12];
            if (sb_len > 13)
                sshdr->ascq = sense_buffer[13];
        }
    }

    return true;
}
EXPORT_SYMBOL(scsi_normalize_sense);
```

&ensp;&ensp;&ensp;&ensp;接下来三条打印，可以根据这些信息来进行内核调试：
```C
        usb_stor_dbg(us, "-- Result from auto-sense is %d\n",
                 temp_result);
        usb_stor_dbg(us, "-- code: 0x%x, key: 0x%x, ASC: 0x%x, ASCQ: 0x%x\n",
                 sshdr.response_code, sshdr.sense_key,
                 sshdr.asc, sshdr.ascq);
#ifdef CONFIG_USB_STORAGE_DEBUG
        usb_stor_show_sense(us, sshdr.sense_key, sshdr.asc, sshdr.ascq);
#endif
```

&ensp;&ensp;&ensp;&ensp;预先设置srb->result的状态；
```C
        /* set the result so the higher layers expect this data */
        srb->result = SAM_STAT_CHECK_CONDITION;
```

&ensp;&ensp;&ensp;&ensp;获取sense data中的descriptor format sense data类型的sense data descriptor，此处scsi_sense_desc_find的参数赋值为4，参考手册得到是需要获取Stream commands类型的描述符（SCSI相关目前还未有过研究，此处就不深究了）；fm_ili这个变量的作用在此处的作用也未看懂，后续看懂了补上；
```C
        scdd = scsi_sense_desc_find(srb->sense_buffer,
                        SCSI_SENSE_BUFFERSIZE, 4);
        fm_ili = (scdd ? scdd[3] : srb->sense_buffer[2]) & 0xA0;
```

&ensp;&ensp;&ensp;&ensp;接下来是判断若拿到的是个空的sense data时，则判断各类情况：
```C
        /*
         * We often get empty sense data.  This could indicate that
         * everything worked or that there was an unspecified
         * problem.  We have to decide which.
         */
        if (sshdr.sense_key == 0 && sshdr.asc == 0 && sshdr.ascq == 0 &&
            fm_ili == 0) {
            /*
             * If things are really okay, then let's show that.
             * Zero out the sense buffer so the higher layers
             * won't realize we did an unsolicited auto-sense.
             */
            if (result == USB_STOR_TRANSPORT_GOOD) {
                srb->result = SAM_STAT_GOOD;
                srb->sense_buffer[0] = 0x0;
            }

            /*
             * ATA-passthru commands use sense data to report
             * the command completion status, and often devices
             * return Check Condition status when nothing is
             * wrong.
             */
            else if (srb->cmnd[0] == ATA_16 ||
                    srb->cmnd[0] == ATA_12) {
                /* leave the data alone */
            }

            /*
             * If there was a problem, report an unspecified
             * hardware error to prevent the higher layers from
             * entering an infinite retry loop.
             */
            else {
                srb->result = DID_ERROR << 16;
                if ((sshdr.response_code & 0x72) == 0x72)
                    srb->sense_buffer[1] = HARDWARE_ERROR;
                else
                    srb->sense_buffer[2] = HARDWARE_ERROR;
            }
        }
```

- 如果传输成功，即result == USB_STOR_TRANSPORT_GOOD，则判定此次传输结果为SAM_STAT_GOOD，且将sense_buffer赋值0，表示无需auto-sense；
- 如果cmnd为ATA_16或者ATA_12，则继续；
- 如果有问题，则将错误码写入sense_buffer中的sense_key字段，此处是分类为descriptor format和fixedformat两类sense data的情况；

----
&ensp;&ensp;&ensp;&ensp;如此，则need_auto_sense的情况处理结束，下面进入正常的结果处理及错误流程处理中；

###### 2.7.4.3.6 （罕见）READ_10命令情况处理
&ensp;&ensp;&ensp;&ensp;正如英文注释所说，某些device在处理READ(10)命令时，不能工作或返回错误data等；如果 US_FL_INITIAL_READ10标记也设置了，则保持跟踪以确保READ(10)命令成功。如果失败，则重试；
```C
    /*
     * Some devices don't work or return incorrect data the first
     * time they get a READ(10) command, or for the first READ(10)
     * after a media change.  If the INITIAL_READ10 flag is set,
     * keep track of whether READ(10) commands succeed.  If the
     * previous one succeeded and this one failed, set the REDO_READ10
     * flag to force a retry.
     */
    if (unlikely((us->fflags & US_FL_INITIAL_READ10) &&
            srb->cmnd[0] == READ_10)) {
        if (srb->result == SAM_STAT_GOOD) {
            set_bit(US_FLIDX_READ10_WORKED, &us->dflags);
        } else if (test_bit(US_FLIDX_READ10_WORKED, &us->dflags)) {
            clear_bit(US_FLIDX_READ10_WORKED, &us->dflags);
            set_bit(US_FLIDX_REDO_READ10, &us->dflags);
        }

        /*
         * Next, if the REDO_READ10 flag is set, return a result
         * code that will cause the SCSI core to retry the READ(10)
         * command immediately.
         */
        if (test_bit(US_FLIDX_REDO_READ10, &us->dflags)) {
            clear_bit(US_FLIDX_REDO_READ10, &us->dflags);
            srb->result = DID_IMM_RETRY << 16;
            srb->sense_buffer[0] = 0;
        }
    }
```

###### 2.7.4.3.7 返回前处理
&ensp;&ensp;&ensp;&ensp;在函数返回前，做最后一次判断：
```C
    /* Did we transfer less than the minimum amount required? */
    if ((srb->result == SAM_STAT_GOOD || srb->sense_buffer[2] == 0) &&
            scsi_bufflen(srb) - scsi_get_resid(srb) < srb->underflow)
        srb->result = DID_ERROR << 16;
```
&ensp;&ensp;&ensp;&ensp;确保传输的数据不小于最小需求，即srb->underflow；

&ensp;&ensp;&ensp;&ensp;last_sector_hacks()判断last sector的情况，也是处理某些错误的流程；

&ensp;&ensp;&ensp;&ensp;当检查错误完后，直接返回，即若没有出错的话，srb->result被赋值SAM_STAT_GOOD；

###### 2.7.4.3.8 Error and abort processing
&ensp;&ensp;&ensp;&ensp;如果出错的话，则使用port reset重新同步device，如果这也失败的话，则将device reset；即调用us->transport_reset来完成；
```C
    /*
     * Error and abort processing: try to resynchronize with the device
     * by issuing a port reset.  If that fails, try a class-specific
     * device reset.
     */
  Handle_Errors:

    /*
     * Set the RESETTING bit, and clear the ABORTING bit so that
     * the reset may proceed.
     */
    scsi_lock(us_to_host(us));
    set_bit(US_FLIDX_RESETTING, &us->dflags);
    clear_bit(US_FLIDX_ABORTING, &us->dflags);
    scsi_unlock(us_to_host(us));

    /*
     * We must release the device lock because the pre_reset routine
     * will want to acquire it.
     */
    mutex_unlock(&us->dev_mutex);
    result = usb_stor_port_reset(us);
    mutex_lock(&us->dev_mutex);

    if (result < 0) {
        scsi_lock(us_to_host(us));
        usb_stor_report_device_reset(us);
        scsi_unlock(us_to_host(us));
        us->transport_reset(us);
    }
    clear_bit(US_FLIDX_RESETTING, &us->dflags);
    last_sector_hacks(us, srb);
```
&ensp;&ensp;&ensp;&ensp;此部分当前限于了解一下，后续若真遇到此类BUG时，再继续深究。

##### 2.7.4.4 usb_stor_control_thread()善后处理
&ensp;&ensp;&ensp;&ensp;在处理完us->proto_handler(srb, us)之后，继续回到usb_stor_control_thread()中来，先设置dev忙状态：usb_mark_last_busy(us->pusb_dev)；

&ensp;&ensp;&ensp;&ensp;善后工作如下：

- 上锁处理srb相关，以及各标记位的检测；
```C
        /* lock access to the state */
        scsi_lock(host);

        /* was the command aborted? */
        if (srb->result == DID_ABORT << 16) {
SkipForAbort:
            usb_stor_dbg(us, "scsi command aborted\n");
            srb = NULL; /* Don't call srb->scsi_done() */
        }

        /*
         * If an abort request was received we need to signal that
         * the abort has finished.  The proper test for this is
         * the TIMED_OUT flag, not srb->result == DID_ABORT, because
         * the timeout might have occurred after the command had 
         * already completed with a different result code.
         */
        if (test_bit(US_FLIDX_TIMED_OUT, &us->dflags)) {
            complete(&(us->notify));
        
            /* Allow USB transfers to resume */
            clear_bit(US_FLIDX_ABORTING, &us->dflags);
            clear_bit(US_FLIDX_TIMED_OUT, &us->dflags);
        }

        /* finished working on this command */
        us->srb = NULL;
        scsi_unlock(host);

        /* unlock the device pointers */
        mutex_unlock(&us->dev_mutex);
```

&ensp;&ensp;&ensp;&ensp;清空us->srb为NULL；通过srb->scsi_done通知SCSI core层srb已经处理完成；
```C
        /* now that the locks are released, notify the SCSI core */
        if (srb) {
            usb_stor_dbg(us, "scsi cmd done, result=0x%x\n",
                    srb->result);
            srb->scsi_done(srb);
        }
```

---
&ensp;&ensp;&ensp;&ensp;至此，USB storage的代码流程完成，其中还有很多部分未完成注释，虽然这些部分平时很难涉及；但是一般出现BUG时，其实几本都是些非常冷门的逻辑导致的出错；所以，后续希望还能够继续持续更新；



---

[^sample_1]: 该段摘抄[蜗窝科技](http://www.wowotech.net/irq_subsystem/cmwq-intro.html)博客文章内某段内容。
[^sample_2]: [USB_Storage_spec.md](https://github.com/TGSP/USB-storage-knowledge/blob/master/USB_Storage_spec.md)手册解析文档中的1.2.1.2小节。
[^sample_3]:此处我也没看懂，是直接摘抄了《[Linux那些事儿之我是U盘](https://github.com/TGSP/USB-storage-knowledge/blob/master/Linux_stuff_am_I_a_USB_drive.pdf)》里面的，待后续学习SCSI架构了再来详细理解说明。
[^sample_4]: SCSI手册可参考T10组织官网内发布的手册：http://www.t10.org
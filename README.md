# 1212300229_p4
周奎君，1212300229，Unix系统分析作业4

###1.第一篇
####原文档地址：[http://lxr.oss.org.cn/source/Documentation/driver-model/bus.txt](http://lxr.oss.org.cn/source/Documentation/driver-model/bus.txt)

翻译如下：

```
总线类型

定义
~~~~~~~~~~~~
可以通过内核文档查看总线类型结构体

int bus_register(struct bus_type * bus);

声明
~~~~~~~~~~~~

每个内核里的总线类型都应该定义成一个和下面实例一样的的静态对象。它们必须初始化name字段，当然了，也可以选择性的初始化对应的回掉函数。

struct bus_type pci_bus_type = {
        .name    = "pci",
        .match   = pci_bus_match,
};

该结构应该被导出到一个头文件中的驱动程序：

extern struct bus_type pci_bus_type;

Registration
~~~~~~~~~~~~

当一个总线驱动被初始化，我们管它叫做总线注册。它会初始化该总线对象的剩下的字段并且将该总线对象插入到一个全局的总线类型列表中。一旦总线对象被注册，它里面的字段都可以被总线驱动所使用。

回调
~~~~~~~~~~~~

match(): 附加驱动程序到设备
~~~~~~~~~~~~~~~~~~~~~~~~~~~
设备ID的结构和用于比较它们的语义的格式本质上是总线特定。驱动程序通常声明一组支持驻留在总线专用驱动结构的设备ID数组。

匹配回调的目的是给总线一个机会通过比较驱动支持与特定设备的设备ID以确定是否一个特定的驱动程序支持特定设备，而不牺牲总线特定功能或类型安全。

当一个驱动注册到总线，该总线的设备列表将会被遍历，然后每个没有与驱动所关联的设备的匹配回调函数将会被调用。

设备以及驱动列表
~~~~~~~~~~~~~~

设备和驱动程序的列表是为了替换很多总线保存的本地列表。它们分别是设备和设备驱动的结构列表体。很多总线驱动可以随意使用该列表，但是有些特定类型的结构需要转化。

LDM核心提供了帮助函数来遍历每个列表。

int bus_for_each_dev(struct bus_type * bus, struct device * start, void * data,
                      int (*fn)(struct device *, void *));
 
int bus_for_each_drv(struct bus_type * bus, struct device_driver * start, 
                      void * data, int (*fn)(struct device_driver *, void *));

这些函数遍历各个列表，并且为每个在列表里的设备和驱动调用回调函数，所有的列表获取都是通过总线锁（读当前的）同步获取的，列表中的每个对象上的引用计数之前回调调用递增;已经获得的下一个对象之后被递减。调用回调当锁定不保持。

文件系统
~~~~~~~~
这是最顶层的总线目录。

每个总线在总线目录里都有一个目录以及两个默认的目录：

        /sys/bus/pci/
        |-- devices
        `-- drivers

被总线注册的驱动在总线的驱动目录下会有一个目录：

         /sys/bus/pci/
         |-- devices
         `-- drivers
             |-- Intel ICH
             |-- Intel ICH Joystick
             |-- agpgart
             `-- e100

发现该类型的总线上的每个设备获取总线的设备目录中的物理层次设备的目录中的符号链接：

        /sys/bus/pci/
         |-- devices
         |   |-- 00:00.0 -> ../../../root/pci0/00:00.0
         |   |-- 00:01.0 -> ../../../root/pci0/00:01.0
         |   `-- 00:02.0 -> ../../../root/pci0/00:02.0
         `-- drivers

导出属性
~~~~~~~~~~
struct bus_attribute {
        struct attribute        attr;
        ssize_t (*show)(struct bus_type *, char * buf);
        ssize_t (*store)(struct bus_type *, const char * buf, size_t count);
};

总线驱动可以导出属性通过使用BUS_ATTR宏指令，当然也能在相似的DEVICE_ATTR宏指令中有效。例如，像这样的定义：

static BUS_ATTR(debug,0644,show_debug,store_debug);

这样定义也是相同的：

static bus_attribute bus_attr_debug;

通过下面这样使用可以在总线文件系统目录中添加或者删除属性：

int bus_create_file(struct bus_type *, struct bus_attribute *);
void bus_remove_file(struct bus_type *, struct bus_attribute *);

```

###2.第二篇
####原文档地址：[http://lxr.oss.org.cn/source/Documentation/driver-model/design-patterns.txt](http://lxr.oss.org.cn/source/Documentation/driver-model/design-patterns.txt)

翻译如下：

```
设备驱动程序设计模式
~~~~~~~~~~~~~~~~~~~~

本文档介绍了设备驱动程序的几个常见设计模式。

子系统维护者很有可能要求驱动程序开发人员以符合这些设计模式。

1. 状态容器
2. container_of()

1. 状态容器
~~~~~~~~~~~~~~~~~~

当内核包含了一些假定它们将在一个特定的系统（单例）只被probed()函数执行一次的设备，它将自定义的假设该设备的驱动程序绑定到将出现在几个实例。这意味着probe()函数以及全部的回调函数将被折返。

最常见的方式来实现这个就是通过使用状态容器的设计模式。它通常拥有这样的形式：

struct foo {
     spinlock_t lock; /* 示例成员 */
     (...)
};
 
static int foo_probe(...)
{
    struct foo *foo;
 
    foo = devm_kzalloc(dev, sizeof(*foo), GFP_KERNEL);
    if (!foo)
        return -ENOMEM;
    spin_lock_init(&foo->lock);
    (...)
}

每当probe()函数被调用的时候,它将在内存中创建foo的结构体实例。这是我们的设备驱动程序的这个实例的状态容器。当然了，所有函数周围的状态实例那些需要访问的状态和成员都需要通过。

例如，如果驱动程序正在注册一个中断处理，你应该允许结构体foo的指针，像这样：

static irqreturn_t foo_handler(int irq, void *arg)
 {
     struct foo *foo = arg;
     (...)
 }
 
 static int foo_probe(...)
 {
     struct foo *foo;
 
     (...)
     ret = request_irq(irq, foo_handler, 0, "foo", foo);
 }

 这样，你在中断处理程序时总能得到一个的指针指向foo的正确实例

 2. container_of()
~~~~~~~~~~~~~~~~~~~~
继续上面的例子，我们添加了一个卸载工作：

struct foo {
      spinlock_t lock;
      struct workqueue_struct *wq;
      struct work_struct offload;
      (...)
  };
  
  static void foo_work(struct work_struct *work)
  {
      struct foo *foo = container_of(work, struct foo, offload);
  
      (...)
  }
  
  static irqreturn_t foo_handler(int irq, void *arg)
  {
      struct foo *foo = arg;
  
      queue_work(foo->wq, &foo->offload);
      (...)
  }
  
  static int foo_probe(...)
  {
      struct foo *foo;
  
      foo->wq = create_singlethread_workqueue("foo-wq");
      INIT_WORK(&foo->offload, foo_work);
     (...)
 }

这个设计模式对于高精度定时器或者一些相似的，将会在回调中返回一个指向结构体成员的指针的单个参数。

container_of()是个定义在<linux/kernel.h>的宏

container_of（）的作用是通过对offsetof（）从标准C的宏中允许类似的面向对象的行为，一些简单的减法，以获得一个指向包含结构从指针到成员。注意，被包含的成员不能是一个指针，而是一个实际成员。

我们可以从这里看到我们避免全局的指针指向我们的foo结构体实例，同时要保持的传递到功函数为单个指针参数的个数。

```




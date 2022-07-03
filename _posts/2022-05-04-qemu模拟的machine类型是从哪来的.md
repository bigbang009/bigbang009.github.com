---
layout:     post
title:      qemu模拟的machine类型是从哪来的
subtitle:   qemu模拟的machine类型是从哪来的
date:       2022-05-04
author:     lwk
catalog: true
tags:
    - linux
    - qemu
---

我们知道在定义虚拟机domain时，即使用libvirt对虚拟机实例进行定义时需要指定machine，例如下图所示

![image](https://user-images.githubusercontent.com/36918717/177044909-a128370f-c60a-4469-8a3d-8614a7e4906d.png)


那么qemu是如何选择machine的呢？

对于x86或者x64使用pc_piix.c用于注册machine信息。

我们以pc-i440fx-5.2为例进行分析，即本文中的default machine type

```

Supported machines are:
microvm              microvm (i386)
pc                   Standard PC (i440FX + PIIX, 1996) (alias of pc-i440fx-5.2)
pc-i440fx-5.2        Standard PC (i440FX + PIIX, 1996) (default)
pc-i440fx-5.1        Standard PC (i440FX + PIIX, 1996)
pc-i440fx-5.0        Standard PC (i440FX + PIIX, 1996)
pc-i440fx-4.2        Standard PC (i440FX + PIIX, 1996)
pc-i440fx-4.1        Standard PC (i440FX + PIIX, 1996)
pc-i440fx-4.0        Standard PC (i440FX + PIIX, 1996)
pc-i440fx-3.1        Standard PC (i440FX + PIIX, 1996)
```
在文件hw/i386/pc_piix.c找到pc-i440fx-5.2相关代码如下：


```

static void pc_i440fx_machine_options(MachineClass *m)
{
    PCMachineClass *pcmc = PC_MACHINE_CLASS(m);
    pcmc->default_nic_model = "e1000";

    m->family = "pc_piix";
    m->desc = "Standard PC (i440FX + PIIX, 1996)";
    m->default_machine_opts = "firmware=bios-256k.bin";
    m->default_display = "std";
    machine_class_allow_dynamic_sysbus_dev(m, TYPE_RAMFB_DEVICE);
    machine_class_allow_dynamic_sysbus_dev(m, TYPE_VMBUS_BRIDGE);
}

static void pc_i440fx_5_2_machine_options(MachineClass *m)
{
    PCMachineClass *pcmc = PC_MACHINE_CLASS(m);
    pc_i440fx_machine_options(m);
    m->alias = "pc";
    m->is_default = true;
    pcmc->default_cpu_version = 1; 
}

DEFINE_I440FX_MACHINE(v5_2, "pc-i440fx-5.2", NULL,
                      pc_i440fx_5_2_machine_options);
                      
#define DEFINE_I440FX_MACHINE(suffix, name, compatfn, optionfn) \
    static void pc_init_##suffix(MachineState *machine) \
    { \
        void (*compat)(MachineState *m) = (compatfn); \
        if (compat) { \
            compat(machine); \
        } \
        pc_init1(machine, TYPE_I440FX_PCI_HOST_BRIDGE, \
                 TYPE_I440FX_PCI_DEVICE); \
    } \
    DEFINE_PC_MACHINE(suffix, name, pc_init_##suffix, optionfn)

 #define DEFINE_PC_MACHINE(suffix, namestr, initfn, optsfn) \
    static void pc_machine_##suffix##_class_init(ObjectClass *oc, void *data) \
    { \
        MachineClass *mc = MACHINE_CLASS(oc); \
        optsfn(mc); \
        mc->init = initfn; \
    } \
    static const TypeInfo pc_machine_type_##suffix = { \
        .name       = namestr TYPE_MACHINE_SUFFIX, \
        .parent     = TYPE_PC_MACHINE, \
        .class_init = pc_machine_##suffix##_class_init, \
    }; \
    static void pc_machine_init_##suffix(void) \
    { \
        type_register(&pc_machine_type_##suffix); \
    } \
    type_init(pc_machine_init_##suffix)
             
 ```
 
 这段代码其实就是把machine注册到qom系统里面

pc_i440fx_5_2_machine_options函数对于qom的class_init,也就是用处初始化qcom的class。

machine的初始化则在machin的选择部分，即在qemu_init()函数里，执行select_machine()
![image](https://user-images.githubusercontent.com/36918717/177044935-a08dbf93-9390-407f-9f71-afbc2e79b3a3.png)

select_machine()调用find_default_machine()函数选择合适的machine并返回
```
static MachineClass *select_machine(void)
{
    GSList *machines = object_class_get_list(TYPE_MACHINE, false);
    MachineClass *machine_class = find_default_machine(machines);
    const char *optarg;
    QemuOpts *opts;
    Location loc;

    loc_push_none(&loc);

    opts = qemu_get_machine_opts();
    qemu_opts_loc_restore(opts);

    optarg = qemu_opt_get(opts, "type");
    if (optarg) {
        machine_class = machine_parse(optarg, machines);
    }

    if (!machine_class) {
        error_report("No machine specified, and there is no default");
        error_printf("Use -machine help to list supported machines\n");
        exit(1);
    }

    loc_pop(&loc);
    g_slist_free(machines);
    return machine_class;
}
```
函数只要执行的操作有，1 找到默认的Machine类 2 参数是否指定machine，如果没有指定则使用默认的 。

我们主要分析object_class_get_list如何找到MachineClass

```

GSList *object_class_get_list(const char *implements_type,
                              bool include_abstract)
{   
    GSList *list = NULL;

    object_class_foreach(object_class_get_list_tramp,
                         implements_type, include_abstract, &list);
    return list;
}   
void object_class_foreach(void (*fn)(ObjectClass *klass, void *opaque),
                          const char *implements_type, bool include_abstract,
                          void *opaque)
{
    OCFData data = { fn, implements_type, include_abstract, opaque };

    enumerating_types = true;
    g_hash_table_foreach(type_table_get(), object_class_foreach_tramp, &data);
    enumerating_types = false;
}
static void object_class_get_list_tramp(ObjectClass *klass, void *opaque)
{
    GSList **list = opaque;

    *list = g_slist_prepend(*list, klass);
}

```
函数主要是遍历type_table_get() 这个hash table里面所有元素，使用object_class_foreach_tramp函数处理， 注意object_class_foreach_tramp函数首先执行type_initialize(type)初始化列表中的所有type，type_table_get() 这个hash table里面注册了系统的所有qom class的类型信息，并且所有class只会初始化一次

也就是说，第一次执行object_class_foreach_tramp，注册过的所有类型都会被初始化。select_machine函数正式第一次调用object_class_foreach_tramp的地方，所以这里所有类型的machine也被初始化了。

object_class_foreach_tramp初始化了所有对象之后，如果对应的ObjectClasss属于MachineClass或者它的子类,则正是我们要找的Class，交给object_class_get_list_tramp 函数处理。object_class_get_list_tramp函数只是把找到的MachineClass放在了传出参数列表上。这样所有machineclass就被收集到了。

通过GDB调试qemu查看对应参数，如果没有指定machine，确实会获取到default machine

```
gdb --args /data0/src/kvm/qemu-5.2.0/build/qemu-system-x86_64 -s -S -m 512 -hda /data0/src/image/image.qcow2 -nographic
```

![image](https://user-images.githubusercontent.com/36918717/177044975-64d8d603-d8eb-4737-96aa-65916235232c.png)








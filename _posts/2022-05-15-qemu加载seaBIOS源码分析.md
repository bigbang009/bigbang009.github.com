---
layout:     post
title:      qemu加载seaBIOS源码分析
subtitle:   qemu加载seaBIOS源码分析
date:       2022-05-15
author:     lwk
catalog: true
tags:
    - linux
    - qemu
    - seaBIOS
---

虚拟化里对于虚拟机有一个重要的模拟就是BIOS的模拟，目前QEMU针对BIOS的模拟主要是开源的seaBIOS。今天重要介绍qemu是如何加载seaBIOS的。

pc_i440fx_machine_options会设置qemu的firmware为bios-256k.bin，代码如下：
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

```
pc_i440fx_machine_options会在虚拟机注册qemu虚拟机机器类型的时候调用，这是由宏DEFINE_PC_MACHINE指定的。这个在之前的文章qemu模拟machine类型是从哪来的一文中有提到过。

main函数首先会将这个default_machine_opts挂到machine的option lists上，之后从这个option lists上把firmware的值赋值到bios_name，由上可知bios_name为bios-256k.bin
```
 /*   
     * Get the default machine options from the machine if it is not already
     * specified either by the configuration file or by the command line.
     */
    if (machine_class->default_machine_opts) {
        qemu_opts_set_defaults(qemu_find_opts("machine"),
                               machine_class->default_machine_opts, 0);
    }
 ```
 
 BIOS固件的加载是在函数old_pc_system_room_init完成的，调用链为pc_init1->pc_memory_init->pc_system_firmware_init->x86_bios_rom_init。该函数主要完成三项工作：

1）打开文件，得到文件信息，创建一个BIOS MemoryRegon。首先通过qemu_find_file和get_image_size得到文件的路径和大小，大小需要是64KB的整数倍，然后调用memory_region_init_ram初始化一个名为bios的MemoryRegion。memory_region_init_ram会在qemu的地址空间分配bios_size，即256KB大小的地址空间
```
void x86_bios_rom_init(MemoryRegion *rom_memory, bool isapc_ram_fw)
{
    char *filename;
    MemoryRegion *bios, *isa_bios;
    int bios_size, isa_bios_size;
    int ret;

    /* BIOS load */
    if (bios_name == NULL) {
        bios_name = BIOS_FILENAME;
    }    
    filename = qemu_find_file(QEMU_FILE_TYPE_BIOS, bios_name);
    if (filename) {
        bios_size = get_image_size(filename);
    } else {
        bios_size = -1;
    }    
    if (bios_size <= 0 || 
        (bios_size % 65536) != 0) { 
        goto bios_error;
    }    
    bios = g_malloc(sizeof(*bios));
    memory_region_init_ram(bios, NULL, "pc.bios", bios_size, &error_fatal);
    if (!isapc_ram_fw) {
        memory_region_set_readonly(bios, true);
    }    
    ret = rom_add_file_fixed(bios_name, (uint32_t)(-bios_size), -1);
    if (ret != 0) {
    bios_error:
        fprintf(stderr, "qemu: could not load PC BIOS '%s'\n", bios_name);
        exit(1);
    }
    g_free(filename);

    /* map the last 128KB of the BIOS in ISA space */
    isa_bios_size = MIN(bios_size, 128 * KiB);
    isa_bios = g_malloc(sizeof(*isa_bios));
    memory_region_init_alias(isa_bios, NULL, "isa-bios", bios,
                             bios_size - isa_bios_size, isa_bios_size);
    memory_region_add_subregion_overlap(rom_memory,
                                        0x100000 - isa_bios_size,
                                        isa_bios,
                                        1);
    if (!isapc_ram_fw) {
        memory_region_set_readonly(isa_bios, true);
    }

    /* map all the bios at the top of memory */
    memory_region_add_subregion(rom_memory,
                                (uint32_t)(-bios_size),
                                bios);
}
```

2）通过宏rom_add_file_fixed调用rom_add_file打开BIOS固件文件
```

#define rom_add_file_fixed(_f, _a, _i)          \
    rom_add_file(_f, NULL, _a, _i, false, NULL, NULL)
```

```
int rom_add_file(const char *file, const char *fw_dir,
                 hwaddr addr, int32_t bootindex,
                 bool option_rom, MemoryRegion *mr,
                 AddressSpace *as
)
{
    MachineClass *mc = MACHINE_GET_CLASS(qdev_get_machine());
    Rom *rom;
    int rc, fd = -1;
    char devpath[100];

    if (as && mr) {
        fprintf(stderr, "Specifying an Address Space and Memory Region is " \
                "not valid when loading a rom\n");
        /* We haven't allocated anything so we don't need any cleanup */
        return -1;
    }

    rom = g_malloc0(sizeof(*rom));
    rom->name = g_strdup(file);
    rom->path = qemu_find_file(QEMU_FILE_TYPE_BIOS, rom->name);
    rom->as = as;
    if (rom->path == NULL) {
        rom->path = g_strdup(file);
    }
    fd = open(rom->path, O_RDONLY | O_BINARY);
    if (fd == -1) {
        fprintf(stderr, "Could not open option rom '%s': %s\n",
                rom->path, strerror(errno));
        goto err;
    }

    if (fw_dir) {
        rom->fw_dir  = g_strdup(fw_dir);
        rom->fw_file = g_strdup(file);
    }
    rom->addr     = addr;
    rom->romsize  = lseek(fd, 0, SEEK_END);
    if (rom->romsize == -1) {
        fprintf(stderr, "rom: file %-20s: get size error: %s\n",
                rom->name, strerror(errno));
        goto err;
    }

    rom->datasize = rom->romsize;
    rom->data     = g_malloc0(rom->datasize);
    lseek(fd, 0, SEEK_SET);
    rc = read(fd, rom->data, rom->datasize);
    if (rc != rom->datasize) {
        fprintf(stderr, "rom: file %-20s: read error: rc=%d (expected %zd)\n",
                rom->name, rc, rom->datasize);
        goto err;
         }
    close(fd);
    rom_insert(rom);
    if (rom->fw_file && fw_cfg) {
        const char *basename;
        char fw_file_name[FW_CFG_MAX_FILE_PATH];
        void *data;

        basename = strrchr(rom->fw_file, '/');
        if (basename) {
            basename++;
        } else {
            basename = rom->fw_file;
        }
        snprintf(fw_file_name, sizeof(fw_file_name), "%s/%s", rom->fw_dir,
                 basename);
        snprintf(devpath, sizeof(devpath), "/rom@%s", fw_file_name);

        if ((!option_rom || mc->option_rom_has_mr) && mc->rom_file_has_mr) {
            data = rom_set_mr(rom, OBJECT(fw_cfg), devpath, true);
        } else {
            data = rom->data;
        }

        fw_cfg_add_file(fw_cfg, fw_file_name, data, rom->romsize);
    } else {
        if (mr) {
            rom->mr = mr;
            snprintf(devpath, sizeof(devpath), "/rom@%s", file);
        } else {
            snprintf(devpath, sizeof(devpath), "/rom@" TARGET_FMT_plx, addr);
        }
    }

    add_boot_device_path(bootindex, NULL, devpath);
    return 0;

err:
    if (fd != -1)
        close(fd);

    rom_free(rom);
    return -1;
 }
 ```
 rom_add_file的主要作用之一就是分配一个Rom结构体，记录BIOS的一些基本信息调用，rom_insert函数将新分配的Rom挂到一个链表上。Rom的定义如下
 ```
 struct Rom {
    char *name;
    char *path;

    /* datasize is the amount of memory allocated in "data". If datasize is less
     * than romsize, it means that the area from datasize to romsize is filled
     * with zeros.
     */
    size_t romsize;
    size_t datasize;

    uint8_t *data;
    MemoryRegion *mr;
    AddressSpace *as;
    int isrom;
    char *fw_dir;
    char *fw_file;
    GMappedFile *mapped_file;

    bool committed;

    hwaddr addr;
    QTAILQ_ENTRY(Rom) next;
};
```
3）最终一步就是将创建的bios MemoryRegion设置为rom memory的子Region，其offset设置为BIOS的加载地址。QEMU使用的SeaBIOS是256KB，bios_size为0x40000, (uint32_t)-bios_size为0xfffc0000,所以这里是把bios的地址加在到了0xfffc0000处，也就是最靠近4GB地址空间的256KB中。这里还会把创建的isa_bios作为BIOS MemoryRegion的别名，并且放在最靠近1MB的128KB位置。

现在分配了BIOS MemoryRegion，并且设置了其基地址和大小，也将BIOS的数据加载到了内存中，但是只是在rom->data中。需要将BIOS的数据复制到MemoryRegion中，并将虚拟机对应的物理内存映射到QEMU的虚拟内存中。

QEMU会在main函数中调用rom_check_and_register_reset，后者的主要工作是将rom_reset挂到reset_handlers链表上，当虚拟机重置时会调用链表上的每一个函数。

rom_reset最重要的任务就是把存放在rom->data中的BIOS数据复制到BIOS MemoryRegion对应的QEMU进程中分配的虚拟内存中。

```

    if (rom_check_and_register_reset() != 0) { 
        error_report("rom check and register reset failed");
        exit(1);
    }    


int rom_check_and_register_reset(void)
{
    hwaddr addr = 0;
    MemoryRegionSection section;
    Rom *rom;
    AddressSpace *as = NULL;

    QTAILQ_FOREACH(rom, &roms, next) {
        if (rom->fw_file) {
            continue;
        }
        if (!rom->mr) {
            if ((addr > rom->addr) && (as == rom->as)) {
                fprintf(stderr, "rom: requested regions overlap "
                        "(rom %s. free=0x" TARGET_FMT_plx
                        ", addr=0x" TARGET_FMT_plx ")\n",
                        rom->name, addr, rom->addr);
                return -1;
            }
            addr  = rom->addr;
            addr += rom->romsize;
            as = rom->as;
        }
        section = memory_region_find(rom->mr ? rom->mr : get_system_memory(),
                                     rom->addr, 1);
        rom->isrom = int128_nz(section.size) && memory_region_is_rom(section.mr);
        memory_region_unref(section.mr);
    }
    qemu_register_reset(rom_reset, NULL);
    roms_loaded = 1;
    return 0;
}
```
现在BIOS MemoryRegion对应的宿主机QEMU进程虚拟内存已经有了BIOS数据，并且这个MR的地址敌营的虚拟机的物理地址是0xfffc000。

CPU在启动后会初始化各个寄存器的值，其中CS被初始化为0xf000,EIP被初始化为0xfff0,CS的基地址被初始化为0xffff0000。

```

static void x86_cpu_reset(DeviceState *dev)
{
    CPUState *s = CPU(dev);
    X86CPU *cpu = X86_CPU(s);
    X86CPUClass *xcc = X86_CPU_GET_CLASS(cpu);
    CPUX86State *env = &cpu->env;
    target_ulong cr4; 
    uint64_t xcr0;
    int i;

    xcc->parent_reset(dev);

    memset(env, 0, offsetof(CPUX86State, end_reset_fields));

    env->old_exception = -1;

    /* init to reset state */

    env->hflags2 |= HF2_GIF_MASK;
    env->hflags &= ~HF_GUEST_MASK;

    cpu_x86_update_cr0(env, 0x60000010);
    env->a20_mask = ~0x0;
    env->smbase = 0x30000;
    env->msr_smi_count = 0; 

    env->idt.limit = 0xffff;
    env->gdt.limit = 0xffff;
    env->ldt.limit = 0xffff;
    env->ldt.flags = DESC_P_MASK | (2 << DESC_TYPE_SHIFT);
    env->tr.limit = 0xffff;
    env->tr.flags = DESC_P_MASK | (11 << DESC_TYPE_SHIFT);

    cpu_x86_load_seg_cache(env, R_CS, 0xf000, 0xffff0000, 0xffff,
                           DESC_P_MASK | DESC_S_MASK | DESC_CS_MASK |
                           DESC_R_MASK | DESC_A_MASK);
    cpu_x86_load_seg_cache(env, R_DS, 0, 0, 0xffff,
                           DESC_P_MASK | DESC_S_MASK | DESC_W_MASK |
                           DESC_A_MASK);
    cpu_x86_load_seg_cache(env, R_ES, 0, 0, 0xffff,
                           DESC_P_MASK | DESC_S_MASK | DESC_W_MASK |
                           DESC_A_MASK);
    cpu_x86_load_seg_cache(env, R_SS, 0, 0, 0xffff,
                           DESC_P_MASK | DESC_S_MASK | DESC_W_MASK |
                           DESC_A_MASK);
    cpu_x86_load_seg_cache(env, R_FS, 0, 0, 0xffff,
                           DESC_P_MASK | DESC_S_MASK | DESC_W_MASK |
                           DESC_A_MASK);
    cpu_x86_load_seg_cache(env, R_GS, 0, 0, 0xffff,
                           DESC_P_MASK | DESC_S_MASK | DESC_W_MASK |
                           DESC_A_MASK);

    env->eip = 0xfff0;
    env->regs[R_EDX] = env->cpuid_version;
    env->eflags = 0x2;

    /* FPU init */
    for (i = 0; i < 8; i++) {
        env->fptags[i] = 1;
    }
    cpu_set_fpuc(env, 0x37f);

    env->mxcsr = 0x1f80;
    /* All units are in INIT state.  */
    env->xstate_bv = 0;

    env->pat = 0x0007040600070406ULL;
    env->msr_ia32_misc_enable = MSR_IA32_MISC_ENABLE_DEFAULT;
    if (env->features[FEAT_1_ECX] & CPUID_EXT_MONITOR) {
        env->msr_ia32_misc_enable |= MSR_IA32_MISC_ENABLE_MWAIT;
    }

    memset(env->dr, 0, sizeof(env->dr));
    env->dr[6] = DR6_FIXED_1;
    env->dr[7] = DR7_FIXED_1;
    cpu_breakpoint_remove_all(s, BP_CPU);
    cpu_watchpoint_remove_all(s, BP_CPU);

    cr4 = 0;
    xcr0 = XSTATE_FP_MASK;
#ifdef CONFIG_USER_ONLY
    /* Enable all the features for user-mode.  */
    if (env->features[FEAT_1_EDX] & CPUID_SSE) {
        xcr0 |= XSTATE_SSE_MASK;
    }
    for (i = 2; i < ARRAY_SIZE(x86_ext_save_areas); i++) {
        const ExtSaveArea *esa = &x86_ext_save_areas[i];
        if (env->features[esa->feature] & esa->bits) {
            xcr0 |= 1ull << i;
        }
    }

    if (env->features[FEAT_1_ECX] & CPUID_EXT_XSAVE) {
        cr4 |= CR4_OSFXSR_MASK | CR4_OSXSAVE_MASK;
    }
    if (env->features[FEAT_7_0_EBX] & CPUID_7_0_EBX_FSGSBASE) {
        cr4 |= CR4_FSGSBASE_MASK;
    }
#endif
    env->xcr0 = xcr0;
    cpu_x86_update_cr4(env, cr4);

    /*
     * SDM 11.11.5 requires:
     *  - IA32_MTRR_DEF_TYPE MSR.E = 0
     *  - IA32_MTRR_PHYSMASKn.V = 0
     * All other bits are undefined.  For simplification, zero it all.
     */
    env->mtrr_deftype = 0;
    memset(env->mtrr_var, 0, sizeof(env->mtrr_var));
    memset(env->mtrr_fixed, 0, sizeof(env->mtrr_fixed));

    env->interrupt_injected = -1;
    env->exception_nr = -1;
    env->exception_pending = 0;
    env->exception_injected = 0;
    env->exception_has_payload = false;
    env->exception_payload = 0;
    env->nmi_injected = false;
#if !defined(CONFIG_USER_ONLY)
    /* We hard-wire the BSP to the first CPU. */
    apic_designate_bsp(cpu->apic_state, s->cpu_index == 0);

    s->halted = !cpu_is_bsp(cpu);
    if (kvm_enabled()) {
        kvm_arch_reset_vcpu(cpu);
    }
#endif
}

```

从前面的分析可知，0xfffffff0处就是BIOS的最后16个字节的开始处，这里都是一个跳转指令，跳转到BIOS的前面执行。





 
 


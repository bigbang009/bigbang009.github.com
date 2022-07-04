---
layout:     post
title:      KVM之kvm_create_vm函数详解
subtitle:   KVM之kvm_create_vm函数详解
date:       2022-05-21
author:     lwk
catalog: true
tags:
    - linux
    - kvm
---

在上一篇文章中，我们阐述了KVM中对/dev/kvm的调用逻辑，当通过系统调用ioctl(/dev/kvm, KVM_CREATE_VM)时，会通过带哦用kvm_dev_ioctl函数进而调用kvm_dev_ioctl_create_vm
```
static int kvm_dev_ioctl_create_vm(unsigned long type)
{
    int r;
    struct kvm *kvm;
    struct file *file;

    kvm = kvm_create_vm(type);
    if (IS_ERR(kvm))
        return PTR_ERR(kvm);
#ifdef CONFIG_KVM_MMIO
    r = kvm_coalesced_mmio_init(kvm);
    if (r < 0)
        goto put_kvm;
#endif
    r = get_unused_fd_flags(O_CLOEXEC);
    if (r < 0)
        goto put_kvm;

    file = anon_inode_getfile("kvm-vm", &kvm_vm_fops, kvm, O_RDWR);
     if (IS_ERR(file)) {
        put_unused_fd(r);
        r = PTR_ERR(file);
        goto put_kvm;
    }

    /*
     * Don't call kvm_put_kvm anymore at this point; file->f_op is
     * already set, with ->release() being kvm_vm_release().  In error
     * cases it will be called by the final fput(file) and will take
     * care of doing kvm_put_kvm(kvm).
     */
    if (kvm_create_vm_debugfs(kvm, r) < 0) {
        put_unused_fd(r);
        fput(file);
        return -ENOMEM;
    }
    kvm_uevent_notify_change(KVM_EVENT_CREATE_VM, kvm);

    fd_install(r, file);
    return r;
put_kvm:
    kvm_put_kvm(kvm);
    return r;
}

```

kvm_dev_ioctl_create_vm函数的主要作用就是创建虚拟机实例，每个虚拟机实例用一个kvm结构体表示，kvm_coalesced_mmio_init主要用来初始化MMIO，anon_inode_getfile创建一个匿名file,其中file_operations就是kvm_vm_fops，私有数据就是刚刚创建的虚拟机，这个file对应的fs返回给用户态的QEMU，表示一台虚拟机，QEMU之后就可以通过该fd对虚拟机进行操作了。
```
static struct file_operations kvm_vm_fops = {
    .release        = kvm_vm_release,
    .unlocked_ioctl = kvm_vm_ioctl,
    .llseek     = noop_llseek,
    KVM_COMPAT(kvm_vm_compat_ioctl),
};

```
接下来重点分析kvm_create_vm函数，该函数是创建虚拟机的核心函数，部分代码如下：

```
static struct kvm *kvm_create_vm(unsigned long type)
{       
    struct kvm *kvm = kvm_arch_alloc_vm();
    int r = -ENOMEM;
    int i; 
        
    if (!kvm)
        return ERR_PTR(-ENOMEM);
        
    spin_lock_init(&kvm->mmu_lock);
    mmgrab(current->mm);
    kvm->mm = current->mm;
    kvm_eventfd_init(kvm);
    mutex_init(&kvm->lock);
    mutex_init(&kvm->irq_lock);
    mutex_init(&kvm->slots_lock);
    INIT_LIST_HEAD(&kvm->devices);

    BUILD_BUG_ON(KVM_MEM_SLOTS_NUM > SHRT_MAX);
    for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
        struct kvm_memslots *slots = kvm_alloc_memslots();

        if (!slots)
            goto out_err_no_arch_destroy_vm;
        /* Generations must be different for each address space. */
        slots->generation = i;
        rcu_assign_pointer(kvm->memslots[i], slots);
    }

    r = kvm_arch_init_vm(kvm, type);
    if (r)
        goto out_err_no_arch_destroy_vm;

    r = hardware_enable_all();
    if (r)
        goto out_err_no_disable;
    r = kvm_init_mmu_notifier(kvm);
    if (r)
        goto out_err_no_mmu_notifier;

    r = kvm_arch_post_init_vm(kvm);
    if (r)
        goto out_err;

    mutex_lock(&kvm_lock);
    list_add(&kvm->vm_list, &vm_list);
    mutex_unlock(&kvm_lock);

    preempt_notifier_inc();

    return kvm;

```

kvm_arch_alloc_vm分配一个KVM结构体，用来表示一台虚拟机，该函数最终根据CPU类型调用vmx/svm的vmx_vm_alloc或svm_vm_alloc函数，本文以intelCPU为例，因此调用vmx_vm_alloc函数分配内存。


```
static inline struct kvm *kvm_arch_alloc_vm(void)
{   
    return kvm_x86_ops->vm_alloc();
}
 
 static struct kvm_x86_ops vmx_x86_ops __ro_after_init = {
    .cpu_has_kvm_support = cpu_has_kvm_support,
    .disabled_by_bios = vmx_disabled_by_bios,
    .hardware_setup = hardware_setup,
    .hardware_unsetup = hardware_unsetup,
    .check_processor_compatibility = vmx_check_processor_compat,
    .hardware_enable = hardware_enable,
    .hardware_disable = hardware_disable,
    .cpu_has_accelerated_tpr = report_flexpriority,
    .has_emulated_msr = vmx_has_emulated_msr,

    .vm_init = vmx_vm_init,
    .vm_alloc = vmx_vm_alloc,
}
 
static struct kvm *vmx_vm_alloc(void)
{
    struct kvm_vmx *kvm_vmx = __vmalloc(sizeof(struct kvm_vmx),
                        GFP_KERNEL_ACCOUNT | __GFP_ZERO,
                        PAGE_KERNEL);
    return &kvm_vmx->kvm;
}
```

kvm->mm最终指向current->mm，即kvm的mm结构指向qemu进程的虚拟内存，也就是虚拟机的物理内存就是qemu进程的虚拟内存。我们用真实的vm实例看下qemu进程的虚拟内存大小是不是虚拟机的物理内存大小.
![image](https://user-images.githubusercontent.com/36918717/177183187-aa8b5545-c5ae-4f73-ad1a-9914c2be6609.png)

![image](https://user-images.githubusercontent.com/36918717/177183198-8d602723-b227-41f9-ab30-4434faffda3d.png)

kvm结构体定义在文件include/linux/kvm_host.h

```
struct kvm {
    spinlock_t mmu_lock;
    struct mutex slots_lock;
    struct mm_struct *mm; /* userspace tied to this vm */
    struct kvm_memslots __rcu *memslots[KVM_ADDRESS_SPACE_NUM];
    struct kvm_vcpu *vcpus[KVM_MAX_VCPUS];

    /*   
     * created_vcpus is protected by kvm->lock, and is incremented
     * at the beginning of KVM_CREATE_VCPU.  online_vcpus is only
     * incremented after storing the kvm_vcpu pointer in vcpus,
     * and is accessed atomically.
     */
    atomic_t online_vcpus;
    int created_vcpus;
    int last_boosted_vcpu;
    struct list_head vm_list;
    struct mutex lock;
    struct kvm_io_bus __rcu *buses[KVM_NR_BUSES];
#ifdef CONFIG_HAVE_KVM_EVENTFD
    struct {
        spinlock_t        lock;
        struct list_head  items;
        struct list_head  resampler_list;
        struct mutex      resampler_lock;
    } irqfds;
    struct list_head ioeventfds;
#endif
    struct kvm_vm_stat stat;
    struct kvm_arch arch;
    refcount_t users_count;
#ifdef CONFIG_KVM_MMIO
    struct kvm_coalesced_mmio_ring *coalesced_mmio_ring;
    spinlock_t ring_lock;
    struct list_head coalesced_zones;
#endif

    struct mutex irq_lock;
#ifdef CONFIG_HAVE_KVM_IRQCHIP
    /*
     * Update side is protected by irq_lock.
     */
    struct kvm_irq_routing_table __rcu *irq_routing;
#endif
#ifdef CONFIG_HAVE_KVM_IRQFD
    struct hlist_head irq_ack_notifier_list;
#endif

#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
    struct mmu_notifier mmu_notifier;
    unsigned long mmu_notifier_seq;
    long mmu_notifier_count;
#endif
    long tlbs_dirty;
    struct list_head devices;
    bool manual_dirty_log_protect;
    struct dentry *debugfs_dentry;
    struct kvm_stat_data **debugfs_stat_data;
    struct srcu_struct srcu;
    struct srcu_struct irq_srcu;
    pid_t userspace_pid;
};

```
KVM有个类型为kvm_arch的arch成员，用于存放与架构相关的数据，kvm_arch_init_vm用来初始化这些数据。紧接着调用hardware_enable_all函数来开启vmx模式，hardware_enable_all在创建第一个虚拟机的时候对每个CPU调用hardware_enable_nolock

```
static int hardware_enable_all(void)
{
    int r = 0; 

    raw_spin_lock(&kvm_count_lock);

    kvm_usage_count++;
    if (kvm_usage_count == 1) { 
        atomic_set(&hardware_enable_failed, 0);
        on_each_cpu(hardware_enable_nolock, NULL, 1);

        if (atomic_read(&hardware_enable_failed)) {
            hardware_disable_all_nolock();
            r = -EBUSY;
        }    
    }    

    raw_spin_unlock(&kvm_count_lock);

    return r;
}

static void hardware_enable_nolock(void *junk)
{
    int cpu = raw_smp_processor_id();
    int r;  
            
    if (cpumask_test_cpu(cpu, cpus_hardware_enabled))
        return;

    cpumask_set_cpu(cpu, cpus_hardware_enabled);

    r = kvm_arch_hardware_enable();

    if (r) {
        cpumask_clear_cpu(cpu, cpus_hardware_enabled);
        atomic_inc(&hardware_enable_failed);
        pr_info("kvm: enabling virtualization on CPU%d failed\n", cpu);
    }
}

int kvm_arch_hardware_enable(void)
{   
    struct kvm *kvm;
    struct kvm_vcpu *vcpu;
    int i;
    int ret;
    u64 local_tsc;
    u64 max_tsc = 0;
    bool stable, backwards_tsc = false;

    kvm_shared_msr_cpu_online();
    ret = kvm_x86_ops->hardware_enable();
    if (ret != 0)
        return ret;
    local_tsc = rdtsc();
    stable = !kvm_check_tsc_unstable();
    list_for_each_entry(kvm, &vm_list, vm_list) {
        kvm_for_each_vcpu(i, vcpu, kvm) {
            if (!stable && vcpu->cpu == smp_processor_id())
                kvm_make_request(KVM_REQ_CLOCK_UPDATE, vcpu);
            if (stable && vcpu->arch.last_host_tsc > local_tsc) {
                backwards_tsc = true;
                if (vcpu->arch.last_host_tsc > max_tsc)
                    max_tsc = vcpu->arch.last_host_tsc;
            }
        }
    }
    if (backwards_tsc) {
        u64 delta_cyc = max_tsc - local_tsc;
        list_for_each_entry(kvm, &vm_list, vm_list) {
            kvm->arch.backwards_tsc_observed = true;
            kvm_for_each_vcpu(i, vcpu, kvm) {
                vcpu->arch.tsc_offset_adjustment += delta_cyc;
                vcpu->arch.last_host_tsc = local_tsc;
                kvm_make_request(KVM_REQ_MASTERCLOCK_UPDATE, vcpu);
            }

            /*
             * We have to disable TSC offset matching.. if you were
             * booting a VM while issuing an S4 host suspend....
             * you may have some problem.  Solving this issue is
             * left as an exercise to the reader.
             */
            kvm->arch.last_tsc_nsec = 0;
            kvm->arch.last_tsc_write = 0;
        }

    }
    return 0;
}
```
同样在kvm_x86_ops->hardware_enable();会调用vmx的hardware_enable函数，该函数主要作用就是设置CR4的VMXE位并且调用VMXON指令开启VMX。
```
static int hardware_enable(void)
{   
    int cpu = raw_smp_processor_id();
    u64 phys_addr = __pa(per_cpu(vmxarea, cpu));
    u64 old, test_bits;
    
    if (cr4_read_shadow() & X86_CR4_VMXE)
        return -EBUSY;

    /*
     * This can happen if we hot-added a CPU but failed to allocate
     * VP assist page for it.
     */
    if (static_branch_unlikely(&enable_evmcs) &&
        !hv_get_vp_assist_page(cpu))
        return -EFAULT;

    INIT_LIST_HEAD(&per_cpu(loaded_vmcss_on_cpu, cpu));
    INIT_LIST_HEAD(&per_cpu(blocked_vcpu_on_cpu, cpu));
    spin_lock_init(&per_cpu(blocked_vcpu_on_cpu_lock, cpu));
    crash_enable_local_vmclear(cpu);

    rdmsrl(MSR_IA32_FEATURE_CONTROL, old);

    test_bits = FEATURE_CONTROL_LOCKED;
    test_bits |= FEATURE_CONTROL_VMXON_ENABLED_OUTSIDE_SMX;
    if (tboot_enabled())
        test_bits |= FEATURE_CONTROL_VMXON_ENABLED_INSIDE_SMX;

    if ((old & test_bits) != test_bits) {
        /* enable and lock */
        wrmsrl(MSR_IA32_FEATURE_CONTROL, old | test_bits);
    }
    kvm_cpu_vmxon(phys_addr);
    if (enable_ept)
        ept_sync_global();

    return 0;
}
```

还有kvm_alloc_memslots以及kvm_init_mmu_notifier函数，这些与内存相关的会放在以后阐述，主要是和内存虚拟化相关。preempt_notifier_inc主要将VCPU线程进行调度。

以上就是对kvm_create_vm函数的阐述，当然里面还有很多细节没有深挖，虚拟化细节相关的我也在逐步了解，有些可能需要等了解到之后再来阐述了。

每天进步一小步，未来进步一大步！













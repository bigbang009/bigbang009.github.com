---
layout:     post
title:      virsh-tool dump虚拟机内存源码分析
subtitle:   virsh-tool dump虚拟机内存源码分析
date:       2022-04-19
author:     lwk
catalog: true
tags:
    - linux
    - virsh-tool
    - dump
    - 虚拟机
---
在之前的文章中，我们在分析虚拟机hung住问题时，尝试使用virsh dump虚拟机内存，以此来通过crash进行分析，因为virsh dump 虚拟机内存实际就是内存快照，和机器在crash时生成的vmcore文件是类似的。

virsh tools源码在libvirt项目的tools目录下，我们找到dump源码所在文件

tools/virsh-domain.c

```

   const vshCmdDef domManagementCmds[]{
       {.name = "dump",
     .handler = cmdDump,
     .opts = opts_dump,
     .info = info_dump,
     .flags = 0
    },
   } 
```

其中，domManagementCmds是个数组，里面包含了所有virsh命令参数信息。

处理dump的核心函数是cmdDump

```
static bool
cmdDump(vshControl *ctl, const vshCmd *cmd) 
{
    virDomainPtr dom;
    bool verbose = false;
    const char *name = NULL; 
    const char *to = NULL; 
    virThread workerThread;
    g_autoptr(GMainContext) eventCtxt = g_main_context_new();
    g_autoptr(GMainLoop) eventLoop = g_main_loop_new(eventCtxt, FALSE);
    virshCtrlData data = {
        .ctl = ctl,
        .cmd = cmd,
        .eventLoop = eventLoop,
        .ret = -1, 
    };    

    if (!(dom = virshCommandOptDomain(ctl, cmd, &name)))
        return false;

    if (vshCommandOptStringReq(ctl, cmd, "file", &to) < 0)
        goto cleanup;

    if (vshCommandOptBool(cmd, "verbose"))
        verbose = true; 

    if (virThreadCreate(&workerThread,
                        true,
                        doDump,
                        &data) < 0)
        goto cleanup;

    virshWatchJob(ctl, dom, verbose, eventLoop,
                  &data.ret, 0, NULL, NULL, _("Dump"));

    virThreadJoin(&workerThread);

    if (!data.ret)
        vshPrintExtra(ctl, _("\nDomain '%s' dumped to %s\n"), name, to);

 cleanup:
    virshDomainFree(dom);
    return !data.ret;
}
```
其中virThreadCreate会创建一个线程，线程函数是doDump，用来执行具体的dump操作，接下来看下doDump函数

```
static void
doDump(void *opaque)
{
    virshCtrlData *data = opaque;
    vshControl *ctl = data->ctl;
    const vshCmd *cmd = data->cmd;
    virDomainPtr dom = NULL;
    const char *name = NULL;
    const char *to = NULL;
    ...
        if (vshCommandOptBool(cmd, "live"))
        flags |= VIR_DUMP_LIVE;
    if (vshCommandOptBool(cmd, "crash"))
        flags |= VIR_DUMP_CRASH;
    if (vshCommandOptBool(cmd, "bypass-cache"))
        flags |= VIR_DUMP_BYPASS_CACHE;
    if (vshCommandOptBool(cmd, "reset"))
        flags |= VIR_DUMP_RESET;
    if (vshCommandOptBool(cmd, "memory-only"))
        flags |= VIR_DUMP_MEMORY_ONLY;
        if (vshCommandOptStringQuiet(ctl, cmd, "format", &format) > 0) {
            if (STREQ(format, "kdump-zlib")) {
                dumpformat = VIR_DOMAIN_CORE_DUMP_FORMAT_KDUMP_ZLIB;
            } else if (STREQ(format, "kdump-lzo")) {
                dumpformat = VIR_DOMAIN_CORE_DUMP_FORMAT_KDUMP_LZO;
            } else if (STREQ(format, "kdump-snappy")) {
                dumpformat = VIR_DOMAIN_CORE_DUMP_FORMAT_KDUMP_SNAPPY;
            } else if (STREQ(format, "elf")) {//dump出的文件格式是ELF
                dumpformat = VIR_DOMAIN_CORE_DUMP_FORMAT_RAW;
            } else {
                vshError(ctl, _("format '%s' is not supported, expecting "
                                "'kdump-zlib', 'kdump-lzo', 'kdump-snappy' "
                                "or 'elf'"), format);
                goto out;
            }
        }
    }
    ....
        if (dumpformat != VIR_DOMAIN_CORE_DUMP_FORMAT_RAW) {
        if (virDomainCoreDumpWithFormat(dom, to, dumpformat, flags) < 0) {
            vshError(ctl, _("Failed to core dump domain '%s' to %s"), name, to);
            goto out;
        }
    } else {
        if (virDomainCoreDump(dom, to, flags) < 0) {
            vshError(ctl, _("Failed to core dump domain '%s' to %s"), name, to);
            goto out;
        }
    }
....
}
```
接下来看下virDomainCoreDump，其中参数dom是指向被dump内存的domain指针，to是dump到具体的位置,一般是文件名，flags是标识。

virDomainCoreDump源码位置在src/libvirt-domain.c文件中
```

int
virDomainCoreDump(virDomainPtr domain, const char *to, unsigned int flags)
{
    virConnectPtr conn;

    VIR_DOMAIN_DEBUG(domain, "to=%s, flags=0x%x", to, flags);

    virResetLastError();

    virCheckDomainReturn(domain, -1);
    conn = domain->conn;
...
    if (conn->driver->domainCoreDump) {
        int ret;
        char *absolute_to;

        /* We must absolutize the file path as the save is done out of process */
        if (virFileAbsPath(to, &absolute_to) < 0) {
            virReportError(VIR_ERR_INTERNAL_ERROR, "%s",
                           _("could not build absolute core file path"));
            goto error;
        }

        ret = conn->driver->domainCoreDump(domain, absolute_to, flags);

        VIR_FREE(absolute_to);

        if (ret < 0)
            goto error;
        return ret;
    }
...
}
```

其中conn->driver->domainCoreDump是具体到对应驱动的函数指针，这里指的是qemu驱动。函数位置在src/qemu/qemu_driver.c

```

   static virHypervisorDriver qemuHypervisorDriver = {
    ...
    .domainCoreDump = qemuDomainCoreDump, /* 0.7.0 */
    .domainCoreDumpWithFormat = qemuDomainCoreDumpWithFormat, /* 1.2.3 */
     ...
}
```

那接下来就是重点看qemuDomainCoreDump函数了

```
static int
qemuDomainCoreDump(virDomainPtr dom,
                   const char *path,
                   unsigned int flags)
{
    return qemuDomainCoreDumpWithFormat(dom, path, 
                                        VIR_DOMAIN_CORE_DUMP_FORMAT_RAW,
                                        flags);
}

```
继续看qemuDomainCoreDumpWithFormat函数

```
static int
qemuDomainCoreDumpWithFormat(virDomainPtr dom,
                             const char *path,
                             unsigned int dumpformat,
                             unsigned int flags)
{
    virQEMUDriverPtr driver = dom->conn->privateData;
    virDomainObjPtr vm;
    qemuDomainObjPrivatePtr priv = NULL;
    bool resume = false, paused = false;
    int ret = -1;
    virObjectEventPtr event = NULL;

    virCheckFlags(VIR_DUMP_LIVE | VIR_DUMP_CRASH |
                  VIR_DUMP_BYPASS_CACHE | VIR_DUMP_RESET |
                  VIR_DUMP_MEMORY_ONLY, -1);

    if (!(vm = qemuDomainObjFromDomain(dom)))
        return -1;
...
    if ((ret = doCoreDump(driver, vm, path, flags, dumpformat)) < 0)
        goto endjob;

    paused = true;
...
}
```
qemuDomainCoreDumpWithFormat函数主要是获取domain指针，然后检查ACL策略，判断是alive模式dump还是非alive模式dump，如果是非alive的话需要停止CPU操作。接着执行doCoreDump函数

```
static int
doCoreDump(virQEMUDriverPtr driver,
           virDomainObjPtr vm,
           const char *path,
           unsigned int dump_flags,
           unsigned int dumpformat)
{
    int fd = -1;
    int ret = -1;
    int rc = -1;
    virFileWrapperFdPtr wrapperFd = NULL;
    int directFlag = 0;
    unsigned int flags = VIR_FILE_WRAPPER_NON_BLOCKING;
    const char *memory_dump_format = NULL;
    g_autoptr(virQEMUDriverConfig) cfg = virQEMUDriverGetConfig(driver);
    g_autoptr(virCommand) compressor = NULL;
...

    if (dump_flags & VIR_DUMP_MEMORY_ONLY) {
        if (!(memory_dump_format = qemuDumpFormatTypeToString(dumpformat))) {
            virReportError(VIR_ERR_INVALID_ARG,
                           _("unknown dumpformat '%d'"), dumpformat);
            goto cleanup;
        }

        /* qemu dumps in "elf" without dumpformat set */
        if (STREQ(memory_dump_format, "elf"))
            memory_dump_format = NULL;

        rc = qemuDumpToFd(driver, vm, fd, QEMU_ASYNC_JOB_DUMP,
                          memory_dump_format);
    } else {
        if (dumpformat != VIR_DOMAIN_CORE_DUMP_FORMAT_RAW) {
            virReportError(VIR_ERR_OPERATION_UNSUPPORTED, "%s",
                           _("kdump-compressed format is only supported with "
                             "memory-only dump"));
            goto cleanup;
        }

        if (!qemuMigrationSrcIsAllowed(driver, vm, false, 0))
            goto cleanup;

        rc = qemuMigrationSrcToFile(driver, vm, fd, compressor,
                                    QEMU_ASYNC_JOB_DUMP);
    }
...
}
```
doCoreDump函数主要是判断flags类型，本文以VIR_DUMP_MEMORY_ONLY为例。然后打开存储dump的文件，执行qemuDumpToFd执行将内存dump至具体的文件里。

```
static int
qemuDumpToFd(virQEMUDriverPtr driver,
             virDomainObjPtr vm,
             int fd,
             qemuDomainAsyncJob asyncJob,
             const char *dumpformat)
{
    qemuDomainObjPrivatePtr priv = vm->privateData;
    bool detach = false;
    int ret = -1;

    if (!virQEMUCapsGet(priv->qemuCaps, QEMU_CAPS_DUMP_GUEST_MEMORY)) {
        virReportError(VIR_ERR_OPERATION_UNSUPPORTED, "%s",
                       _("dump-guest-memory is not supported"));
        return -1;
    }
...
    if (dumpformat) {
        ret = qemuMonitorGetDumpGuestMemoryCapability(priv->mon, dumpformat);

        if (ret <= 0) {
            virReportError(VIR_ERR_INVALID_ARG,
                           _("unsupported dumpformat '%s' "
                             "for this QEMU binary"),
                           dumpformat);
            ignore_value(qemuDomainObjExitMonitor(driver, vm));
            return -1;
        }
    }

    ret = qemuMonitorDumpToFd(priv->mon, fd, dumpformat, detach);

    if ((qemuDomainObjExitMonitor(driver, vm) < 0) || ret < 0)
        return -1;

    if (detach)
        ret = qemuDumpWaitForCompletion(vm);

    return ret;
}
```
```

int
qemuMonitorDumpToFd(qemuMonitorPtr mon,
                    int fd,  
                    const char *dumpformat,
                    bool detach)
{           
    int ret;
    VIR_DEBUG("fd=%d dumpformat=%s", fd, dumpformat);

    QEMU_CHECK_MONITOR(mon);

    if (qemuMonitorSendFileHandle(mon, "dump", fd) < 0)
        return -1;

    ret = qemuMonitorJSONDump(mon, "fd:dump", dumpformat, detach);
        
    if (ret < 0) {
        if (qemuMonitorCloseFileHandle(mon, "dump") < 0)
            VIR_WARN("failed to close dumping handle");
    }

    return ret;
}

```
这里主要是使用到了qemu的QMP机制，将dump命令发送给QMP进行将指定的domain内存dump至指定的文件中，关于QMP本文这里先不详细讨论，以后会详细介绍QMP的机制。

以上就是通过virsh dump命令实现dump虚拟机内存的详细过程，对于分析虚机问题有很大的帮助，尤其是当虚机hung死而又没有产生vmcore等文件的时候，可通过该方式进行分析。









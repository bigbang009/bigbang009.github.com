---
layout:     post
title:      一次centos8上ethtool显示问题
subtitle:   一次centos8上ethtool显示问题
date:       2021-09-05
author:     lwk
catalog: true
tags:
    - linux
    - ethtool
---

在新机房发现虚拟机的宿主机获取网卡基础信息时显示的是无效数据，如下图所示：

![image](https://user-images.githubusercontent.com/36918717/177042628-4d59796b-c3a9-4768-8735-a9c6c52ba87e.png)

新机房虚拟机的宿主机使用的镜像版本是CentOS Linux release 8.4.2105，kernel是5.10.26-2，经IDC团队排查，和内核版本无关系，但和ethtool版本有关系，centos8的默认带的是5.8版本，使用低版本的ethtool就没问题。

我下载了ethtool的源码包，分别有5.7/5.8/5.9版本，将编译后的二进制放在新机房的虚拟机测试机发现，5.8/5.9版本会出现展示无效数字问题，而低于5.8版本的就没这个问题。如下图所示：

![image](https://user-images.githubusercontent.com/36918717/177042637-214b3d61-1906-4d1c-9030-79104963256a.png)
![image](https://user-images.githubusercontent.com/36918717/177042639-2b8d9f4f-4563-41d3-a549-20c3e8089e53.png)

那么ethtool从5.7版本到5.8版本做了什么修改才会导致展示的数字无效呢？

 我们先把5.8编译出二进制，并GDB调试下
 
 ![image](https://user-images.githubusercontent.com/36918717/177042651-fb90e591-632e-43a3-92d9-cbec2a0694e9.png)

接下来我们编译5.7版本的二进制，并且GDB看下：

![image](https://user-images.githubusercontent.com/36918717/177042657-21d0c4fc-2ff7-40f5-9a51-0c8986c17a9d.png)

我们把args打印出来看到结果如下：

```
{{opts = 0x42a275 "-s|--change",no_dev = false, func = 0x408390 <do_sset>, nlfunc = 0x427750<nl_sset>, help = 0x42b60a "Change generic options",
   xhelp = 0x42d310 "\t\t[ speed %d ]\n\t\t[ duplex half|full ]\n\t\t[port tp|aui|bnc|mii|fibre|da ]\n\t\t[ mdix auto|on|off ]\n\t\t[ autoneg on|off]\n\t\t[ advertise %x[/%x] | mode on|off ... [--] ]\n\t\t[ phyad %d ]\n\t\t[xcvr internal|ext"...}, {opts = 0x42a265 "-a|--show-pause",no_dev = false, func = 0x404cc0 <do_gpause>, nlfunc = 0x0,
   help = 0x42b621 "Show pause options", xhelp = 0x0}, {opts =0x42b634 "-A|--pause", no_dev = false, func = 0x4081a0<do_spause>, nlfunc = 0x0, help = 0x42b63f "Set pause options",
   xhelp = 0x42d458 "\t\t[ autoneg on|off ]\n\t\t[ rx on|off ]\n\t\t[tx on|off ]\n"}, {opts = 0x42b651 "-c|--show-coalesce", no_dev =false, func = 0x404c30 <do_gcoalesce>,
   nlfunc = 0x0, help = 0x42b664 "Show coalesce options", xhelp =0x0}, {opts = 0x42b67a "-C|--coalesce", no_dev = false, func =0x407b70 <do_scoalesce>, nlfunc = 0x0,
   help = 0x42b688 "Set coalesce options",
   xhelp = 0x42d490 "\t\t[adaptive-rx on|off]\n\t\t[adaptive-txon|off]\n\t\t[rx-usecs N]\n\t\t[rx-frames N]\n\t\t[rx-usecs-irqN]\n\t\t[rx-frames-irq N]\n\t\t[tx-usecs N]\n\t\t[tx-framesN]\n\t\t[tx-usecs-irq N]\n\t\t[tx-frames-irq N]\n\t\t[stats-block"...},{opts = 0x42b69d "-g|--show-ring", no_dev = false, func = 0x404b60<do_gring>, nlfunc = 0x0,
   help = 0x42b6ac "Query RX/TX ring parameters", xhelp = 0x0},{opts = 0x42b6c8 "-G|--set-ring", no_dev = false, func = 0x407920<do_sring>, nlfunc = 0x0,
   help = 0x42b6d6 "Set RX/TX ring parameters", xhelp = 0x42d640"\t\t[ rx N ]\n\t\t[ rx-mini N ]\n\t\t[ rx-jumbo N ]\n\t\t[ tx N]\n"}, {
   opts = 0x42d678 "-k|--show-features|--show-offload", no_dev =false, func = 0x4057f0 <do_gfeatures>, nlfunc = 0x0,
   help = 0x42d6a0 "Get state of protocol offload and otherfeatures", xhelp = 0x0}, {opts = 0x42b6f0"-K|--features|--offload", no_dev = false, func = 0x4072f0<do_sfeatures>,
   nlfunc = 0x0, help = 0x42d6d8 "Set protocol offload and otherfeatures", xhelp = 0x42b708 "\t\tFEATURE on|off ...\n"}, {opts =0x42b71e "-i|--driver", no_dev = false,
   func = 0x404a40 <do_gdrv>, nlfunc = 0x0, help = 0x42b72a"Show driver information", xhelp = 0x0}, {opts = 0x42b742"-d|--register-dump", no_dev = false, func = 0x40b960<do_gregs>,
   nlfunc = 0x0, help = 0x42b755 "Do a register dump", xhelp =0x42d700 "\t\t[ raw on|off ]\n\t\t[ file FILENAME ]\n"}, {opts =0x42b768 "-e|--eeprom-dump", no_dev = false,
    func= 0x40b6d0 <do_geeprom>, nlfunc = 0x0, help = 0x42b779 "Do a EEPROMdump", xhelp = 0x42d728 "\t\t[ raw on|off ]\n\t\t[ offset N ]\n\t\t[length N ]\n"}, {
   opts = 0x42b78a "-E|--change-eeprom", no_dev = false, func =0x406fa0 <do_seeprom>, nlfunc = 0x0, help = 0x42b79d "Change bytesin device EEPROM",
   xhelp = 0x42d758 "\t\t[ magic N ]\n\t\t[ offset N ]\n\t\t[ length N]\n\t\t[ value N ]\n"}, {opts = 0x42b7bb "-r|--negotiate",no_dev = false, func = 0x4049e0 <do_nway_rst>,
   nlfunc = 0x0, help = 0x42b7ca "Restart N-WAY negotiation",xhelp = 0x0}, {opts = 0x42b7e4 "-p|--identify", no_dev = false, func= 0x404940 <do_phys_id>, nlfunc = 0x0,
   help = 0x42d798 "Show visible port identification (e.g.blinking)", xhelp = 0x42d7d0 ' ' <repeats 15 times>, "[TIME-IN-SECONDS ]\n"}, {opts = 0x42b7f2 "-t|--test", no_dev =false,
   func = 0x4046a0 <do_test>, nlfunc = 0x0, help = 0x42b7fc"Execute adapter self test", xhelp = 0x42d7f8 ' ' <repeats 15times>, "[ online | offline | external_lb ]\n"}, {
    opts = 0x42b816 "-S|--statistics",no_dev = false, func = 0x404680 <do_gnicstats>, nlfunc = 0x0, help =0x42b826 "Show adapter statistics", xhelp = 0x0}, {
   opts = 0x42b83e "--phy-statistics", no_dev = false, func =0x404660 <do_gphystats>, nlfunc = 0x0, help = 0x42b84f "Show phystatistics", xhelp = 0x0}, {
   opts = 0x42d830 "-n|-u|--show-nfc|--show-ntuple", no_dev =false, func = 0x405920 <do_grxclass>, nlfunc = 0x0,
   help = 0x42d850 "Show Rx network flow classification options orrules",
    xhelp = 0x42d888 "\t\t[ rx-flow-hashtcp4|udp4|ah4|esp4|sctp4|tcp6|udp6|ah6|esp6|sctp6 [context %d] |\n\t\t  rule %d ]\n"}, {opts = 0x42d8f0"-N|-U|--config-nfc|--config-ntuple",
   no_dev = false, func = 0x406060 <do_srxclass>, nlfunc = 0x0, help= 0x42d918 "Configure Rx network flow classification options orrules",
   xhelp = 0x42d958 "\t\trx-flow-hashtcp4|udp4|ah4|esp4|sctp4|tcp6|udp6|ah6|esp6|sctp6 m|v|t|s|d|f|n|r... [context%d] |\n\t\tflow-typeether|ip4|tcp4|udp4|sctp4|ah4|esp4|ip6|tcp6|udp6|ah6|esp6|sctp6\n\t\t\t[ src%x:%x:%x:%x:%x:%"...}, {opts = 0x42b863"-T|--show-time-stamping", no_dev = false, func = 0x404210<do_tsinfo>, nlfunc = 0x0,
   help = 0x42dc40 "Show time stamping capabilities", xhelp =0x0}, {opts = 0x42dc60 "-x|--show-rxfh-indir|--show-rxfh", no_dev =false, func = 0x405000 <do_grxfh>, nlfunc = 0x0,
   help = 0x42dc88 "Show Rx flow hash indirection table and/or RSShash key", xhelp = 0x42b87b "\t\t[ context %d ]\n"}, {opts =0x42b88d "-X|--set-rxfh-indir|--rxfh", no_dev = false,
   func = 0x4098b0 <do_srxfh>, nlfunc = 0x0, help = 0x42dcc0"Set Rx flow hash indirection table and/or RSS hash key",
   xhelp = 0x42dcf8 "\t\t[ context %d|new ]\n\t\t[ equal N | weight W0W1 ... | default ]\n\t\t[ hkey %x:%x:%x:%x:%x:.... ]\n\t\t[ hfunc FUNC ]\n\t\t[delete ]\n"}, {
   opts = 0x42b8a8 "-f|--flash", no_dev = false, func = 0x4040f0<do_flash>, nlfunc = 0x0, help = 0x42dd78 "Flash firmware image fromthe specified file to a region on the device",
   xhelp = 0x42ddc0 ' ' <repeats 15 times>, "FILENAME [REGION-NUMBER-TO-FLASH ]\n"}, {opts = 0x42b8b3"-P|--show-permaddr", no_dev = false, func = 0x403130<do_permaddr>,
   nlfunc = 0x429540 <nl_permaddr>, help = 0x42ddf8 "Showpermanent hardware address", xhelp = 0x0}, {opts = 0x42b8c6"-w|--get-dump", no_dev = false, func = 0x403ec0<do_getfwdump>,
   nlfunc = 0x0, help = 0x42b8d4 "Get dump flag, data", xhelp =0x42b8e8 "\t\t[ data FILENAME ]\n"}, {opts = 0x42b8fd"-W|--set-dump", no_dev = false, func = 0x4065e0<do_setfwdump>,
   nlfunc = 0x0, help = 0x42b90b "Set dump flag of the device",xhelp = 0x42b927 "\t\tN\n"}, {opts = 0x42b92c"-l|--show-channels", no_dev = false, func = 0x403df0<do_gchannels>,
   nlfunc = 0x0, help = 0x42b93f "Query Channels", xhelp = 0x0},{opts = 0x42b94e "-L|--set-channels", no_dev = false, func = 0x406d40<do_schannels>, nlfunc = 0x0,
   help = 0x42b960 "Set Channels",
   xhelp = 0x42de18 ' ' <repeats 15 times>, "[ rx N ]\n", '' <repeats 15 times>, "[ tx N ]\n", ' ' <repeats 15times>, "[ other N ]\n", ' ' <repeats 15 times>, "[combined N ]\n"}, {
   opts = 0x42b96d "--show-priv-flags", no_dev = false, func =0x403c20 <do_gprivflags>, nlfunc = 0x0, help = 0x42b97f "Queryprivate flags", xhelp = 0x0}, {
   opts = 0x42b993 "--set-priv-flags", no_dev = false, func =0x406b20 <do_sprivflags>, nlfunc = 0x0, help = 0x42b9a4 "Set privateflags", xhelp = 0x42b9b6 "\t\tFLAG on|off ...\n"}, {
--Type <RET> for more, q to quit, cto continue without paging--jRET
   opts = 0x42de88 "-m|--dump-module-eeprom|--module-info",no_dev = false, func = 0x40bd70 <do_getmodule>, nlfunc = 0x0,
   help = 0x42deb0 "Query/Decode Module EEPROM information and opticaldiagnostics if available",
   xhelp = 0x42df00 "\t\t[ raw on|off ]\n\t\t[ hex on|off ]\n\t\t[offset N ]\n\t\t[ length N ]\n"}, {opts = 0x42b9c9 "--show-eee",no_dev = false, func = 0x40b5a0 <do_geee>,
   nlfunc = 0x0, help = 0x42b9d4 "Show EEE settings", xhelp =0x0}, {opts = 0x42b9e6 "--set-eee", no_dev = false, func = 0x406900<do_seee>, nlfunc = 0x0,
   help = 0x42b9f0 "Set EEE settings", xhelp = 0x42df48"\t\t[ eee on|off ]\n\t\t[ advertise %x ]\n\t\t[ tx-lpi on|off ]\n\t\t[tx-timer %d ]\n"}, {opts = 0x42ba01 "--set-phy-tunable",
   no_dev = false, func = 0x40a580 <do_set_phy_tunable>, nlfunc =0x0, help = 0x42ba13 "Set PHY tunable",
   xhelp = 0x42df98 "\t\t[ downshift on|off [count N] ]\n\t\t[fast-link-down on|off [msecs N] ]\n\t\t[ energy-detect-power-down on|off [msecsN] ]\n"}, {
   opts = 0x42ba23 "--get-phy-tunable", no_dev = false, func =0x403970 <do_get_phy_tunable>, nlfunc = 0x0, help = 0x42ba35 "GetPHY tunable",
   xhelp = 0x42e010 "\t\t[ downshift ]\n\t\t[ fast-link-down ]\n\t\t[energy-detect-power-down ]\n"}, {opts = 0x42ba45 "--reset",no_dev = false, func = 0x4036d0 <do_reset>,
   nlfunc = 0x0, help = 0x42ba4d "Reset components",
   xhelp = 0x42e058 "\t\t[ flags %x ]\n\t\t[ mgmt ]\n\t\t[ mgmt-shared]\n\t\t[ irq ]\n\t\t[ irq-shared ]\n\t\t[ dma ]\n\t\t[ dma-shared ]\n\t\t[filter ]\n\t\t[ filter-shared ]\n\t\t[ offload ]\n\t\t[ offload-shared ]\n\t\t[mac ]\n\t\t[ mac-shared ]\n\t\t[ phy"...}, {opts = 0x42ba5e"--show-fec", no_dev = false, func = 0x4035e0 <do_gfec>, nlfunc= 0x0,
   help = 0x42ba69 "Show FEC settings", xhelp = 0x0}, {opts =0x42ba7b "--set-fec", no_dev = false, func = 0x4054c0<do_sfec>, nlfunc = 0x0, help = 0x42ba85 "Set FEC settings",
   xhelp = 0x42e188 "\t\t[ encoding auto|off|rs|baser|llrs[...]]\n"}, {opts = 0x42ba96 "-Q|--per-queue", no_dev = false,func = 0x40c170 <do_perqueue>, nlfunc = 0x0,
   help = 0x42baa5 "Apply per-queue command. ",
   xhelp = 0x42e1b8 "The supported sub commands include--show-coalesce, --coalesce", ' ' <repeats 13 times>,"[queue_mask %x] SUB_COMMAND\n"}, {opts = 0x42babf"-h|--help",
   no_dev = true, func = 0x402bf0 <show_usage>, nlfunc = 0x0, help =0x42bac9 "Show this help", xhelp = 0x0}, {opts = 0x42bad8"--version", no_dev = true, func = 0x4023e0 <do_version>,
nlfunc = 0x0,help = 0x42bae2 "Show version number", xhelp = 0x0}, {opts = 0x0,no_dev = false, func = 0x0, nlfunc = 0x0, help = 0x0, xhelp = 0x0}}
```

 从这里可以看到，args[27]的数据是{opts = 0x42b92c "-l|--show-channels", no_dev = false,func = 0x403df0 <do_gchannels>,

    nlfunc = 0x0, help = 0x42b93f "QueryChannels", xhelp = 0x0}

可以看出args[27].nlfunc=0x0。

 

但为什么args[27].nlfunc没有对应函数呢？

![image](https://user-images.githubusercontent.com/36918717/177042678-fb4c689c-c0f6-4d3e-badc-64ff359f946b.png)

因此，低于5.8版本会正常的进入到do_gchannels()函数，最终通过ioctl系统调用获取网卡配置信息
![image](https://user-images.githubusercontent.com/36918717/177042688-ea000e86-4935-4bbe-9307-8c6787d21fd4.png)


版本号是5.7以及版本号是5.8的在centos7上会走到ioctl系统调用获取网卡队列信息；版本号是5.7的在centos8上也是走到ioctl系统调用获取网卡队列信息；版本号是5.8在centos8上通过netlink获取网卡队列信息，在调用show_u32()函数转换时，由于attr是空，所以print n/a.

至于attr为什么为空，主要是由于通过netlink获取到的值为0导致。






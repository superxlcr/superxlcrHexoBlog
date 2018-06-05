---
title: Linux磁盘学习小结
date: 2016-03-01 20:37:28
tags: [linux,操作系统]
categories: [操作系统]
description: 磁盘连接方式与设备文件名关系、磁盘的组成、磁盘分区表、开机流程
---
最近在看《鸟哥的Linux私房菜》学习Linux下关于磁盘的知识，在此做个小结：  

# 磁盘连接方式与设备文件名关系
目前个人计算机常见的磁盘接口有 IDE 和 SATA（主流） 两种。对于 IDE 接口，计算机一般提供2个接口，而每个接口又可以连接2个 IDE 设备，故在Linux其设备名称为<font color=#ff0000>/dev/hd[a-d]</font>。但由于种种原因， IDE 接口现基本已弃用。现在个人计算机上主流的均为 SATA 接口，由于 SATA/USB/SCSI 等磁盘接口均使用 SCSI 模块来驱动，故在Linux下其设备名称均为<font color=#ff0000>/dev/sd[a-p]</font>，且由于 SATA 接口没有固定的顺序，设备名称<font color=#ff0000>根据Linux内核检测到磁盘的顺序决定。</font>

# 磁盘的组成
磁盘主要由盘片，机械手臂，磁头和主轴马达组成，而盘片上又有扇区（每个大小为512byte）和柱面（圆环）两种单位。其中，整块磁盘的第一个扇区特别重要，因为第一扇区记录了<font color=#ff0000>主引导分区（Master Boot Record，MBR），</font>一个可以安装加载程序的地方。除此以外，第一扇区还记录了<font color=#ff0000>分区表</font>，用来记录整块硬盘分区的状态。

# 磁盘分区表
磁盘分区的最小单位是<font color=#ff0000>柱面</font>，其实所谓的“分区”就是修改第一扇区里的分区表。硬盘默认的分区表仅能写入四组分区信息，其中：<font color=#ff0000>主分区和扩展分区最多可以有4个，扩展分区最多只能有一个，只有扩展分区才能继续切割逻辑分区</font>。（如果你想把磁盘分为5个区以上则必须有扩展分区）

# 开机流程
在计算机开机时，系统首先会去执行 BIOS （Basic Input Output System），然后 BIOS 会去分析计算机上的设备文件，读取 MBR 里的<font color=#ff0000>引导加载程序（Boot loader，用于提供菜单，载入内核文件和加载其他的boot loader）</font>，由 loader 加载内核文件启动操作系统。
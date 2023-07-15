---
layout: post
title: "备份的基础概念"
date:   2022-5-10
tags: [cloud]
comments: true
author: Reynold
notice: 如需转载请通过github联系本人，否则禁止转载
---

# VM backup definition

Virtual machine backup (VM backup) is the process of backing up the virtual machines (VMs) running in an enterprise environment.

VMs usually run as guests on hypervisors that emulate a computer system, and allow multiple VMs to share a physical host hardware system.

As enterprises increasingly rely on virtualization, VMs have become a vital part of enterprise IT environments. Business applications databases, and even containerized workloads run on VMs and generate vast amounts of data that need to be protected using a robust data protection solution. The applications that enable companies to backup and restore all the files that comprise entire virtual machines are called VM backup software.

# What is VM backup?

虚拟机备份是虚拟机的数据保护解决方案，它执行与物理服务器使用的传统备份解决方案类似的功能。

Virtual machine backup applications can perform a full backup of all files in a VM, an incremental backup, or differential backup. 

vm备份软件需要频繁和定期运行，以保护vm文件、配置和不断变化的数据。

现代的vm备份软件提供更强大的功能，允许进行更快且对虚拟机影响更小的备份。

## VM backup concepts

- Differential backup –  备份从上次全量备份以来变化的文件。
- File-level backup – Backup that is defined at the level of files and folders.
- Full backup – A backup of all files.
- Full vm backup – 构成整个虚拟机的所有文件的备份，包括磁盘映像、配置文件等。
- Image-level backup – or volume level 备份整个存储卷
- Incremental backup – Backup of only the files that have changed since the last backup, either full or incremental.
- **Quiescing** – A process to bring the data of the virtual machine to a state suitable for backups including, flushing of buffers from the operating systems memory cache to disk or other application specific tasks.

# 运行在虚拟机内部的备份客户端

虚拟机如同物理机一样，可以使用运行在系统内部的备份软件进行备份。采用这种办法，备份客户端可以在虚拟机进行备份的时候静默其系统。这种方法典型的应用是，对虚拟机内部磁盘存储数据进行文件层次的备份。

This method has advantages and disadvantages. Advantages can be there are no procedural changes or skills required since the backup agent is similar process to that of a physical server backup. This method can help with consistency of application data for some business critical workloads like databases. A disadvantage could be higher usage of the hosts resources as the backup operation is being carried out. However, computer resources today are more capable and the backup software is less resource intensive.

# 运行在宿主机的备份客户端

 因为虚拟机事实上就是由若干个配置、镜像文件等组成，所以运行在宿主机的vm管理程序可以对其进行备份。 This allows for the files that make up the virtual machine to be backed up as files, but with a few differences. Since these files are being written to while the virtual machine is running, the VM must be power off. 

If your VM backups include transactional apps such as database and email servers, quiescing them to make sure that they are in a proper state for backup is referred to as application-consistent as an example. Before a backup starts, the application is paused to ensure that any outstanding transactions and writes are written to disk. This step ensures that the server is aware of the operation and that no data will be lost if VM recovery is needed. Quiescing only works with applications that support the pausing and writing of pending data whenever necessary.（对应用的）静默，只能工作在那些支持挂起和恢复数据的应用之上（MySQL:FTWRL）.

# vm backup的应用场景

- 业务的连续性。借助 vm 备份软件，企业可以保护其所有数据和配置，以便在发生不可预测的中断后恢复业务。
- 数据保护。如果没有适当的保护，数据容易受到潜在的丢失和损坏。vm backup可保留数据的完整性并提供丢失数据的可用副本。
- 灾难恢复。众所周知，所有企业和 IT 环境都将面临可能导致数据丢失、损坏或中断 IT 运营的意外事件。使用vm backup软件可降低降低中断的风险。

# 静默备份

## 一致性备份

| 一致性                                    | 细节                                                         | 恢复                                                         |      |
| ----------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| 崩溃一致性 **Crash-consistent**           | 表示备份的数据类似系统异常宕机时的状态（应用对文件的写还没有停止，应用的业务操作未全部flush，如数据库事务） | vm启动，然后进行磁盘检测然后恢复损坏错误。应用程序实现自己的数据验证。例如，数据库应用程序可以使用其事务日志进行验证。如果事务日志中的条目不在数据库中，数据库将回滚事务，直到数据一致。 |      |
| 文件系统一致性 **File-system consistent** | 文件系统一致性备份情况下，文件系统缓存中的数据应该刷入磁盘中，文件备份处于同一时刻。 | 应用程序需要实现自己的“修复”机制，以确保恢复的数据是一致的。 |      |
| 应用一致性 **Application-consistent**     | 表示备份的数据是应用可用的数据；（应用写完成，如事务完成）   | 没有数据损坏或丢失。应用程序以一致的状态启动。               |      |
|                                           |                                                              |                                                              |      |

## 图示

![image-20220912215144495](https://raw.githubusercontent.com/QureL/qurel.github.io/main/images/kernels\一致性备份分层.png)

以运行MySQL的Linux系统为例，进行文件系统一致性备份的大致流程为：

![image-20220912221515557](https://raw.githubusercontent.com/QureL/qurel.github.io/main/images/kernels\mysql备份流程)

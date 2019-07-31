---
title: Win10+Ubuntu双系统：UEFI+GPT和Legacy+MBR引导模式
date: 2018-05-31
tags:
    - 操作系统
categories: 操作系统
---

## 磁盘分区格式介绍

一般来说，磁盘分区表有两种格式：MBR 和 GPT

### MBR 分区

在 `windows` 操作系统下最多支持4个主分区或3个主分区+1个扩展分区（包含多个逻辑分区），扩展分区必须划分为逻辑分区才能使用，1个扩展分区可以划分多个逻辑分区, `MBR` 分区表不支持容量大于 `2.2TB` 的分区(一些硬盘制造商将他们的容量较大的磁盘升级到了 `4KB` 的扇区,这意味着 `MBR` 的有效容量上限提升到了 `16 TB`)

如下图 : 是一个 MBR 分区表示例：1 个主分区+1 个扩展分区（划分了 3 个逻辑分区）

![2019-7-31-17-27.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-27.png)

<!-- more -->

### GPT 分区

GPT 分区 对分区数量没有限制，但在 windows 系统上最多可以支持 128 个主分区GPT 分区表突破了 MBR 最大支持 2.2T 分区的限制，貌似最大支持 18EB 的分区如下图是一个 GPT 分区表示例：划分了 7 个主分区

![2019-7-31-17-28.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-28.png)

### 检测磁盘分区表格式的方法

当然检测磁盘分区表格式的方法大概有两种:

- 打开Windows下磁盘管理---->右击一个磁盘属性

![2019-7-31-17-30.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-30.png)

- 第二种就是通过分区工具查看

![2019-7-31-17-31.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-31.png)

### GPT和MBR转化

当然GPT也可以转化为MBR,相反MBR也可以转化为GPT

![2019-7-31-17-32.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-32.png)


## BIOS引导方式

目前主要的系统引导方式也有两种：传统的LegacyBIOS和新型的UEFI BIOS

一般来说，有如下两种引导+磁盘分区表组合方式：LegacyBIOS+MBR和UEFI BIOS+GPT
Legacy BIOS无法识别GPT分区表格式，所以也就没有LegacyBIOS+GPT组合方式；UEFI BIOS可同时识别MBR分区和GPT分区，所以UEFI下，MBR和GPT磁盘都可用于启动操作系统。不过由于微软限制，UEFI下使用Windows安装程序安装操作系统是只能将系统安装在GPT磁盘中。

再来说说传统Legacy BIOS和新型UEFI BIOS引导方式的工作原理吧：

### Legacy BIOS原理

```java
LegacyBIOS----->MBR----->活动的主分区(一般的为C盘)→\bootmgr→\Boot\BCD→\Windows\system32\winload.exe
```

传统Legacy BIOS引导windows操作系统时，是通过一个活动的主分区下的bootmgr（启动管理器）文件导入根目录下boot文件夹里的BCD（启动设置数据）文件，然后BCD文件根据自身的配置内容加载系统启动文件winload.exe（位置：根目录\Windows\system32\winload.exe）来启动系统。

一个BCD文件可以加载多个系统启动文件从而实现引导多个系统的启动通过EasyBCD工具看以看到BCD文件的内容，如下是我的win8.1和win10两个系统的BCD内容：

![2019-7-31-17-33.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-33.png)

当然要是Windows和ubantu的话就是如下样子:反正都在活动区(一般的在c盘)

![2019-7-31-17-34.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-34.png)

需要注意的是：MBR磁盘格式下，windows系统的启动文件（bootmgr、BCD）必须存放在活动的主分区内，这样才能正常引导系统启动（MBR磁盘分区格式下，只允许有一个分区是活动的）。

如果是使用微软原版操作系统按照windows安装程序来进行系统安装，系统会自动创建一个隐藏的活动主分区（win7貌似是100M，win8貌似是350M）用来存放启动文件；

如果采用其他安装方式来安装系统，由于系统默认并不会自动创建这个活动的主分区，启动文件将会存放我们的系统盘里，所以我们在采用其它安装方式安装第一个系统时，需要确保安装系统的分区是活动的、而且是主分区，而安装第二个、第三个…系统时，就不必要求必须是主分区了，逻辑分区也可以，因为安装第二个、第三个…系统时，我们已经有了一个活动的主分区了（第一个系统所在的分区），这个活动的主分区下的BCD文件里已经包含了我们的第二个、第三个…系统的启动信息用来启动第二个、第三个…系统。(设置这些系统的时候可以用EasyBCD)

### UEFI BIOS原理

- esp引导分区中的文件结构

```java
efi\boot\bootx64.efi
efi\microsoft\boot\bcd
```

- efi启动过程

uefi bios启动时，自动查找硬盘下esp分区的bootx64.efi，然后由bootx64.efi引导
efi下的bcd文件，由bcd引导指定系统文件（一般为c:\windows\system32\winload.efi）

```java
UEFIBIOS---->EFI系统分区（FAT格式的分区)---->\efi\boot\bootx64.efi---->\efi\Microsoft\boot\BCD---->\Windows\system32\winload.efi
```

UEFI BIOS引导windows系统时，是通过一个FAT格式分区下的bootmgfw.efi文件来导入BCD文件，然后BCD文件根据自身的配置内容加载系统引导文件winload.efi(对比legacy引导发现，UEFI的引导文件winload.efi,而Legacy的引导文件为winload.exe)

需要注意的是：GPT磁盘格式下，windows系统的启动文件（bootmgfw.efi、BCD）是存放在一个FAT格式的分区里的，有些出厂预装win8系统的电脑下将该FAT分区称之为ESP分区或EFI分区

如下图，ESP和EFI分区一般都是隐藏的FAT分区，可以通过DG分区工具来创建ESP分区,预装系统的时候的MSR分区没神马用.不用管他

![2019-7-31-17-35.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-35.png)

![2019-7-31-17-36.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-36.png)

可见，UEFI+GPT模式引导windows系统时，并不需要MBR主分区来存储主引导记录，也不需要活动分区，只需要你一个存放了引导启动文件的fat格式分区就可以了，这个Fat分区当然也可以是U盘等外接USB设备了。

就目前情况而言，GPT分区表磁盘不支持32位的win7以及win7之前的系统，支持64位的XP、win7、win8、win10和32位的win8、win10。一般地，GPT磁盘多与64位windows系统组合搭配。同时Ubantu16.04支持UEFI启动.

## 引导修复教程

再来说说引导丢失、损坏导致系统无法正常进入情况下，如何通过修复引导来使系统正常启动。

引导问题故障举例：

常见的引导丢失、损坏情况说明如下：

![2019-7-31-17-37.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-37.png)

上图，Winload.exe文件有问题，可见是Legacy BIOS引导文件出错；如果此处是winload.efi，则应推测是UEFI BIOS引导文件出错。

![2019-7-31-17-38.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-38.png)

上图，NTLDR is missing，NTLDR文件丢失。推断为:XP等NT5.x架构操作系统引导丢失。

NTLDR是如win 2000、XP、win2003等NT5.x架构操作系统的启动管理器文件，与之对应的bootmgr则是如Vista、win7/8/9/2008/2012等NT6.x架构操作系统的启动管理器，如下图：

![2019-7-31-17-39.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-39.png)

上图，Bootmgr is missing，推断为:win7、win8等NT6.x架构操作系统引导丢失。

![2019-7-31-17-40.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-40.png)

### 引导修复工具

下面介绍的两种方法所涉及到NABOOTAutofix、BOOTICE、DG等工具一般PE下都会集成的，这里就不给大家放单独的下载链接了。PE制作及使用的话，请百度`电脑店` `大白菜` `老毛桃`等关键字。

### 使用NTBOOT AutoFix工具来修复引导

如果你的系统无法正常进入，那么请到PE下运行NTBOOT AutoFix进行修复；如果你是多系统，其中有一个系统可以正常进入，其它系统引导丢失，那么就可以在这个正常的系统下使用NTBOOT AutoFix进行修复，一般PE下都会集成这个软件。
选择你的系统盘符，如下图

![2019-7-31-17-41.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-41.png)

**【注意】**

使用该软件进行UEFI+GPT模式系统引导修复时，需要建立ESP/EFI分区，可通过DG等工具为ESP/EFI分区并建立盘符，打开NTBOOT引导修复工具，在里面选择ESP/EFI分区所在盘符，修复即可

### 使用BOOTICE工具来修复引导

[BOOTICE工具下载](https://pan.baidu.com/s/1i6QaDdb)

此工具不是专门用来修复引导的，其功能很是强大，这里只讲如何借助它修复引导

## Legacy+MBR修复

运行BOOTICE后，切换到“BCD编辑”，然后“新建BCD”

![2019-7-31-17-42.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-42.png)

新建BCD，文件名为:BCD

![2019-7-31-17-43.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-43.png)

然后点击“查看/修改”

![2019-7-31-17-44.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-44.png)

点击“添加”，选择“windowsvista/7/2008”（这是NT6.x架构系统，当然win8/10也适用)

![2019-7-31-17-45.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-45.png)

点击“添加”后，如下图为默认的初始BCD内容

![2019-7-31-17-46.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-46.png)


## UEFI+GPT修复

![2019-7-31-17-47.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-47.png)

接下来，我们按照上面的方法再次添加win10的BCD信息，修改好之后，保存当前系统设置，win10的BCD信息就添加好了；然后再点击“保存全局设置”，这样，win8和win10的引导信息就会保存到我们创建的这个BCD文件中了，如下图：

![2019-7-31-17-48.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-48.png)

创建好BCD文件后，我们只需在PE下将这个BCD替换到相应的目录下就可以完成引导修复了。

- 如果是UEFI+GPT模式的，随此处BCD替换文件的目录为ESP/EFI这个Fat分区：efi\Microsoft\BCD(都是隐藏分区要在pe下查看)
- 如果是Legacy+MBR模式，若磁盘有一个隐藏的活动主分区，我们需要先给这个隐藏的主分区添加盘符（PE下磁盘管理添加盘符或借助DG工具添加），然后将该BCD文件替换到这个活动主分区:\Boot\BCD(都是隐藏分区要在pe下查看)

## 命令行修复指定efi分区

如果误删EFI分区,在不用pe环境或者软件的情况下

![2019-7-31-17-49.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-49.png)


正常情况下是不能删除的，不要手贱 不要手贱 不要手贱！

### 首先要有Windows的efi启动环境

![2019-7-31-17-50.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-50.png)

### 还存在efi分区

- 做完这个直接重启选择含有efi启动的u盘
- 进到安装界面以后按Shift+F10打开一个命令行窗口
- 如果你的EFI分区还在 只是被破坏需要修复的话，执行`bcdboot c:\windows /l` 即可，c是Windows安装的盘符，不是的话自己改下盘符就可以

### 不存在efi分区

如果没有EFI分区 执行以下命令

```java
diskpart
list disk
select disk * (选择你要重建EFI分区的盘的编号，以数字代替*)
list partition (如果有大于500MB的未分配空间，跳过下两步）
select partition * (选择你要减少500MB空间的分区的编号，以数字代替*)
shrink desired = 500
create partition efi size = 500
format quick fs = fat32
exit
```

因为我硬盘上没有未分配空间，上面的命令是从已经存在的分区分出500MB以便能创造新的EFI分区
然后执行执行`bcdboot c:\windows /l`     c是Windows安装盘符，这条命令是把系统盘的引导信息复制到EFI分区

## PE环境下修复

### 用bootice自动修复

我们建议大家启动64位win8PE，用它带的bcdboot来修复。

- 指定esp分区修复

1.启动64位win8PE，并用esp分区挂载器或diskgenuis挂载esp分区
2.打开cmd命令行，输入以下命令并运行
`bcdboot c:\windows /s o: /f uefi /l zh-cn`

其中：

`c:\windows`  硬盘系统目录，根据实际情况修改
`/s o`:     指定esp分区所在磁盘，根据实际情况修改
`/f uefi`   指定启动方式为uefi
`/l zh-cn`  指定uefi启动界面语言为简体中文

注：64位win7PE不带/s参数，故win7PE不支持bios启动下修复

- 不指定esp分区修复

环境为64位win7或win8PE，只有uefi启动进入PE才可以
不用挂载esp分区，直接在cmd命令行下执行：
`bcdboot c:\windows /l zh-cn`

**注意:**不指定esp分区的情况是已经存在了esp分区
其中
`c:\windows`  硬盘系统目录，根据实际情况修改
`/l zh-cn`  指定uefi启动界面语言为简体中文
注：在8PE中，我们也可以在uefi启动进入pe后，挂载esp分区用方法（一）修复

### 用bootice手动修复

从efi引导启动过程来看，虽然它的文件很多，但主要用到的就是两文件，我们完全可以在各pe下挂载esp分区，从硬盘系统中复制bootx64.efi文件，然后用用bootice制作好bcd，就完成efi引导修复。

1. 启动任一pe,用esp分区挂载器或diskgenuis挂载esp分区
2. 查看esp分区是否可正常读写，如不正常可重新格式化为fat16或fat32分区格式。
3. 在esp分区中建立如下空文件夹结构

```
\efi\boot\   （bootx64.efi等复制）
\efi\microsoft\boot\ （bcd等建立）
```

4. 复制硬盘系统中的bootmgfw.efi（一般在c:\windows\boot\efi下）到esp分区的\efi\boot\
下，并重命名为bootx64.efi
5. 打开bootice软件，有esp分区的\efi\microsoft\boot\下新建立一bcd文件，
打开并编辑bcd文件，添加“windows vista\7\8启动项(参考上面图片)
指定磁盘为硬盘系统盘在的盘，
指定启动分区为硬盘系统分区（一般为c:）
指定启动文件为：\Windows\system32\winload.efi， 是*.efi，不是*.exe，要手工改过来
最后保存当前系统设置并退出。
这样子,精简的UEFI引导就手工修复了，实机和虚拟机测试通过。

## efi系统分区设定盘符

查看esp分区不一定在pe下,在正常的Windows下也可以

以管理员身份运行，输入：

![2019-7-31-17-51.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-51.png)

```java
diskpart
list disk
select disk 0
list part
sel part x   (x为EFI分区分区号）
set id=ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
assign letter=y   (y为分配的盘符）
 ```

![2019-7-31-17-52.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-52.png)

一步一步来！ 每一行是一步！

## 安装Ubuntu

在安装前要介绍下ubuntu的各个目录

### ubuntu目录介绍

- / 根目录，建议在根目录下面只有目录，不要直接有文件。
- swap 交换空间，相当于Windows上的虚拟内存(一般和物理运存一样就可以)。
- /boot 包含了操作系统的内核和在启动系统过程中所要用到的文件，目前可有可无
- /home 用户的home目录所在地，这个分区的大小取决于有多少用户。如果是多用户共同使用一台电脑的话，这个分区是完全有必要的，况且根用户也可以很好地控制普通用户使用计算机，如对用户或者用户组实行硬盘限量使用，限制普通用户访问哪些文件等。 以往Linux系统主要是提供服务器使用，所以/home这个目录使用量并不高。但随著Linux的桌面应用发展，不少人也拿来在日常上使用，这时/home就变成存储媒体中，最占容量的目录。假如你安装Ubuntu主要是桌面应用，那你可能需要把最大的空间留给他。 额外分割出/home有个最大的好处，当你重新安装系统时，你不需要特别去备份你的个人文件，只要在安装时，选择不要格式化这个分区，重新挂载为/home就不会丢失你的数据。 还有一个特别的应用：假如你会在你的计算机上，安装两个或更多的Linux系统，你可以共享/home这个分区。简单地说，你的个人文件可以在切换到其它Linux系统时，仍正常使用

- /tmp 用来存放临时文件。这对于多用户系统或者网络服务器来说是有必要的。这样即使程序运行时生成大量的临时文件，或者用户对系统进行了错误的操作，文件系统的其它部分仍然是安全的。因为文件系统的这一部分仍然还承受着读写操作，所以它通常会比其它的部分更快地发生问题。这个目录是任何人都能访问的，所以需要定期清理。

- /usr Linux系统存放软件的地方，如有可能应将最大空间分给它
除了系统的基本程序外，其它所有的应用程序多放在这个目录当中。除了/home,/var这种变动数据的存放目录外，/usr大概是会是使用容量最大的目录，不过一般Linux下的应用程序通常不大，所以大多数的桌面应用顶多 **3~4GB** 的空间就已经相当足够了，若是服务器，多半也是 **2~3GB** 就足够了。

- /bin
/usr/bin
/usr/local/bin 存放标准系统实用程序。

- /srv 一些服务启动之后，这些服务所需要访问的数据目录，如WWW服务器需要的网页数据就可以放在/srv/www中。

- /etc 系统主要的设置文件几乎都放在这个目录内。

- /lib
/usr/lib
/usr/local/lib 系统使用的函数库的目录。

- /root 系统管理员的家目录。

- /lost+found 该目录在大多数情况下都是空的，但当实然停电或者非正常关机后，有些文件临时存入在此。

- /dev 设备文件，在Linux系统上，任何设备都以文件类型存放在这个目录中，如硬盘设备文件，软驱、光驱设备文件等。

- /mnt
- /media 挂载目录，用来临时挂载别的文件系统或者别的硬件设备（如光驱、软驱）。

- /opt 用于存储第三方软件的目录，不过我们还是习惯放在/usr/local下

- /proc 此目录信息是在内存中由系统自行产生的，存储了一些当前的进程ID号和CPU、内存的映射等，因为这个目录下的数据都在内存中，所以本身不占任何硬盘空间。

- /sbin
/usr/sbin
/usr/local/sbin 存放一些系统管理员才会用到的执行命令。

- /var 主要放置系统执行过程中经常变化的文件，例如缓存（cache）或者是随时更改的登录文件（log file）。
假如你的计算机主要是提供网页服务，或者是mysql数据库，那/var会大量增加，你最好能够把/var额外分割出来。与/home的概念类似，重新安装时，不要格式化，仍可保留原来的数据。
在服务器的应用时，数据的安全是相当重要的，额外分区对数据的安全也有所帮助。此外，/var/log是系统log档保存的位置，养成有问题就去找log的好习惯，有助于解决问题。所以这也加强了额外分区的重要性。当一个服务器出现系统问题，甚至毁损时，除了你的数据外，之前的系统纪录也相当重要，找出为什么系统会出问题，可以帮助管理器快速排除障碍。

- /var/log 系统日志记录分区，如果设立了这一单独的分区，这样即使系统的日志文件出现了问题，它们也不会影响到操作系统的主分区。

## 本人方案

由于采用了GPT模式的硬盘分区,所以接下来的分区都是主分区
分为3个区

1. 挂载点/: 安装系统和软件；大小为60G；分区格式为ext4； 这个是用的固态硬盘某个分区,用固态是为了快
2. 挂载点/home： 相当于“我的文档”；大小100G; 分区格式ext4； 这个是用的机械硬盘某个分区
3. swap： 充当虚拟内存；大小等于物理内存大小；分区格式为swap ,用的机械硬盘
4. /boot:不用分配

## 多系统共用esp

如果电脑之前就有Windows10或者通过自己按照上面的方法安装了Windows10，会存在一个几百兆的esp分区，当安装多个系统的时候可以将引导都安装到这个分区上，接下来利用ubuntu官网介绍制作u盘efi启动（自行谷歌），最后安装ubuntu引导到esp分区上就可以了

## 修复ubuntu启动

如果ubuntu的引导和Windows的引导在一个esp分区上那就需要修复下引导了

- 启动ubuntu安装盘 用efi方式启动然后选择试用ubuntu 在命令行输入

```java
sudo add-apt-repository ppa:yannubuntu/boot-repair
sudo apt-get update
sudo apt-get install -y boot-repair
boot-repair
```

修复后重启就可以了

![2019-7-31-17-53.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-53.png)

[EasyUEFI的软件以及我自己的备份](https://pan.baidu.com/s/1bqglHkN)


## 自己电脑安装：

### 宏碁E1-471g(我在家里开发用)

-  这款电脑是老式电脑，采用的启动是Legacy+MBR引导模式
- 我的光驱位置换成了500G机械硬盘，主硬盘位置放上了固态，**注意：固态最后放才能被识别**
- 安装完了Ubuntu后直接进入的是Windows，采用easybcd设置启动项

### 联想Y700

- 这款电脑是采用的UEFI+GPT启动模式
- 先在pe下的分区助手将硬盘转化为GPT格式
- 磁盘转化完后，在分出一个500M的efi系统引导分区（FAT格式的），安装Ubuntu的时候将引导放到efi分区上，这个分区存放这以后Windows和Ubuntu的引导文件

## 针对联想Y700电脑还有一个毛病

就是无缘无故的键盘失灵,在这我升级了下BIOS,我测试过了(用了半个月)倒是没有出现键盘失灵的情况,其实网上也有好多说直接弄个外接键盘,我只想说呵呵,那是在逃避问题并没有解决问题,好了我就不废话了直接上图:
[Y700-15ISK_bios cdcn53ww](http://pan.baidu.com/s/1o81BqNS)
查看BIOS版本:`win+R`   输入`msinfo32`出厂的BIOS版本:

![2019-7-31-17-54.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-54.png)

升级后:

![2019-7-31-17-55.png](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/Win10%2BUbuntu%E5%8F%8C%E7%B3%BB%E7%BB%9F%EF%BC%9AUEFI%2BGPT%E5%92%8CLegacy%2BMBR%E5%BC%95%E5%AF%BC%E6%A8%A1%E5%BC%8F/2019-7-31-17-55.png)



## 参考
- https://www.cnblogs.com/exmyth/p/4069117.html
- https://zhidao.baidu.com/question/580753087.html
- https://www.chiphell.com/thread-1522885-1-1.html
- http://bbs.pcbeta.com/forum.php?mod=viewthread&ordertype=1&tid=1727468
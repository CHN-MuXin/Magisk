# 开发者指南

## 语言
[English (英文)](guides.md)

## BusyBox
Magisk 附带一个完整功能的 BusyBox 二进制文件 (包含完整的 SELinux 支持)。 可执行文件位于 `/data/adb/magisk/busybox`。 Magisk 的 BusyBox 支持运行时可切换到 "ASH 独立 Shell 模式"。 这种独立模式的意思是，当在BusyBox的`ash`shell中运行时。每一条命令都会直接使用BusyBox中的小程序。不管设置的是什么 "PATH"。 例如，像`ls`、`rm`、`chmod`这样的命令将**不使用**`PATH`中的内容（在Android中默认为`/system/bin/ls`、`/system/bin/rm`和`/system/bin/chmod`），而是直接调用BusyBox内部小程序。这确保了脚本总是在一个可预测的环境中运行，并且无论在哪个Android版本上运行，都有全部的命令。要强迫一个命令**不使用**BusyBox，你必须用完整的路径调用可执行文件。

在Magisk的上下文中运行的每一个shell脚本都会在BusyBox的`ash`shell中执行，并启用独立模式。对于与第三方开发者有关的内容，这包括所有启动脚本和模块安装脚本。

对于那些想在Magisk之外使用这个 "独立模式 "功能的人来说，有两种方法可以启用它：

1. 设置环境变量 `ASH_STANDALONE` 为 `1`<br>示例: `ASH_STANDALONE=1 /data/adb/magisk/busybox sh <script>`
2. 使用命令行选项切换:<br>`/data/adb/magisk/busybox sh -o standalone <script>`

为了确保随后执行的所有`sh` shell也能在独立模式下运行，选项1是首选方法（这也是Magisk和Magisk应用程序内部使用的方法），因为环境变量会继承到子进程。


## Magisk 模块
Magisk模块是一个放在`/data/adb/modules`中的文件夹，结构如下：

```
/data/adb/modules
├── .
├── .
|
├── $MODID                  <--- 该文件夹以模块的ID命名
│   │
│   │      *** 模块标识 ***
│   │
│   ├── module.prop         <--- 这个文件存储了模块的元数据
│   │
│   │      *** 主要内容 ***
│   │
│   ├── system              <--- 如果skip_mount不存在，这个文件夹将被挂载
│   │   ├── ...
│   │   ├── ...
│   │   └── ...
│   │
│   │      *** 状态标识 ***
│   │
│   ├── skip_mount          <--- 如果存在，Magisk将不会挂载你的 system 文件夹
│   ├── disable             <--- 如果存在该模块将被禁用
│   ├── remove              <--- 如果存在，该模块将在下次重启时被移除
│   │
│   │      *** 可选文件 ***
│   │
│   ├── post-fs-data.sh     <--- 这个脚本在 post-fs-data 中执行
│   ├── service.sh          <--- 这个脚本在 late_start 服务中执行
|   ├── uninstall.sh        <--- 这个脚本会在 Magisk 删除该模块的时候执行
│   ├── system.prop         <--- 这个文件中的属性会被 resetprop 加载为系统属性 
│   ├── sepolicy.rule       <--- 额外的自定义 sepolicy 规则
│   │
│   │      *** 自动生成，请勿手动创建或修改。 ***
│   │
│   ├── vendor              <--- 符号链接至 $MODID/system/vendor
│   ├── product             <--- 符号链接至  $MODID/system/product
│   ├── system_ext          <--- 符号链接至  $MODID/system/system_ext
│   │
│   │      *** 允许任何其他文件/文件夹  ***
│   │
│   ├── ...
│   └── ...
|
├── another_module
│   ├── .
│   └── .
├── .
├── .
```


#### module.prop


这是 `module.prop` 的**严格**格式。

```
id=<string>
name=<string>
version=<string>
versionCode=<int>
author=<string>
description=<string>
```
- `id`必须符合这个正则表达式：`^[a-zA-Z][a-zA-Z0-9._-]+$`<br>
例如: ✓ `a_module`, ✓ `a.module`, ✓ `module-101`, ✗ `a module`, ✗ `1_module`, ✗ `a-module`<br>
这是你的模块的**唯一标识符**。一旦发布，你不应该改变它。
- `versionCode` 必须是一个 **整数**。用来比较版本
- 其它未提到的可以是任何 **单独行** 字符串.
- 确保使用 `UNIX (LF)` 换行符，而不是 `Windows (CR+LF)` 或 `Macintosh (CR)`。

#### Shell 脚本 (`*.sh`)
请阅读 [启动脚本](#启动脚本) 部分已了解 `post-fs-data.sh` 和 `service.sh` 之间的区别, 对于大多数模块开发者来说，如果你只需要运行一个启动脚本 `service.sh` 应该是最好的。

在你的模块的所有脚本中，请使用 `MODDIR=${0%/*}` 来获得你的模块的基本目录路径；**不要在**脚本中硬编码你的模块路径。

#### system.prop
这个文件的格式与 `build.prop` 相同。每一行由 `[key]=[value]` 组成。

#### sepolicy.rule
如果你的模块需要一些额外的sepolicy补丁，请把这些规则加入这个文件。模块安装脚本和Magisk的守护程序将确保这个文件被复制到 `magiskinit` 可以在启动前读取的地方，以确保这些规则被正确注入。

这个文件中的每一行都将被视为一个政策声明。关于政策声明的格式的更多细节，请查看[magiskpolicy](tools.md#magiskpolicy)的文档。


#### `system` 文件夹

你希望Magisk为你 替换/注入 的所有文件都应该放在这个文件夹里。请阅读[Magic Mount](details.md#magic-mount)部分，了解Magisk如何挂载你的文件。

## Magisk 模块安装程序

Magisk模块安装程序是一个打包在zip文件中的Magisk模块，可以在Magisk应用程序或自定义恢复程序（如TWRP）中刷入。安装包的文件结构与Magisk模块相同（更多信息请看前面的章节）。最简单的Magisk模块安装程序只是一个打包在zip文件中的Magisk模块，此外还有以下文件：

- `update-binary`: 下载最新的[module_installer.sh](https://github.com/topjohnwu/Magisk/blob/master/scripts/module_installer.sh)，并重命名/复制该脚本为`update-binary`。
- `updater-script`：这个文件应该只包含字符串 "#MAGISK"。

默认情况下，`update-binary`将检查/设置环境，加载实用功能，提取模块安装程序压缩包到你的模块将被安装的地方，最后做一些琐碎的任务和清理，这应该涵盖大多数简单模块的需求。

```
module.zip
│
├── META-INF
│   └── com
│       └── google
│           └── android
│               ├── update-binary      <--- 你下载的module_installer.sh
│               └── updater-script     <--- 只应包含字符串 "#MAGISK"
│
├── customize.sh                       <--- (可选的，更多的细节在后面)
│                                           这个脚本将由update-binary提供资源。
├── ...
├── ...  /* 模块的其他文件 */
│
```

#### 定制服务

如果你需要自定义模块的安装过程，你可以选择在安装程序中创建一个新的脚本，叫做`customize.sh`。这个脚本将在所有文件被提取并应用默认权限和 *secontext* 后被`update-binary`（而不是执行！）。如果你的模块包括基于ABI的不同文件，或者你需要为某些文件设置特殊的权限/上下文（例如，`/system/bin`中的文件），这将非常有用。

如果你需要更多的定制，并且喜欢自己做所有的事情，在`customize.sh`中声明`SKIPUNZIP=1`来跳过提取和应用默认权限/上下文的步骤。请注意，这样做，你的`customize.sh`将负责处理自己安装的一切。

#### `customize.sh` 环境

这个脚本将在Magisk的BusyBox `ash` shell中运行，并启用 "独立模式"。为了方便起见，下面的变量和shell函数是可用的：

##### Variables
- `MAGISK_VER` (string): 当前安装的Magisk的版本字符串 (例如： `v20.0`)
- `MAGISK_VER_CODE` (int): 当前安装的Magisk的版本代码 (例如： `20000`)
- `BOOTMODE` (bool): 如果该模块被安装在Magisk应用程序中，则为 "true"。
- `MODPATH` (path): 你的模块文件应该安装的路径
- `TMPDIR` (path): 一个你可以临时存储文件的地方
- `ZIPFILE` (path): 你的模块的安装压缩包
- `ARCH` (string): 设备的CPU架构。值是`arm`，`arm64`，`x86`，或`x64`。
- `IS64BIT` (bool): 如果`$ARCH`是`arm64`或`x64`，则为`true`。
- `API` (int): 设备的API级别（安卓版本） (例如： `21` 代表 Android 5.0)

##### 方法

```
ui_print <msg>
    打印<msg>到控制台
    避免使用'echo'，因为它不会显示在自定义恢复的控制台中。

abort <msg>
    打印错误信息<msg>到控制台并终止安装
    避免使用`exit`，因为它将跳过终止清理的步骤

set_perm <目标> <所有者> <组> <权限> [上下文]
    如果没有设置[上下文]，默认为 "u:object_r:system_file:s0"
    这个函数是以下命令的速记：
       chown owner.group target
       chmod permission target
       chcon context target

set_perm_recursive <目录> <所有者> <组> <目录权限> <文件权限> [上下文]
    如果没有设置[上下文]，默认为 "u:object_r:system_file:s0"
    对于<目录>中的所有文件，它将调用。
       set_perm file owner group filepermission context
    对于<目录>中的所有目录（包括它自己），它将调用。
       set_perm dir owner group dirpermission context
```

##### 简单替换

您可以在变量名“REPLACE”中声明要直接替换的文件夹列表。 模块安装程序脚本将选取此变量并为您创建`.replace` 文件。 一个示例声明： 

```
REPLACE="
/system/app/YouTube
/system/app/Bloatware
"
```

上面的列表将会创建以下文件：`$MODPATH/system/app/YouTube/.replace` 和 `$MODPATH/system/app/Bloatware/.replace` 

#### 笔记

- 当你的模块与Magisk应用程序一起下载时，`update-binary`将被**强制**替换为最新的[`module_installer.sh`](https://github.com/topjohnwu/Magisk/blob/master/scripts/module_installer.sh)，以确保所有安装程序使用最新的脚本。**不要***尝试在`update-binary`中添加任何自定义逻辑，因为这毫无意义。
- 由于历史原因，**不要**在你的模块安装程序中添加一个名为`install.sh`的文件。这个特定的文件以前被使用过，将被区别对待。
- **不要在** `customize.sh` 文件末尾调用 `exit` 。 模块安装程序要执行最终结果。

## 提交模块

你可以提交一个模块到**Magisk-Module-Repo**，这样用户就可以在Magisk应用中直接下载你的模块。

- 按照上一节的说明，为你的模块创建一个有效的安装程序。
- 创建`README.md`（文件名应该完全一样），包含你的模块的所有信息。如果你不熟悉Markdown的语法，[Markdown Cheat Sheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)将很方便。
- 用你的个人GitHub账户创建一个仓库，并将你的模块安装程序上传到仓库里
- 通过这个链接创建一个提交请求。[提交](https://github.com/Magisk-Modules-Repo/submission)

## 模块技巧

#### 删除文件
如何以无系统方式删除文件？ 实际上使文件**消失** 很复杂（可能，不值得付出努力）。 用一个虚拟文件替换它应该就足够了！ 创建一个同名的空文件并将其放置在模块内的相同路径中，它将用一个虚拟文件替换您的目标文件。 

#### 删除文件夹
与上面提到的一样，实际上使文件夹**消失** 不值得付出努力。 用空文件夹替换它应该足够了！ 对于模块开发人员来说，一个方便的技巧是将要删除的文件夹添加到“customize.sh”中的“REPLACE”列表中。 如果你的模块没有提供对应的文件夹，它会创建一个空文件夹，并自动将`.replace`添加到空文件夹中，这样虚拟文件夹就会正确替换`/system`中的文件夹。 

## 启动脚本

在 Magisk 中，你可以使用两种不同的模式运行启动脚本： **post-fs-data** 和 **late_start service** 。

- post-fs-data 模式
    - 这个阶段是阻断的。在执行完成之前，或者在10秒过后，启动过程会暂停。
    - 脚本在任何模块被安装之前运行。这使得模块开发者可以在挂载之前动态地调整他们的模块。
    - 此阶段发生在 Zygote 启动之前 , which pretty much means everything in Android
    - **仅在必要时以这种模式运行脚本！** 
- late_start service 模式
    - 这个阶段是非阻塞的。 您的脚本与启动过程并行运行。 
    - **这是运行大多数脚本的推荐选择！** 

在 Magisk 中，也有两种类型的脚本： **普通脚本** 和 **模块脚本**。

- 普通脚本
    - 放在 `/data/adb/post-fs-data.d` 或 `/data/adb/service.d`
    - 仅当脚本可执行时才执行 (添加执行权限, `chmod +x script.sh`)
    - `post-fs-data.d` 中的脚本以 post-fs-data 模式运行，`service.d` 中的脚本以 late_start server 模式运行。 
    - 模块**不应该**添加普通脚本，因为这违反了封装。
- 模块脚本
    - 放置在模块的文件夹中
    - 只有模块被启用的时候才会执行
    - `post-fs-data.sh` 在 post-fs-data 模式下运行，`service.sh` 在 late_start server 模式下运行。
    - 模块要求启动脚本应该**只使用**模块脚本而不是普通脚本。

这些脚本将在 Magisk的 BusyBox `ash` shell 中运行，并启用 "独立模式"。

## 根目录叠加系统

由于`/`在系统为根的设备上是只读的，Magisk提供了一个覆盖系统，使开发者能够替换rootdir中的文件或添加新的`*.rc`脚本。这个功能主要是为定制内核的开发者设计的。

覆盖文件应放在启动镜像ramdisk的`overlay.d`文件夹中，并遵循这些规则：

1. 所有 "overlay.d "中的 "*.rc "文件将在 "init.rc "之后被读取并串联起来。
2. 现有的文件可以被位于同一相对路径的文件取代
3. 对应于不存在的文件的文件将被忽略

为了在你的自定义 `*.rc` 脚本中有你想要引用的额外文件，将它们添加到 `overlay.d/sbin` 中。 上述 3 条规则不适用于此特定文件夹中的所有内容，因为它们将直接复制到 Magisk 的内部 `tmpfs` 目录（以前总是位于 `/sbin`）。

由于 Android 11 中的更改，不再保证`/sbin` 文件夹存在。 在这种情况下，Magisk 会随机生成 `tmpfs` 文件夹。 当`magiskinit` 将它注入`init.rc` 时，`*.rc` 脚本中每次出现的模式`${MAGISKTMP}` 都会被Magisk `tmpfs` 文件夹替换。 这也适用于 Android 11 之前的设备，因为在这种情况下，`${MAGISKTMP}` 将简单地替换为 `/sbin`，因此最佳做法是 **NEVER** 在你的 `*.rc 中硬编码 `/sbin` ` 引用其他文件时的脚本。 

以下是如何使用自定义 `*.rc` 脚本设置 `overlay.d` 的示例： 

```
ramdisk
│
├── overlay.d
│   ├── sbin
│   │   ├── libfoo.ko      <--- 这两个文件将被复制 
│   │   └── myscript.sh    <--- 到 Magisk 的 tmpfs 目录 
│   ├── custom.rc          <--- 这个文件将被注入到 init.rc 
│   ├── res
│   │   └── random.png     <--- 此文件将替换 /res/random.png 
│   └── new_file           <--- 该文件将被忽略，因为 
│                               /new_file 不存在
├── res
│   └── random.png         <--- 该文件将被替换为 
│                               /overlay.d/res/random.png
├── ...
├── ...  /* initramfs 文件的其余部分  */
│
```

下面是一个 `custom.rc` 的例子： 

```
# 使用 ${MAGISKTMP} 来引用 Magisk 的 tmpfs 目录 

on early-init
    setprop sys.example.foo bar
    insmod ${MAGISKTMP}/libfoo.ko
    start myservice

service myservice ${MAGISKTMP}/myscript.sh
    oneshot
```

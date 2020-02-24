---
title: User Mode Linux Quicknote
date: 2020-02-24 00:00:00
tags: [linux, kernel]
---


**User-Mode Linux**[^1] 簡稱 UML，類似 container 等級的輕量虛擬化，但不需要和 host 共用 kernel。
UML 在 host 上已單一執行檔的形式存在，提供 kernel 測試開發一個更好的隔離環境 (sandbox)。

UML 並不是一個熱門的項目，且限制不少，例如速度不快且不支援 [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) [^2]，
因此整體應用範圍較為侷限，比較完整的應用還是需要使用 [KVM](https://www.linux-kvm.org/) + [QEMU](https://www.qemu.org/)。


## Compile

測試可以直接套用該平台精簡的預設選項 `make ARCH=um defconfig`。若不是無中生有則需要設定額外架構專屬的選項，不知道怎麼選可以參考 Kconfig[^3][^4] 的說明。
後續的例子需要用到的 hostfs 務必開啟。

```
Host filesystem (HOSTFS) [N/m/y/?] (NEW) y
```

直接執行 `make ARCH=um -j$(nproc)` 進行編譯，完成之後會產生一隻 `linux` 執行檔，但直接執行還不會動，因為還缺少 rootfs。

```console
[    0.240000] VFS: Cannot open root device "98:0" or unknown-block(98,0): error -6
[    0.240000] Please append a correct "root=" boot option; here are the available partitions:
[    0.240000] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(98,0)
[    0.240000] CPU: 0 PID: 1 Comm: swapper Not tainted 5.4.13 #15
[    0.240000] Stack:
[    0.240000]  6349bd50 607a8e00 600bad00 600badba
[    0.240000]  00008001 62d5a000 6349bd60 607a8e5c
[    0.240000]  6349be80 6006eb49 6098560b 600badba
[    0.240000] Call Trace:
```


## Rootfs

準備 rootfs 的方法很多，常見的工具有 debian 的 [debootstrap](https://wiki.debian.org/Debootstrap), [Multistrap](https://wiki.debian.org/Multistrap)，或 [buildroot](https://buildroot.org/), [OpenEmbedded](http://www.openembedded.org/) 等。

但是 rootfs 以 image 形式還是有點麻煩，而 UML 支援透過 hostfs 存取 host 底下特定目錄，因此也不意外可以透過 hostfs 指定目錄作為 rootfs。
手邊如果有現成的 [chroot](https://en.wikipedia.org/wiki/Chroot) 環境，就可以直接拿來餵給 UML 用。

此外，也可以拿 [Gentoo Stage 3](https://www.gentoo.org/downloads/) 或 [Alpine Mini Root](https://alpinelinux.org/downloads/) 解開來用即可。
好處是自帶套件管理系統，之後打通網路後新增應用程式會方便很多。

以下以 gentoo 為例，直接拉 [stage3 tarball](https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/current-stage3-amd64/) 解開到 `$HOME/hostfs` 來用。

```console
$ mkdir $HOME/hostfs
$ wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-<release>.tar.xz
# tar xpvf stage3-amd64-<release>.tar.xz --xattrs-include='*.*' --numeric-owner -C $HOME/hostfs
```


## Launch with Hostfs

執行下列指令就可以啟動 UML 帶起位於 `$HOME/hostfs` 的 rootfs。UML 可以使用額外的參數，詳情可參可 `linux --help`。

```console
# ./linux root=/dev/root rootfstype=hostfs rootflags=$HOME/hostfs/ rw mem=128MB init=/bin/bash 
```

記得把 procfs 和 sysfs 這兩個和 kernel 溝通重要的界面掛起來。如果有開啟 `CONFIG_DEVTMPFS_MOUNT`，啟動的過程中會自動掛載 devtmpfs 到 /dev，也要先卸載掉。

```console
UML# mount -t proc proc proc/
UML# mount -t sysfs sys sys/
```


## Install Kernel Modules

```console
$ make ARCH=um modules
$ make ARCH=um _modinst_ MODLIB=$HOME/lib/modules/$KERNELRELEASE
```


## Debugging with GDB

(... 這次沒機會用到 ...)


## References
- Jserv: 透過 User-Mode-Linux 來學習核心設計 [(1)](http://blog.linux.org.tw/~jserv/archives/001871.html) [(2)](http://blog.linux.org.tw/~jserv/archives/001872.html). 2007.
- Jserv: [建構 User-Mode Linux 的實驗環境](https://hackmd.io/@sysprog/user-mode-linux-env). 2020.
- [Wikipedia: User-mode Linux](https://en.wikipedia.org/wiki/User-mode_Linux)
- [Gentoo: user-mode linux guide](https://wiki.gentoo.org/wiki/User-mode_Linux/Guide)
- [Arch Linux: user-mode linux](https://wiki.archlinux.org/index.php/User-mode_Linux)
- [Christine Dodrill: How to Use User Mode Linux](https://christine.website/blog/howto-usermode-linux-2019-07-07). 2019.
- [Rob's quick and dirty UML howto](https://www.landley.net/code/UML.html)
- [Advanced testing with UserModeLinux](https://static.sched.com/hosted_files/osseu19/ca/slides.pdf). 2019.

[^1]: [The User-mode Linux Kernel Home Page](http://user-mode-linux.sourceforge.net/)
[^2]: [UML Status Report](http://events17.linuxfoundation.org/sites/events/files/slides/slides_5.pdf). 2017.
[^3]: [linux/arch/um/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/um/Kconfig)
[^4]: [linux/arch/um/drivers/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/um/drivers/Kconfig)


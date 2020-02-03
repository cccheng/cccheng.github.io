---
title: User Mode Linux Quicknote
date: 2020-01-01 00:00:00
tags: [linux, kernel]
---

**User-Mode Linux**[^1] 簡稱 UML，類似 container 等級的輕量虛擬化，但不需要和 host 共用 kernel。UML 在 host[^4] 上已單一執行檔的形式存在，提供 kernel 開發一個更好的隔離環境 (sandbox)。

## Compile

將編譯的架構改成 um 即可：`make ARCH=um -j$(nproc)`，會有額外架構專屬的選項，不知道怎麼選可以參考 Kconfig[^2][^3] 的說明。
以下情境需要用到 hostfs 務必開啟。

```console
Host filesystem (HOSTFS) [N/m/y/?] (NEW) y
```

編譯完畢之後會產生一隻 `linux` 執行檔，但直接執行還不會動，因為還缺少 rootfs。

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

準備 rootfs image 的方法很多，debian 的 [debootstrap](https://wiki.debian.org/Debootstrap)

另外也可以直接透過 hostfs 把特定目錄作為 rootfs，省去製作 image，類似準備 [chroot](https://en.wikipedia.org/wiki/Chroot) 或 [FreeBSD jail](https://en.wikipedia.org/wiki/FreeBSD_jail)[^4] 的環境。
這時候可以拿 [Gentoo Stage 3](https://www.gentoo.org/downloads/) 或 [Alpine Mini Root](https://alpinelinux.org/downloads/) 解開來用即可。以下以 gentoo 為例。

```console
$ mkdir $HOME/gentoo
$ wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-<release>.tar.xz
# tar xpvf stage3-amd64-<release>.tar.xz --xattrs-include='*.*' --numeric-owner -C $HOME/gentoo
```

## Launch

user-mode linux 可以使用額外的參數，詳情可參可 `linux --help`。

```console
# ./linux root=/dev/root rootfstype=hostfs rootflags=$HOME/Downloads/gentoo/ rw mem=128MB init=/bin/bash 
```

```
# mount -t proc proc proc/
# mount -t sysfs sys sys/
```

啟動的過程中會自動掛載 devtmpfs 到 /dev，先退掉。

```
# umount /dev
```

## References
- Jserv: 透過 User-Mode-Linux 來學習核心設計 [(1)](http://blog.linux.org.tw/~jserv/archives/001871.html) [(2)](http://blog.linux.org.tw/~jserv/archives/001872.html). 2007.
- Jserv: [建構 User-Mode Linux 的實驗環境](https://hackmd.io/@sysprog/user-mode-linux-env)
- [Wikipedia: User-mode Linux](https://en.wikipedia.org/wiki/User-mode_Linux)
- [Gentoo: user-mode linux guide](https://wiki.gentoo.org/wiki/User-mode_Linux/Guide)
- [Arch Linux: user-mode linux](https://wiki.archlinux.org/index.php/User-mode_Linux)
- [Christine Dodrill: How to Use User Mode Linux](https://christine.website/blog/howto-usermode-linux-2019-07-07). 2019.
- [Rob's quick and dirty UML howto](https://www.landley.net/code/UML.html)
- [UML Status Report](http://events17.linuxfoundation.org/sites/events/files/slides/slides_5.pdf). 2017.

[^1]: [The User-mode Linux Kernel Home Page](http://user-mode-linux.sourceforge.net/)
[^2]: [linux/arch/um/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/um/Kconfig)
[^3]: [linux/arch/um/drivers/Kconfig](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/um/drivers/Kconfig)
[^4]: [Creating and Controlling Jails in FreeBSD](https://www.freebsd.org/doc/handbook/jails-build.html)


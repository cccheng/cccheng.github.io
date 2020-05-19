---
title: Remove Snap in Xubuntu 20.04
date: 2020-05-04 00:00:00
tags: [linux]
---


最近把手邊的機器換上 Xubuntu 20.04，但開機過程中馬上遭遇兩個錯誤。


## Failed to activate swapfile

第一個錯誤是 systemd 抱怨無法啟用 swap。看起來安裝時如果沒有另外切一個獨立的 swap 分割區，系統會自動在根目錄添加一個 1.5GB 的檔案作為 swap file。
但 swap file 如果是放在 Btrfs 上有些限制需要注意，否則就會啟用失敗。

```console
systemd[1]: Failed to activate swap /swapfile.
```

首先是 kernel 版本，Btrfs 在 Linux-5.0 之前為了躲問題，故意不實作支援 swapfile 所需的函式[^1][^2]，後來由 Facebook 的 Omar Sandoval 解決了[^3] 。
其次是 swap file 不能開啟 [Copy-on-Write](https://en.wikipedia.org/wiki/Copy-on-write) 或 [Compression](https://btrfs.wiki.kernel.org/index.php/Compression)，否則也會失敗。

Xubuntu 20.04 已經採用 Linux-5.4，因此不是版本問題。問題在於 Xubuntu 自動建立的 swapfile 並沒有關閉 CoW，重新建立一個即可。

```console
# rm /swapfile
# touch /swapfile
# chattr +C /swapfile
# fallocate -l 1599283200 /swapfile
# mkswap /swapfile
# swapon /swapfile
```

---


## Failed to mount Mount unit for &lt;snap-app&gt;

第二個問題則是幾個神秘的 Mount unit 掛載失敗，原因是以為沒用到就把 [Squashfs](https://en.wikipedia.org/wiki/SquashFS) 關了，原來系統背後的 snap 有偷偷在使用。

```console
Failed to mount Mount unit for core18
Failed to mount Mount unit for gtk-common-themes
```

[Snap](https://en.wikipedia.org/wiki/Snap_(package_manager)) 是 Canonical 主推的軟體包裝方式，可以解決相依行問題並且自帶 [sandboxing](https://lwn.net/Articles/694757/)，缺點就是肥了一點。
類似機制的競爭對手還有 [Flatpak](https://flatpak.org/) 和 [AppImage](https://appimage.org/) 。
Ubuntu 自 16.04 已開始內建 snap，到了 19.10 也悄悄地把 [Chromium](https://www.chromium.org/) 轉換成 snap 形式，所以這次升級直接放棄安裝。但系統內仍然以下 snap packages。

```console
$ Snap list
Name               Version          Rev   Tracking       Publisher   Notes
core18             20200311         1705  latest/stable  canonical✓  base
gtk-common-themes  0.1-36-gc75f853  1506  latest/stable  canonical✓  -
snapd              2.44.3           7264  latest/stable  canonical✓  snapd
```

直接移除也不影響系統運作：

```console
$ sudo snap remove gtk-common-themes
$ sudo snap remove core18
$ sudo apt purge gnome-software-plugin-snap
$ sudo apt purge snapd squashfs-tools
```

最後檢查一下相關的目錄 `/snap`, `/var/snap`, `/var/lib/snapd`，應該都被清掉了。

---


[^1]: [Does btrfs support swap files?](https://btrfs.wiki.kernel.org/index.php/FAQ#Does_btrfs_support_swap_files.3F)
[^2]: [Swap file support](https://btrfs.wiki.kernel.org/index.php/Project_ideas#Swap_file_support)
[^3]: [Btrfs: implement swap file support](https://lwn.net/Articles/763949/)


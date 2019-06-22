上周在更新Manjaro Linux的时候误触了电源键，导致内核更新了一半系统强制关机，重启时正常进入grub但无法正常引导进入系统。

由于不想重装系统（一大堆环境和工具的配置还是相当繁琐的），加上初步判断应该仅仅是内核引导镜像没能正常安装导致的问题，所以决定先用liveUSB进行急救。

需要准备的工具：

- 一个使用较新版本Manjaro Linux的liveUSB（可以使用dd将镜像直接写入ｕ盘）
- 待修复设备需要联网环境（没有其实也不用担心，不过最好还是需要联网环境）

下面开始修复启动。

首先通过liveUSB启动，在liveUSB的中我们原先的系统文件是保存在电脑的磁盘上的，默认不会被挂载，所以我们先要把除了/home以外的系统目录挂载到当前的任意目录，我们选择挂载在/mnt中：

```bash
sudo mkdir /mnt/manjaro
sudo mount /dev/sda2 /mnt/manjaro # sda2为/分区所在设备，可以使用lsblk查看
```

随后是关键的一步，因为在UEFI下安装Manjaro Linux时我们都额外为`/boot/efi/`进行了单独的分区，所以我们这里也需要挂载它。默认挂载根目录时并不会挂载这个目录，因为它们不在同一个分区，我的efi目录根据lsblk显示位于/dev/sda1：

```bash
$ lsblk

NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 465.8G  0 disk
├─sda1   8:1    0   200M  0 part /boot/efi
├─sda2   8:2    0  50.8G  0 part /
└─sda3   8:3    0 414.8G  0 part /home
```

所以我们把efi目录也挂载进文件系统，否则内核无法重新安装：

```bash
sudo mount /dev/sda1 /mnt/manjaro/boot/efi
```

另外对于一些虚拟目录，例如/dev和/sys，我们也需要手动绑定，否则chroot后运行pacman会出错：

```bash
sudo mount --bind /dev /mnt/manjaro/dev
sudo mount --bind /proc /mnt/manjaro/proc
sudo mount --bind /sys /mnt/manjaro/sys
```

这样系统文件就准备完成了，现在我们在挂载目录下chroot，然后重新安装内核：

```bash
cd /mnt/manjaro
chroot .
pacman -S linux # 如果这一步报错，检查自己系统目录是否正确挂载，如果正确挂载则先运行pacman -S archlinux-keyring
# 内核重装完成后继续上次未完成的系统更新
pacman -Syu
```

如果你安装了不同版本的内核的话，这样只能修复系统默认版本的内核，对于Manjaro 18.0来说是4.14.x。
所以在grub引导界面我们需要选择对应的版本才能正常启动，如果想恢复其他版本的内核，可以在用默认版本内核启动后去Manjaro Settings Manager中重新安装。

这样就避免了重装修复了Manjaro的启动问题。

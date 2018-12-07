# 鸟哥的 Linux 私房菜 - 基础篇  学习笔记

This file is written to learn 《鸟哥的 Linux 私房菜 - 基础篇》 note.

## 强制更新 Linux 内核的分区表信息

```markdown
[root@study ~]# partprobe -s
```

## 内存空间 (swap) 建立

### 使用硬盘分区建立 swap

1. 先进行分区：GPT 格式的用 gdisk 分区，MBR 格式的用 fdisk 分区。

    ```markdown
    [root@study ~]# gdisk /dev/vda
    ... # 中间省略 ...
    [root@study ~]# partprobe
    [root@study ~]# lsblk
    NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    vda             252:0    0   40G  0 disk
    ... # 中间省略 ...
    `-vda6          252:6    0  512M  0 part  # 确定这里存在才行！
    ```

2. 分区格式化成 swap

    ```markdown
    [root@study ~]# mkswap /dev/vda6
    Setting up swapspace version 1, size = 524284 KiB
    no label, UUID=6b17e4ab-9bf9-43d6-88a0-73ab47855f9d
    [root@study ~]# blkid /dev/vda6
    /dev/vda6: UUID="6b17e4ab-9bf9-43d6-88a0-73ab47855f9d" TYPE="swap"
    # 确定格式化成功！且使用 blkid 确实可以抓到这个设备！
    ```

3. 载入系统

    ```markdown
    [root@study ~]# free
                total        used        free      shared  buff/cache   available
    Mem:        1275140      227244      330124      7804      717772      875536  # 实际内存
    Swap:       1048572      101340      947232                                    # swap 相关
    ... # 中间省略 ...

    [root@study ~]# swapon /dev/vda6
    [root@study ~]# free
                  total        used        free      shared  buff/cache   available
    Mem:        1275140      227940      329256        7804      717944      874752
    Swap:       1572856      101260     1471596  # shared 有增加
    ... # 中间省略 ...

    [root@study ~]# swapon -s
    Filename                 Type            Size    Used    Priority
    /dev/dm-1                partition       1048572 101260  -1
    /dev/vda6                partition       524284  0       -2
    # 上面列出目前使用的 swap 设备有哪些的意思！

    [root@study ~]# nano /etc/fstab
    UUID="6b17e4ab-9bf9-43d6-88a0-73ab47855f9d"  swap  swap  defaults  0  0
    # 写入指定的文件。
    ```

### 使用文件建立 swap

1. 使用 dd 命令建立一个文件。

    ```markdown
    [root@study ~]# dd if=/dev/zero of=/tmp/swap bs=1M count=128
    128+0 records in
    128+0 records out
    134217728 bytes (134 MB) copied, 1.7066 seconds, 78.6 MB/s

    [root@study ~]# ll -h /tmp/swap
    -rw-r--r--. 1 root root 128M Jun 26 17:47 /tmp/swap
    ```

2. 使用 mkswap 将新建的文件格式化为 swap 的文件格式。

    ```markdown
    [root@study ~]# mkswap /tmp/swap
    Setting up swapspace version 1, size = 131068 KiB
    no label, UUID=4746c8ce-3f73-4f83-b883-33b12fa7337c
    # 这是指令要特别小心，因为写错，将可能使文件系统挂掉！
    ```

3. 使用 swapon 将新建的 swap 文件启动。

    ```markdown
    [root@study ~]# swapon /tmp/swap
    [root@study ~]# swapon -s
    Filename            Type            Size    Used    Priority
    /dev/dm-1           partition       1048572 100380  -1
    /dev/vda6           partition       524284  0       -2
    /tmp/swap           file            131068  0       -3
    ```

4. 使用 swapoff 关掉 swap file, 并设置自动启动。

    ```markdown
    [root@study ~]# nano /etc/fstab
    /tmp/swap  swap  swap  defaults  0  0
    # 為何這裡不要使用 UUID 呢？這是因為系統僅會查詢區塊裝置 (block device) 不會查詢檔案！
    # 所以，這裡千萬不要使用 UUID, 不然系統會查不到喔！

    [root@study ~]# swapoff /tmp/swap /dev/vda6
    [root@study ~]# swapon -s
    Filename                                Type            Size    Used    Priority
    /dev/dm-1                               partition       1048572 100380  -1
    # 確定已經回復到原本的狀態了！然後準備來測試！

    [root@study ~]# swapon -a
    [root@study ~]# swapon -s
    # 最終你又會看正確的三個 swap 出現囉！這也才確定你的 /etc/fstab 設定無誤！
    ```

## 文件系统的特殊权限查看与操作

### 磁盘空间浪费问题

```markdown
[root@study ~]# ll -sh
total 7.1G
4.0K drwxrwxr-x  4 jiegui jiegui 4.0K 12月  7 19:36 ./
4.0K drwxr-xr-x 17 jiegui jiegui 4.0K 11月 24 17:19 ../
   0 -rw-rw-r--  1 jiegui jiegui    0 12月  7 19:36 000
4.0K -rw-rw-r--  1 jiegui jiegui    4 12月  7 19:36 111
 20K -rw-rw-r--  1 jiegui jiegui  14K 9月  15 20:12 index.html
4.0K -rw-rw-r--  1 jiegui jiegui 1.0K 11月 24 10:03 tm
```

### 使用 GNU 的 parted 进行磁盘分区

parted: 同时支持 GPT 和 MBR 的指令。

```markdown
[root@study ~]# parted [裝置] [指令 [參數]]
選項與參數：
指令功能：
    新增分割：mkpart [primary|logical|extended] [ext4|vfat|xfs] 開始 結束
    顯示分割：print
    刪除分割：rm [partition]
```

範例一：以 parted 列出目前本機的分割表資料

```markdown
[root@study ~]# parted /dev/vda print
Model: Virtio Block Device (virtblk)         <==磁碟介面與型號
Disk /dev/vda: 42.9GB                        <==磁碟檔名與容量
Sector size (logical/physical): 512B/512B    <==每個磁區的大小
Partition Table: gpt                         <==是 GPT 還是 MBR 分割
Disk Flags: pmbr_boot

Number  Start   End     Size    File system     Name                  Flags
 1      1049kB  3146kB  2097kB                                        bios_grub
 2      3146kB  1077MB  1074MB  xfs
 3      1077MB  33.3GB  32.2GB                                        lvm
 4      33.3GB  34.4GB  1074MB  xfs             Linux filesystem
 5      34.4GB  35.4GB  1074MB  ext4            Microsoft basic data
 6      35.4GB  36.0GB  537MB   linux-swap(v1)  Linux swap
```

```markdown
[root@study ~]# parted /dev/vda unit mb print
```

範例二：將 /dev/sda 這個原本的 MBR 分割表變成 GPT 分割表！(危險！危險！勿亂搞！無法復原！)

```markdown
[root@study ~]# parted /dev/sda print
Model: ATA QEMU HARDDISK (scsi)
Disk /dev/sda: 2148MB
Sector size (logical/physical): 512B/512B
Partition Table: msdos    # 確實顯示的是 MBR 的 msdos 格式喔！

[root@study ~]# parted /dev/sda mklabel gpt
Warning: The existing disk label on /dev/sda will be destroyed and all data on 
this disk will be lost. Do you want to continue?
Yes/No? y

[root@study ~]# parted /dev/sda print
# 你應該就會看到變成 gpt 的模樣！只是...後續的分割就全部都死掉了！
```

範例三：建立一個約為 512MB 容量的分割槽

```markdown
[root@study ~]# parted /dev/vda mkpart primary fat32 36.0GB 36.5GB
[root@study ~]# partprobe -s
[root@study ~]# lsblk /dev/vda7
[root@study ~]# mkfs -t vfat /dev/vda7
[root@study ~]# blkid /dev/vda7
/dev/vda7: SEC_TYPE="msdos" UUID="6032-BF38" TYPE="vfat"

[root@study ~]# nano /etc/fstab  # 添加自动挂载
UUID="6032-BF38"  /data/win  vfat  defaults   0  0
```

## 文件、文件系统的压缩、打包与备份

### 打包指令：tar

```markdown
[dmtsai@study ~]$ tar [-z|-j|-J] [cv] [-f new-filename] filename  <==打包與壓縮
[dmtsai@study ~]$ tar [-z|-j|-J] [tv] [-f filename]               <==察看檔名
[dmtsai@study ~]$ tar [-z|-j|-J] [xv] [-f filename] [-C folder]   <==解壓縮
選項與參數：
-c: 建立打包檔案，可搭配 -v 來察看過程中被打包的檔名(filename)
-t: 察看打包檔案的內容含有哪些檔名，重點在察看『檔名』就是了；
-x: 解打包或解壓縮的功能，可以搭配 -C (大寫) 在特定目錄解開
    特別留意的是， -c, -t, -x 不可同時出現在一串指令列中。
-z: 透過 gzip  的支援進行壓縮/解壓縮：此時檔名最好為 *.tar.gz
-j: 透過 bzip2 的支援進行壓縮/解壓縮：此時檔名最好為 *.tar.bz2
-J: 透過 xz    的支援進行壓縮/解壓縮：此時檔名最好為 *.tar.xz
    特別留意， -z, -j, -J 不可以同時出現在一串指令列中
-v: 在壓縮/解壓縮的過程中，將正在處理的檔名顯示出來！
-f filename: -f 後面要立刻接要被處理的檔名！建議 -f 單獨寫一個選項囉！(比較不會忘記)
-C 目錄:      這個選項用在解壓縮，若要在特定目錄解壓縮，可以使用這個選項。

其他後續練習會使用到的選項介紹：
-p(小寫): 保留備份資料的原本權限與屬性，常用於備份(-c)重要的設定檔
-P(大寫): 保留絕對路徑，亦即允許備份資料中含有根目錄存在之意；
--exclude=FILE: 在壓縮的過程中，不要將 FILE 打包！
```

#### 最常用的命令：

```markdown
luyu@ubuntu:~$ tar -cjf  filename.tar.bz2 files/folder   # 压缩
luyu@ubuntu:~$ tar -cJf  filename.tar.xz files/folder    # 压缩
luyu@ubuntu:~$ tar -jtvf filename.tar.bz2                # 查询
luyu@ubuntu:~$ tar -xf   filename.tar.bz2 -C new-folder  # 特定目录解压，解压格式自动识别
```

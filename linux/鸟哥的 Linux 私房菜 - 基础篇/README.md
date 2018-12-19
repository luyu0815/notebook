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

## 文件和文件系统的压缩、打包与备份

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

#### tar 备份 /etc 的资料在另外一部系统上复原，无法正常登系统

原因是因为 /etc/shadow 这个密码文件的 SELinux 类型在还原的时候被更改了，导致登陆系统程序无法顺利存取。

处理方法：

  - 在第一次复原系统后不要立即开机，先使用 restorecon -Rv /etc 自动修复一下 SELinux 的类型即可。
  - 透过各种方式进入系统，建立 /。autorelabel 文件，重新开机后系统会自动修复 SELinux 的类型，并且再次重新开机，之后就正常。
  - 救援方式登陆系统，修改 /etc/selinux/comfig 文件，将 SELinux 改成 permissive 模式，重新开机后系统就正常了。

### xfsdump 备份完整的文件系统，xfsrestore 文件系统还原

- xfsdump xfsrestore 只能备份还原 xfs 格式的文件系统。

#### xfsdump 备份文件系统

```markdown
[root@study ~]# xfsdump [-L S_label] [-M M_label] [-l #] [-f 備份檔] 待備份資料
[root@study ~]# xfsdump -I
選項與參數：
-L  ：xfsdump 會紀錄每次備份的 session 標頭，這裡可以填寫針對此檔案系統的簡易說明
-M  ：xfsdump 可以紀錄儲存媒體的標頭，這裡可以填寫此媒體的簡易說明
-l  ：是 L 的小寫，就是指定等級～有 0~9 共 10 個等級喔！ (預設為 0，即完整備份)
-f  ：有點類似 tar 啦！後面接產生的檔案，亦可接例如 /dev/st0 裝置檔名或其他一般檔案檔名等
-I  ：從 /var/lib/xfsdump/inventory 列出目前備份的資訊狀態

[root@study ~]# xfsdump -l 0 -L boot_all -M boot_all -f /srv/boot.dump /boot
[root@study ~]# ll /var/lib/xfsdump/inventory
[root@study ~]# xfsdump -I
[root@study ~]# xfsdump -l 1 -L boot_2 -M boot_2 -f /srv/boot.dump1 /boot  # 增量备份
[root@study ~]# xfsdump -I
```

#### xfsrestore 还原文件系统

```markdown
[root@study ~]# xfsrestore -I                                       <==用來察看備份檔案資料
[root@study ~]# xfsrestore [-f 備份檔] [-L S_label] [-s] 待復原目錄 <==單一檔案全系統復原
[root@study ~]# xfsrestore [-f 備份檔] -r 待復原目錄                <==透過累積備份檔來復原系統
[root@study ~]# xfsrestore [-f 備份檔] -i 待復原目錄                <==進入互動模式
選項與參數：
-I  ：跟 xfsdump 相同的輸出！可查詢備份資料，包括 Label 名稱與備份時間等
-f  ：後面接的就是備份檔！企業界很有可能會接 /dev/st0 等磁帶機！我們這裡接檔名！
-L  ：就是 Session 的 Label name 喔！可用 -I 查詢到的資料，在這個選項後輸入！
-s  ：需要接某特定目錄，亦即僅復原某一個檔案或目錄之意！
-r  ：如果是用檔案來儲存備份資料，那這個就不需要使用。如果是一個磁帶內有多個檔案，
      需要這東西來達成累積復原
-i  ：進入互動模式，進階管理員使用的！

# 1. 直接將資料給它覆蓋回去即可！
[root@study ~]# xfsrestore -f /srv/boot.dump -L boot_all /boot

# 2. 將備份資料在 /tmp/boot 底下解開！
[root@study ~]# mkdir /tmp/boot
[root@study ~]# xfsrestore -f /srv/boot.dump -L boot_all /tmp/boot
[root@study ~]# du -sm /boot /tmp/boot
109     /boot
99      /tmp/boot
# 咦！兩者怎麼大小不一致呢？沒關係！我們來檢查看看！

[root@study ~]# diff -r /boot /tmp/boot
Only in /boot: testing.img
# 看吧！原來是 /boot 我們有增加過一個檔案啦！
```

```markdown
# 僅復原備份檔內的 grub2 到 /tmp/boot2/ 裡頭去！
[root@study ~]# mkdir /tmp/boot2
[root@study ~]# xfsrestore -f /srv/boot.dump -L boot_all -s grub2 /tmp/boot2
```

- 仅还原部分文件的 xfsrestore 互动模式

```markdown
root@study ~]# mkdir /tmp/boot3
[root@study ~]# xfsrestore -f /srv/boot.dump -i /tmp/boot3
 ========================== subtree selection dialog ==========================

the following commands are available:
        pwd
        ls [ <path> ]
        cd [ <path> ]
        add [ <path> ]       # 可以加入復原檔案列表中
        delete [ <path> ]    # 從復原列表拿掉檔名！並非刪除喔！
        extract              # 開始復原動作！
        quit
        help

 -> ls
          455517 initramfs-3.10.0-229.el7.x86_64kdump.img
             138 initramfs-3.10.0-229.el7.x86_64.img
             141 initrd-plymouth.img
             136 symvers-3.10.0-229.el7.x86_64.gz
             135 config-3.10.0-229.el7.x86_64
             134 System.map-3.10.0-229.el7.x86_64
             133 .vmlinuz-3.10.0-229.el7.x86_64.hmac
         1048704 grub2/
             131 grub/

 -> add grub
 -> add grub2
 -> add config-3.10.0-229.el7.x86_64
 -> extract
 
[root@study ~]# ls -l /tmp/boot3
```

### cpio 备份任何东西，需要配合 find 使用

- 备份

```markdown
# find / | cpio -ocvB > /dev/st0
```

- 还原

```markdown
# cpio -idvc < /dev/st0
```

## vim 编辑器

### 三个模式：

- 一般指令模式（command mode）

  [Ctrl] + [f] : 向下翻页
  
  [Ctrl] + [b] : 向上翻页
  
  数字 0 : 移动到当前行的最前面
  
  $ : 移动到当前行的最后面
  
  G : 移动到文件的最后一行
  
  gg : 移动到文件的第一行
  
  n<Enter> : n 为数字，光标向下移动 n 行
  
  :n1,n2s/word1/word2/g : n1 与 n2 为数字，在第 n1 与 n2 行之间查找 word1 这个
  字符串，并将该字符串替换成 word2
  
  :1,$s/word1/word2/g : 从第一行到最后一行查找 word1 字符串，并将该字符串替换成 word2
  
  :1,$s/word1/word2/gc : 从第一行到最后一行查找 word1 字符串，并将该字符串替换成
  word2 且提示是否替换
  
  x/X : x 向后删除一个字符，X 向前删除一个字符
  
  dd : 删除当前整行
  
  ndd : 删除光标所在的向下 n 行之间查找
  
  yy : 复制当前行
  
  nyy : 复制光标所在行到向下 n 行
  
  p/P : p 在光标下一行粘贴，P 在光标上一行粘贴
  
  u : 撤销
  
  [Ctrl]+r : 恢复（与撤销相对）
  
  . : 小数点，重复上一个动作

- 编辑模式（insert mode）

  i : 光标所在处插入
  
  I : 光标所在行的第一个非空白字符处开始插入/a/A/o/O/r/R
  
  a : 光标所在处的下一个字符开始插入
  
  A : 光标所在行的最后一个字符后开始插入
  
  o : 光标所在行的下一行插入
  
  O : 光标所在行的上一行插入
  
  r/R : 替换，r 替换一次，R 一直替换

- 指令列命令模式（command-line mode）

  ZZ: 如果文件没有更新则不保存离开，如果文件已经更改则保存离开
  
  :w [filename] : 另存为
  
  :r [filename] : 将另外一个文件的内容插入到当前文件光标的后面
  
  :n1,n2 w [filename] : 将 n1 到 n2 的内容存到另一个文件中
  
  :! command : 暂时离开 vim 执行 command 的显示结果
  
  :set nu : 显示行号，（:set nonu 取消显示）

### vim 的额外功能

#### 区块选择

  v : 字符选择
  
  V : 行选择
  
  [Ctrl]+v : 区块选择
  
  y/d/p : 将选择的区域 复制/删除/粘贴

#### 多文件编辑

  :n : 编辑下一个文件
  
  :N : 编辑上一个文件
  
  :files : 列出目前这个 vim 的打开的所有文件

#### vim 多窗口视图

| x | x |
|-|-|
|:sp| 开启 |
|[Ctrl]+w+j| 按键 |













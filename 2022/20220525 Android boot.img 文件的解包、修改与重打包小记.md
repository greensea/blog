本文记录 Andoird 的 boot.img 文件的组成、解包、修改以及重打包方法。

Android 的文件由以下内容组成：

	* kernel，文件名一般为 zImage
	* ramdisk，文件名一般为 initrd.gz。该文件解压之后是一个 cpio archive 文件（亦是 cramfs 文件系统？）
	* second，这是第二阶段启动文件系统

此外，上述文件的地址偏移量也是在打包的时候必须提供的信息。


---------------

使用 abootimg 可以轻松地解压。

https://github.com/ggrandou/abootimg

----------------

## 处理过程

	* 使用 abootimg 将 boot.img 解包，得到 kernel、ramdisk、second 等文件
	* 使用 cpio 将 ramdisk 文件解包
	* 修改 ramdisk 中的文件
	* 使用 cpio 将修改好的文件重新打包成新的 ramdisk 文件
	* 使用 abootimg 将新的 ramdisk 文件更新到 boot.img 中
	* 使用 fastboot 刷入 abootimg


注意，cpio 解包和打包过程均应使用 root 用户。

### 使用 cpio 解包 ramdisk
```
gzip -d boot.img-ramdisk.gz
mkdir ramdisk
cd ramdisk
cpio -i -d -H newc -F ../boot.img-ramdisk --no-absolute-filenames
```

### 使用 cpio 制作 ramdisk
```
# 在 ramdisk 目录下
sudo find . | sudo cpio --create --format='newc' >../new-boot.img-ramdisk
```

### 使用 abootimg 将新的 ramdisk 更新到 boot.img 中
```
abootimg -u boot.img -r new-boot.img-ramdisk
```

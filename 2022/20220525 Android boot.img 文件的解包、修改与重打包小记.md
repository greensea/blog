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

### 摘录——Android 系统在连接电源时自动开机的方法

https://android.stackexchange.com/questions/20021/automatically-power-on-android-when-the-charger-is-connected

```
fastboot oem off-mode-charge 0 is the genuine method if your device supports. It's Google's recommended method but not all OEMs/vendors implement the command in bootloader. Or on some devices it's reset on next reboot. If off-mode-charge is disabled, bootloader won't pass androidboot.mode=charger commandline parameter to kernel when charger is inserted, so device boots normally.

Otherwise when ro.bootmode property is set to charger on boot, init doesn't continue the normal boot process. Instead limited number of services are started and charging animation is displayed. So you can instruct init to reboot the device whenever charger mode is detected. Create a new .rc file or edit any existing one:

# /system/etc/init/off_mode_charge.rc

on charger
    setprop sys.powerctl reboot,leaving-off-mode-charging
Or execute reboot binary:

on charger
    exec - -- /system/bin/reboot leaving-off-mode-charging
But if SELinux is enforcing, stock policy may not let init execute /system/bin/reboot. So use Magisk's context (or whatever rooting solution you use):

on charger
    exec u:r:magisk:s0 -- /system/bin/reboot
Don't forget to set permissions on *.rc file (chown 0.0, chmod 0644, chcon u:object_r:system_file:s0).

It's also possible to continue boot process instead of restarting the device by replacing class_start charger with trigger late-init in /init.rc file:

on charger
    #class_start charger
    trigger late-init
Or by setting property sys.boot_from_charger_mode:

on charger
    setprop sys.boot_from_charger_mode 1
This method should work on all devices irrespective of OEM as it doesn't depend on vendor-specific charging binaries like playlpm, battery_charging, chargeonlymode, zchgd, kpoc_charger and so on.
Also replacing binaries of important services like healthd - which take care of a lot of things related to battery, storage etc. - is not a good idea. In this case if the service runs both in charger and normal mode, device may get into bootloop.
On non-System-as-Root devices it's not necessary to modify /system partition (e.g. if you don't want to break dm-verity for OTA updates to work). Simply unpack boot.img and edit /init.rc file in ramdisk.
```
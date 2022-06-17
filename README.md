# elinuxCore 100ask-t113-pro 最小T113嵌入式Linux系统

## 配置Host开发环境



## 设置交叉编译工具链

```ba
export PATH=$PATH:/home/book/eLinuxCore_100ask-t113-pro/toolchain/gcc-linaro-7.2.1-2017.11-x86_64_arm-linux-gnueabi/bin
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
```



## 编译 bootloader



### 编译 boot0 spl



### 编译 uboot

```bash
book@100ask:~/eLinuxCore_100ask-t113-pro/u-boot$ make sun8iw20p1_auto_defconfig
book@100ask:~/eLinuxCore_100ask-t113-pro/u-boot$ make -j8
book@100ask:~/eLinuxCore_100ask-t113-pro/u-boot$ cp u-boot-sun8iw20p1.bin ../
```



## 编译kernel和设备树

### 编译内核镜像

```
book@100ask:~/eLinuxCore_100ask-t113-pro/linux$ make sun8iw20p1smp_defconfig
book@100ask:~/eLinuxCore_100ask-t113-pro/linux$ make zImage -j8
book@100ask:~/eLinuxCore_100ask-t113-pro/linux$ cp arch/arm/boot/zImage ../
```

* 打包生成特定格式

```bash
#buildroot kernel boot images.
mkbootimg --kernel zImage  --ramdisk  ramdisk.img --board sun8iw20p1 --base  0x40200000 --kernel_offset  0x0 --ramdisk_offset  0x01000000 -o  boot.img
```

### 编译设备树文件

```bash
book@100ask:~/eLinuxCore_100ask-t113-pro/linux$ make dtbs
  DTC     arch/arm/boot/dts/sun8iw20p1-t113-100ask-t113-pro.dtb
  
book@100ask:~/eLinuxCore_100ask-t113-pro/linux$ cp  arch/arm/boot/dts/sun8iw20p1-t113-100ask-t113-pro.dtb ../
```

### 编译内核模块

```bash
book@virtual-machine:~/eLinuxCore_100ask-t113-pro/linux$ make modules -j8
```



## 分阶段打包

### 打包boot_package.fex



dragonsecboot  -pack boot_package.cfg

```bash
book@virtual-machine:~/buildroot_100ask_t113-pro/br2t113pro/board/100ask/t113-pro$ cat boot_package.cfg
[package]
;item=Item_TOC_name,         Item_filename,
item=u-boot, u-boot-sun8iw20p1.bin
item=optee, optee.fex
item=logo, bootlogo.bmp.lzma
item=dtb, sun8iw20p1-t113-100ask-t113-pro.dtb
```



### 生成 env.fex

* TF卡镜像 启动环境变量

```bash
book@virtual-machine:~/buildroot_100ask_t113-pro/br2t113pro/board/100ask/t113-pro$ cat env-sd.cfg
cma=16M
console=ttyS3,115200
dsp0_partition=dsp0
earlycon=uart8250,mmio32,0x05000000
fdtcontroladdr=44884e70
init=/sbin/init
initcall_debug=0
keybox_list=widevine,ec_key,ec_cert1,ec_cert2,ec_cert3,rsa_key,rsa_cert1,rsa_cert2,rsa_cert3
loglevel=8
mmc_root=/dev/mmcblk0p5
rootfstype=ext4,rw
setargs_mmc=setenv  bootargs earlycon=${earlycon} clk_ignore_unused initcall_debug=${initcall_debug} console=${console} loglevel=${loglevel} root=${mmc_root}  init=${init} partitions=${partitions} cma=${cma} snum=${snum} mac_addr=${mac} wifi_mac=${wifi_mac} bt_mac=${bt_mac} specialstr=${specialstr} gpt=1
#kernel command arguments
root_partition=rootfs
boot_partition=boot
#uboot system env config
bootdelay=2
kernel=boot.img
mmc_boot_part=4
mmc_dev=0
boot_check=sunxi_card0_probe;mmcinfo;mmc part
boot_mmc=fatload mmc ${mmc_dev}:${mmc_boot_part} 45000000 ${kernel}; bootm 45000000
boot_dsp0=fatload mmc 0:6  43000000 dsp0.fex; bootr 43000000 0 0
bootcmd=run setargs_mmc  boot_check  boot_dsp0 boot_mmc

```


mkenvimage -r -p 0x00 -s 0x20000 -o env.fex env-sd.cfg

* spi nand启动环境变量

```bash
book@virtual-machine:~/buildroot_100ask_t113-pro/br2t113pro/board/100ask/t113-pro$ cat env-spinand.cfg

#kernel command arguments
earlycon=uart8250,mmio32,0x05000000
initcall_debug=0
console=ttyS3,115200
nand_root=ubi0_5
mmc_root=/dev/mmcblk0p5
mtd_name=sys
rootfstype=ubifs,rw
init=/sbin/init
loglevel=8
cma=8M
mac=
wifi_mac=
bt_mac=
specialstr=
keybox_list=widevine,ec_key,ec_cert1,ec_cert2,ec_cert3,rsa_key,rsa_cert1,rsa_cert2,rsa_cert3
dsp0_partition=dsp0
#set kernel cmdline if boot.img or recovery.img has no cmdline we will use this
setargs_nand=setenv bootargs ubi.mtd=${mtd_name} earlycon=${earlycon} clk_ignore_unused initcall_debug=${initcall_debug} console=${console} loglevel=${loglevel} root=${nand_root} rootfstype=${rootfstype} init=${init} partitions=${partitions} cma=${cma} snum=${snum} mac_addr=${mac} wifi_mac=${wifi_mac} bt_mac=${bt_mac} specialstr=${specialstr} gpt=1
setargs_nand_ubi=setenv bootargs ubi.mtd=${mtd_name} earlycon=${earlycon} clk_ignore_unused initcall_debug=${initcall_debug} console=${console} loglevel=${loglevel} root=${nand_root} rootfstype=${rootfstype} init=${init} partitions=${partitions} cma=${cma} snum=${snum} mac_addr=${mac} wifi_mac=${wifi_mac} bt_mac=${bt_mac} specialstr=${specialstr} gpt=1
setargs_mmc=setenv  bootargs earlycon=${earlycon} clk_ignore_unused initcall_debug=${initcall_debug} console=${console} loglevel=${loglevel} root=${mmc_root}  init=${init} partitions=${partitions} cma=${cma} snum=${snum} mac_addr=${mac} wifi_mac=${wifi_mac} bt_mac=${bt_mac} specialstr=${specialstr} gpt=1
#nand command syntax: sunxi_flash read address partition_name read_bytes
#0x4007f800 = 0x40080000(kernel entry) - 0x800(boot.img header 2k)
boot_dsp0=sunxi_flash read 43000000 ${dsp0_partition};bootr 43000000 0 0
boot_normal=sunxi_flash read 43000000 boot;bootm 43000000
boot_recovery=sunxi_flash read 43000000 recovery;bootm 43000000
boot_fastboot=fastboot

#uboot system env config
bootdelay=0
#default bootcmd, will change at runtime according to key press
#default nand boot
bootcmd=run setargs_nand boot_dsp0 boot_normal

```

mkenvimage -r -p 0x00 -s 0x20000 -o env.fex env-spinand.cfg



## 编译busybox最小文件系统



## 完整打包生成 TF卡 sdcard.img系统镜像并烧写

### 打包系统镜像

genimage

```bash
image 100ask-t113-pro_sdcard.img {
        hdimage{
                partition-table-type = "hybrid"
                gpt-location = 1M
        }
        partition boot0 {
                in-partition-table = "no"
                image = "boot0_sdcard.fex"
                offset = 8K
        }

        partition boot-packages {
                in-partition-table = "no"
                image = "boot_package.fex"
                offset = 16400K
        }

        partition boot-resource {
                image = "boot-resource.fex"
        }

        partition env {
                image = "env.fex"
                size = 128k
        }

        partition env-redund {
                image = "env.fex"
                size = 128k
        }

        partition boot {
                partition-type = 0xC
                bootable = "true"
                image = "boot.vfat"
        }

        partition rootfs {
                partition-type = 0x83
                image = "rootfs.ext4"
        }
        partition dsp0 {
                partition-type = 0xC
                image = "dsp.vfat"
        }
}
image dsp.vfat{
        vfat{
                files = {"dsp0.fex"}
        }
        size = 1M
}

image boot.vfat {
        vfat {
                files = {
                        "boot.img",
                        "zImage",
                        "sun8iw20p1-t113-100ask-t113-pro.dtb"
                }
        }
        size = 32M
}

```



### 烧写生成的系统镜像并启动

wind32diskimage



## 完整打包生成SPI NAND系统镜像 并烧写

### 打包Spi NAND系统镜像



### 烧写生成的系统镜像 并启动






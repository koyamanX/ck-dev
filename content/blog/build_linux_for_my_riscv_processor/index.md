---
title: "Building Linux kernel for my RISC-V SoC(RV32X\_SoC)"
date: 2021-03-16T00:11:05+09:00
author: "@koyamanX"
categories: ["Linux"]
tags: ["RISC-V", "Linux", "CPU"]
draft: false
---

## 概要
自作のRISC-V SoC向けのLinuxカーネルのビルドフローをメモとして残す。
アーキテクチャはRV32IMAである。
特権モードは(M, S, U)をサポートしており、SV32ページングを実装している。
クロスコンパイラとしてriscv32-unknown-linux-gnu-gcc(version 10.1.0)がインストールされていることとする。
<!--more-->
## Build Linux Kernel
### Linuxのレポジトリをクローンする。
バージョンはv5.11とする。
```bash
git clone https://github.com/torvalds/linux
git checkout v5.11
```
### 依存ツールをインストールする。
```bash
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
                 gawk build-essential bison flex texinfo gperf libtool patchutils bc \
				                  zlib1g-dev libexpat-dev git
```


### コンフィグレーションを行う。
`lib/Kconfig.debug`の`menu "Kernel hacking"`の次の行に以下を追記する。
これで、`Kernel hacking`の設定項目に`Enable Early PRINTK`が追加される。
```
config EARLY_PRINTK                                                                                                                               
	bool "Enable Early PRINTK"
```

`ARCH=riscv`でRISC-Vをアーキテクチャとして指定する。
`arch/$ARCH/configs`にデフォルトのdefconfigが用意されている。
`defconfig`でデフォルトのコンフィグレーションをもとに`.config`を作る。
`menuconfig`でdefconfigした情報をもとに、手動でコンフィグレーションを行う。
```bash
cd linux
make ARCH=riscv CROSS_COMPILE=riscv32-unknown-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv32-unknown-linux-gnu- menuconfig
```
```
Platform type -> Base ISA => RV32I
Platform type -> Symmetric Multi-Processing => off
Platform type -> supported PMU type => Base Performance Monitoring Unit => off
Platform type -> FPU support => off
Platform type -> Emit compressed instruction when building Linux => off (UEFI runtime supportのdepends、オフにする)
Boot options -> UEFI runtime support => off (Compress instruction emmitionを無効化できるようになる。)
Kernel features -> Timer frequency => いい感じにする
Networking support => off
Boot options -> Build-in kernel command line => earlycon=sbi
Kernel hacking -> ログレベルを最大にする。
```

#### defconfig
defconfigを保存する。
```
make savedefconfig
cp defconfig arch/riscv/configs/rv32xsoc_defconfig
```
defconfigを読み込み、.configを作成する。
```
make ARCH=riscv CROSS_COMPILE=riscv32-unknown-linux-gnu- rv32xsoc_defconfig
```

```
CONFIG_SYSVIPC=y
CONFIG_NO_HZ_IDLE=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_SCHED=y
CONFIG_CFS_BANDWIDTH=y
CONFIG_CGROUP_BPF=y
CONFIG_NAMESPACES=y
CONFIG_USER_NS=y
CONFIG_CHECKPOINT_RESTORE=y
CONFIG_BLK_DEV_INITRD=y
CONFIG_EXPERT=y
# CONFIG_SYSFS_SYSCALL is not set
CONFIG_BPF_SYSCALL=y
CONFIG_SOC_SIFIVE=y
CONFIG_SOC_VIRT=y
CONFIG_ARCH_RV32I=y
CONFIG_CMODEL_MEDANY=y
# CONFIG_RISCV_ISA_C is not set
# CONFIG_FPU is not set
CONFIG_CMDLINE="earlycon=sbi"
# CONFIG_EFI is not set
CONFIG_JUMP_LABEL=y
CONFIG_ARCH_MMAP_RND_BITS=17
# CONFIG_COMPAT_32BIT_TIME is not set
CONFIG_MODULES=y
CONFIG_MODULE_UNLOAD=y
CONFIG_PAGE_REPORTING=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
# CONFIG_BLK_DEV is not set
CONFIG_BLK_DEV_SD=y
CONFIG_SCSI_VIRTIO=y
CONFIG_ATA=y
CONFIG_SATA_AHCI_PLATFORM=y
CONFIG_INPUT_MOUSEDEV=y
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y
CONFIG_SERIAL_OF_PLATFORM=y
CONFIG_SERIAL_EARLYCON_RISCV_SBI=y
CONFIG_HVC_RISCV_SBI=y
CONFIG_VIRTIO_CONSOLE=y
CONFIG_HW_RANDOM=y
CONFIG_HW_RANDOM_VIRTIO=y
# CONFIG_HWMON is not set
CONFIG_DRM=y
CONFIG_BACKLIGHT_CLASS_DEVICE=y
CONFIG_FRAMEBUFFER_CONSOLE=y
# CONFIG_USB_SUPPORT is not set
# CONFIG_VIRTIO_MENU is not set
# CONFIG_VHOST_MENU is not set
# CONFIG_IOMMU_SUPPORT is not set
CONFIG_RPMSG_VIRTIO=y
CONFIG_EXT4_FS=y
CONFIG_EXT4_FS_POSIX_ACL=y
CONFIG_AUTOFS4_FS=y
CONFIG_MSDOS_FS=y
CONFIG_VFAT_FS=y
CONFIG_TMPFS=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_KEYS=y
CONFIG_CRYPTO_DEV_VIRTIO=y
CONFIG_CRC_ITU_T=y
CONFIG_CRC7=y
CONFIG_EARLY_PRINTK=y
CONFIG_PRINTK_TIME=y
CONFIG_PRINTK_CALLER=y
CONFIG_CONSOLE_LOGLEVEL_DEFAULT=15
CONFIG_CONSOLE_LOGLEVEL_QUIET=15
CONFIG_MESSAGE_LOGLEVEL_DEFAULT=7
CONFIG_DYNAMIC_DEBUG=y
CONFIG_FRAME_WARN=2048
CONFIG_DEBUG_FS=y
CONFIG_DEBUG_PAGEALLOC=y
CONFIG_SCHED_STACK_END_CHECK=y
CONFIG_DEBUG_VM=y
CONFIG_DEBUG_VM_VMACACHE=y
CONFIG_DEBUG_VM_RB=y
CONFIG_DEBUG_VM_PGFLAGS=y
CONFIG_DEBUG_VIRTUAL=y
CONFIG_DEBUG_MEMORY_INIT=y
CONFIG_SOFTLOCKUP_DETECTOR=y
CONFIG_WQ_WATCHDOG=y
CONFIG_DEBUG_TIMEKEEPING=y
CONFIG_DEBUG_RT_MUTEXES=y
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_MUTEXES=y
CONFIG_DEBUG_RWSEMS=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_STACKTRACE=y
CONFIG_DEBUG_LIST=y
CONFIG_DEBUG_PLIST=y
CONFIG_DEBUG_SG=y
CONFIG_RCU_EQS_DEBUG=y
CONFIG_DEBUG_BLOCK_EXT_DEVT=y
# CONFIG_FTRACE is not set
# CONFIG_RUNTIME_TESTING_MENU is not set
CONFIG_MEMTEST=y
```

コンフィグレーションは以下のコマンドで削除できる。
```bash
make mrproper
```

### Linuxのビルドを行う。
```bash
make CFLAGS="-march=rv32ima -mabi=ilp32" LDFLAGS="-march=rv32ima -mabi=ilp32" ARCH=riscv CROSS_COMPILE=riscv32-unknown-linux-gnu- all -j4
```

### OpenSBIのPayloadとしてビルドする
```bash
cd rv32x_dev/software/opensbi
make CROSS_COMPILE=riscv32-unknown-elf- PLATFORM=rv32xsoc FW_PAYLOAD_PATH=~/linux/arch/riscv/boot/Image FW_FDT_PATH=~/rv32x_dev/software/bootrom/rv32xsoc.dtb
```

### シミュレーターで実行
```bash
clear;./rv32x_simulation ../software/opensbi/build/platform/rv32xsoc/firmware/fw_payload.elf --no-sim-exit --no-log 2>/dev/null
```

### References
- [https://dtyler.io/articles/2017/12/29/about-kernel-make-config/](https://dtyler.io/articles/2017/12/29/about-kernel-make-config/)
- [http://jborza.com/emulation/2020/04/16/updating-riscv-environment.html](http://jborza.com/emulation/2020/04/16/updating-riscv-environment.html)
- [https://www.atmarkit.co.jp/ait/articles/0808/28/news129_2.html](https://www.atmarkit.co.jp/ait/articles/0808/28/news129_2.html)
- [https://cstmize.hatenablog.jp/entry/2019/10/14/_QEMU%2BOpenSBI%28boot_loader%29%E3%81%A7linux_kernel%E3%81%AE%E8%B5%B7%E5%8B%95](https://cstmize.hatenablog.jp/entry/2019/10/14/_QEMU%2BOpenSBI%28boot_loader%29%E3%81%A7linux_kernel%E3%81%AE%E8%B5%B7%E5%8B%95)
- [https://riscv.org/wp-content/uploads/2019/12/Summit_bootflow.pdf](https://riscv.org/wp-content/uploads/2019/12/Summit_bootflow.pdf)
- [https://risc-v-getting-started-guide.readthedocs.io/en/latest/linux-qemu.html#prerequisites](https://risc-v-getting-started-guide.readthedocs.io/en/latest/linux-qemu.html#prerequisites)

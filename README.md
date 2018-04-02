# AutoYaST
无人值守安装SUSE的服务器配置方法和文件

# 准备好相关的配置文件，按实际的情况进行修改
1. dhcpd.conf   -- DHCP配置文件
2. message      -- Linux启动引导镜像显示文件
3. default      -- 用于 x86 的 BIOS 启动引导镜像配置文件
4. grub.cfg     -- 用于 x86 的 UEFI 文件引导镜像配置文件
5. autoinst.xml -- AutoYast 配置成功后，生成控制操作系统安装 XML 配置文件 

# 按Conf_AutoYaSH.sh文件中的说明来顺利进行服务器配置
## 安装相关的模块
```shell
instModule {
  zypper install -y dhcp*
  zypper install -y tftp
  zypper install -y vsftpd
#syslinux包提供了一个非常有用的文件:/usr/share/syslinux/pxelinux.0	
  zypper install -y syslinux
}
```

## 挂载ISO文件，mount完才能备份
```shell
MountISO{
  mkdir -p /srv/install/x86/sles12/sp0/cd1
  mount  -o loop -t iso9660 /install/iso/SLE-12-Server-DV.iso /srv/install/x86/sles12/sp0/cd1/
}
```

## 备份相关的文件系统
```shell
BackupFile {
  cp /etc/dhcpd.conf ./dhcpd.conf.$today.bak
  cp -f /srv/install/x86/sles12/sp0/cd1/boot/x86_64/loader/message ./message.$today.bak
  cp -f /srv/install/x86/sles12/sp0/cd1/boot/x86_64/loader/isolinux.cfg ./default.$today.bak
  cp -f /srv/install/x86/sles12/sp0/cd1/EFI/BOOT/grub.cfg ./grub.cfg.$today.bak
  cp -f /root/autoinst.xml ./autoinst.xml.$today.bak
}
```

## 配置DHCP服务
```shell
confDHCP {
  cp -f dhcpd.conf /etc/dhcpd.conf
#启动DHCP服务
  yast dhcp-server
}
```

## 使用 YaST 设置NFS安装服务器
```shell
confNFS {
#挂载目录为/srv/install/x86/sles12/sp0/ + #挂载目录为/srv/install/x86/sles12/sp0/cd1/
  yast instserver
  cp -f autoinst.xml /srv/install/x86/sles12/sp0/
  showmount -e localhost
}
```

## 使用 YaST 设置 TFTP 服务器
```shell
confTFTP {
  yast tftp-server
#在 /srv/tftpboot 中创建一个结构以支持各种选项
#采用 64 位 UEFI 固件的 PC，在 UEFI 开机模式下只能运行 64 位操作系统引导程序
#在 Legacy 开机模式（即 BIOS 兼容开机模式）下，通常不区分操作系统的比特数
#大多数 Linux 发行版已使用 GRUB 作为 UEFI 下的引导程序
  mkdir -p /srv/tftpboot/BIOS/x86
  mkdir -p /srv/tftpboot/BIOS/x86/pxelinux.cfg
  mkdir -p /srv/tftpboot/EFI/x86/boot
  mkdir -p /srv/tftpboot/EFI/aarch64/boot
  mkdir -p /srv/install/aarch64/sles12/sp0/cd1
}
```

## 复制BOOT引导所需文件
```shell
confBOOT {
#将 x86 BIOS 和 UEFI 引导所需的 kernel 、 initrd 和 message 文件复制到相应的位置
#Linux启动引导镜像：linux内核(linux二进制文件) + 根文件系统(initrd二进制文件）+启动显示信息(message)
  cd /srv/install/x86/sles12/sp0/cd1/boot/x86_64/loader/ && cp -a linux initrd message /srv/tftpboot/BIOS/x86/
  cd /srv/install/x86/sles12/sp0/cd1/boot/x86_64/loader/ && cp -a linux initrd /srv/tftpboot/EFI/x86/boot
  cp -f message /srv/tftpboot/BIOS/x86/message
}
```

## 用于 x86 的 BIOS
```shell
#PXE启动引导镜像(Bootstrap file：pxelinux.0)+启动菜单配置项(pxelinux.cfg文件夹下的default文件)
#启动菜单配置项default引用了 2 个重要的文件：linux和initrd。
#将 pxelinux.0 复制到 TFTP 文件夹并为配置文件准备一个子文件夹
confBIOS {
  cp /usr/share/syslinux/pxelinux.0 /srv/tftpboot/BIOS/x86/
# cp /srv/install/x86/sles12/sp0/cd1/boot/x86_64/loader/isolinux.cfg /srv/tftpboot/BIOS/x86/pxelinux.cfg/default
  cp -f default /srv/tftpboot/BIOS/x86/pxelinux.cfg/default
}
```

## 用于 x86 的 UEFI 文件
```shell
#复制 UEFI 引导所需的所有 grub2 文件(bootx64.efi + grub.efi + MokManager.efi)
#将内核和 initrd 文件复制到目录结构(linux + initrd)
confUEFI {
  cd /srv/install/x86/sles12/sp0/cd1/EFI/BOOT && cp -a bootx64.efi grub.efi MokManager.efi /srv/tftpboot/EFI/x86/
# cp /srv/install/x86/sles12/sp0/cd1/EFI/BOOT/grub.cfg /srv/tftpboot/EFI/x86/grub.cfg
  cp -f grub.cfg /srv/tftpboot/EFI/x86/grub.cfg
}
```

## 用于 AARCH64 的 UEFI 文件
```shell
#复制 UEFI 引导所需的所有 grub2 文件(bootx64.efi + grub.efi + MokManager.efi)
#将内核和 initrd 文件复制到目录结构(linux + initrd)
confAARCH64 {
  cd /srv/install/aarch64/sles12/sp0/cd1/EFI/BOOT && cp -a bootaa64.efi /srv/tftpboot/EFI/aarch64/
# cp /srv/install/x86/sles12/sp0/cd1/EFI/BOOT/grub.cfg /srv/tftpboot/EFI/x86/grub.cfg
  cp -f grub.cfg /srv/tftpboot/EFI/x86/grub.cfg
}
```

# 配置网络引导
接下来为将要安装系统的机器 配置网络引导，配置将 PXE 选项包含在 BIOS 引导序列中
1. 华硕主板：BIOS--高级--内置设备--Realtek PXE OPROM

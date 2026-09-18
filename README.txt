The 10GbE controller (Realtek R8127) on my Beelink Mini NAS Wildcat Lake 304 (running proxmox 9.2.11) doesn't work because of the incorrect driver (r8169) being installed by default on proxmox

Transferring my data from the mini PC to a proper 12-bay NAS running Unraid (4x8TB SAS drive array + 1TB M.2 SSD cache) was saturating the 1 gigabit network link (~125MB/s max) and the 2x8TB ZFS mirrored SATA drives in proxmox should be able to put out ~250MB/s combined read. After these fixes I can max out the transfer rate to the drives' limit over the 10Gb link 

#Proxmox version info:
    Kernel Version: Linux 7.0.14-17-pve (2026-09-10T10:16Z)
    Boot Mode: EFI
    Manager Version: pve-manager/9.2.11/f6997e698c7933ea

#My hardware relevant for this:
    - 10Gtek 𝟭.𝟮𝟱/𝟮.𝟱/𝟱/𝟭𝟬𝗚-𝗧 𝗦𝗙𝗣+ 𝘁𝗼 𝗥𝗝𝟰𝟱 CAT.6a Copper Transceiver, Auto-Negotiation SFP+ Ethernet Module, up to 30-Meter, for Cisco SFP-10G-T-X, Netgear and More (https://a.co/d/07jabzrL)
    - 10Gtek 10Gb PCI-E NIC Network Card, Dual SFP+ Port, with Intel 82599ES Controller, PCI Express Ethernet LAN Adapter Support Windows Server/Linux/VMware, Compare to Intel X520-DA2(E10G42BTDA) (https://a.co/d/0iffu21G)
    - Beelink ME Pro 2-Bay AI NAS Mini PC Intel® Wildcat Lake 304 (https://www.bee-link.com/products/beelink-me-pro-2-bay-304)       - 3x Cat6 RJ45 ethernet cables to link  mini-PC <-10G-> NAS <-1G-> router <-2.5G-> mini-PC
    - Adtran router provided by my ISP: single 1x2.5G + 4x1G LAN ports. The NAS (running Unraid , mini-PC, and openmediavault VM 


#From https://www.realtek.com/Download/List?cate_id=584  
#get "10G Ethernet LINUX driver r8127 for kernel up to 7.1" --> r8127-11.016.00.tar.bz2  

#Installation from proxmox node shell:  
cd /tmp
wget https://github.com/schmore/realtek-rtl8127/raw/refs/heads/main/r8127-11.016.00.tar.bz2
bunzip2 r8127-11.016.00.tar.bz2
tar -xf r8127-11.016.00.tar
cd r8127-11.016.00
./autorun.sh
    Check old driver and unload it.
    rmmod r8169
    Build the module and install
    warning: pahole version differs from the one used to build the kernel
      The kernel was built with: 130
      You are using:             0
    Skipping BTF generation for r8127.ko due to unavailability of vmlinux
    Warning: modules_install: missing 'System.map' file. Skipping depmod.
    Backup r8169.ko
    rename r8169.ko to r8169.bak
    DEPMOD 7.0.14-17-pve
    load module r8127
    Updating initramfs. Please wait.
    update-initramfs: Generating /boot/initrd.img-7.0.14-17-pve
    dracut-install: Failed to find module 'r8169' /lib/modules/7.0.14-17-pve/kernel/drivers/net/ethernet/realtek/r8169.bak
    Running hook script 'zz-proxmox-boot'..
    Re-executing '/etc/kernel/postinst.d/zz-proxmox-boot' in new private mount namespace..
    No /etc/kernel/proxmox-boot-uuids found, skipping ESP sync.
    Completed.

#Now verify the new driver is loaded:  
root@node: lspci -nnk
58:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8127 10GbE Controller [10ec:8127] (rev 05)
      Subsystem: Realtek Semiconductor Co., Ltd. Device [10ec:0123]
      Kernel driver in use: r8127
      Kernel modules: r8127

#Make sure the old driver is not used on reboot:  
echo "blacklist r8169" | tee /etc/modprobe.d/blacklist-r8169.conf

#Optional - Turn off aspm because it may cause instability:  
echo "options r8127 aspm=0" | tee /etc/modprobe.d/r8127.conf

update-initramfs -u -k all
reboot

#Test the controller is working:
root@node:~# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: nic0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 78:55:36:0b:86:94 brd ff:ff:ff:ff:ff:ff
    altname enx7855360b8694
3: nic1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master vmbr0 state UP mode DEFAULT group default qlen 1000
    link/ether 78:55:36:0b:86:93 brd ff:ff:ff:ff:ff:ff
    altname enx7855360b8693
4: wlp89s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 50:bb:b5:db:49:80 brd ff:ff:ff:ff:ff:ff
    altname wlx50bbb5db4980
5: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 78:55:36:0b:86:93 brd ff:ff:ff:ff:ff:ff
....  

#the controller is nic0. If not initially reporting state UP, run:  
root@node:~# ip link set nic0 up
#test the properties  
root@LilNASME:~# ethtool nic0
Settings for nic0:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
                                10000baseT/Full
                                2500baseT/Full
                                5000baseT/Full
        Supported pause frame use: Symmetric Receive-only
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
                                10000baseT/Full
                                2500baseT/Full
                                5000baseT/Full
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Link partner advertised link modes:  10baseT/Full
                                             100baseT/Full
                                             1000baseT/Full
                                             10000baseT/Full
                                             2500baseT/Full
                                             5000baseT/Full
        Link partner advertised pause frame use: No
        Link partner advertised auto-negotiation: Yes
        Link partner advertised FEC modes: Not reported
        Speed: 10000Mb/s
        Duplex: Full
        Auto-negotiation: on
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        MDI-X: on
        Supports Wake-on: pumbg
        Wake-on: d
        Current message level: 0x00000033 (51)
                               drv probe ifdown ifup
        Link detected: yes
  #See "Link detected: yes"  and "Speed: 10000Mb/s". For my hardware, 10g is the expected output between the realtek controller on the mini-PC and the 10Gb PCIe network card
  #Now, add nic0 to a virtual bridge in Proxmox so the card can be used:  
   Proxmox GUI --> Node --> System --> Network  
   Create --> Linux Bridge  
   name: vmbr1  
   Bridge ports: nic0  
   click Create, then "Apply Configuration"  
  
  #do not add anything under IPv4/CIDR. This will stop VMs from using the bridge freely
  #add vmbr1 to the desired VMs as a network device (VirtIO paravirtualized) 
  # assign the associated interface in the VM as an ethernet interface with a static IP


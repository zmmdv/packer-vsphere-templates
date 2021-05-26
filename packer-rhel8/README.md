# Red Hat Enterprise Linux 8.3 Template via HashiCorp Packer #

|   [OS Version](https://access.redhat.com/)  | [Packer](https://www.packer.io/)     | [vCenter](https://www.vmware.com/products/vcenter-server.html) | Tested On|
| :------------:|:----------:|:-------:|:-------------------------:|
|   `RHEL 8.3`  | `1.6.6`    | `6.7`   | `Manjaro Cinnamon 20.2.1` |
|               |            |         | `CentOS 7 2003`           |


Before you can create your template you have change few things

1. First: Operating system kickstart file (anaconda-ks.cfg) in this repo it is ks.cfg
2. Second: you have fill variables with yours
3. You have to install one of these packages on your OS for packer so he can create kickstart.iso 

* `xorriso`
* `mkisofs`
* `hdiutil (normally found in macOS)`
* `oscdimg (normally found in Windows as part of the Windows ADK)`

4. Starting command is `packer build rhel8.json` from packer-rhel8 directory
5. NOTE! In RHEL 8 and CentOS 8 there is no more feature to mount kickstat file to floppy drive
   so you have to use local packer http server or your local http server or making iso of your
   kickstart file and uploading him to vSphere datastore and the mount him via packer. But
   Here is some reasons why i use mkisofs and mounting it from my local machine.

 * In your company network can be added network restrictions which can block your local packer http server
 * If you get new VLAN in your network you also will not get access and just waist your time
 * If you upload your kickstart file as .iso to vSphere datastore it's unsecure for you
 * So the best an fastest way is mount your ks.cfg as .iso file from local machine :+1:

# Here Small templates how it must be. #

## ks.cfg file also known as kickstart ##

file provided by [Gwyn Bleidd](https://github.com/caermeglaeddyv/) 

    # version=RHEL8
    # Reboot after installation
    # Use graphical install
    graphical

    repo --name="AppStream" --baseurl=file:///run/install/sources/mount-0000-cdrom/AppStream

    %packages
    @^minimal-environment
    kexec-tools

    %end

    # Keyboard layouts
    keyboard --vckeymap=us --xlayouts='us'
    # System language
    lang en_US.UTF-8

    # Firewall configuration
    firewall --enabled --service=ssh
    # Network information
    network  --bootproto=static --device=ens192 --gateway=192.168.1.254 --ip=192.168.1.100 --nameserver=8.8.8.8,8.8.8.4 --netmask=255.255.255.0 --noipv6 --activate
    network  --hostname=redhat8-template

    # Use CDROM installation media
    cdrom

    # System authorization information
    auth --enableshadow --passalgo=sha512

    # Run the Setup Agent on first boot
    firstboot --enable
    
    # System services
    services --enabled="chronyd"

    ignoredisk --only-use=sda
    
    # System bootloader configuration
    bootloader --location=mbr --boot-drive=sda
    
    # Partition clearing information
    clearpart --none --initlabel
    
    # Disk partitioning information
    part /boot --fstype="xfs" --ondisk=sda --size=1024
    part pv.138 --fstype="lvmpv" --ondisk=sda --size=58000
    volgroup rhel --pesize=4096 pv.138
    logvol /home --fstype="xfs" --size=4096 --name=home --vgname=rhel
    logvol /var/tmp --fstype="xfs" --size=1024 --name=var_tmp --vgname=rhel
    logvol /var --fstype="xfs" --size=12288 --name=var --vgname=rhel
    logvol /var/log --fstype="xfs" --size=4088 --name=var_log --vgname=rhel
    logvol /var/log/audit --fstype="xfs" --size=4088 --name=var_log_audit --vgname=rhel
    logvol / --fstype="xfs" --size=10220 --name=root --vgname=rhel

    # System timezone
    timezone Asia/Baku --isUtc

    # Root password (PASSWORD_!@#) encrypted in sha512 with openssl (comman openss passwd -6)
    rootpw --iscrypted $6$tb5BvUJ0ZoUfWqMk$Ywan7r/UOIGUD9jE5nEsoXR06P3Ws5.0iwwKjwb1G5dv2vQ0E8RknTDgwD0vYFesyLO4Lb8c3ERHgT2yTXyG1/

    %addon com_redhat_kdump --disable --reserve-mb='auto'
    %end

    %post
    # This is done here in the post section, after the system has completed install and before a reboot
    subscription-manager register --username USERNAME --password PASSWORD
    # Register System Purpose, and then auto attach:
    subscription-manager role --set="Red Hat Enterprise Linux Server"
    subscription-manager service-level --set="Self-Support"
    subscription-manager usage --set="Production"
    subscription-manager attach
    %end

    %anaconda
    pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
    pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
    pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
    %end

    %post --interpreter=/usr/bin/bash --log=/root/ks-post.log
    # Install perl package to customize template with Terraform in future,
    # and open-vm-tools package to use VMware Tools to interact with guest os
    # through vCenter API
    yum install -y perl open-vm-tools
    
    # Perform general package update:
    yum update -y
    
    # Start VMware Tools systemd service:
    systemctl start vmtoolsd
    %end

    reboot --eject


## After customizing your kickstart file, you have to fill variables in rhel8.json with yours ##

    "vcenter_server": "192.168.1.10",                                                    `It is your vCenter IP or FQDN`
    "vcenter_username": "administrator@vsphere.local",                                   `It is your vCenter User`
    "vcenter_user_password" : "PASSWORD_!@#",                                            `It is your vCenter User Password`
    "vcenter_datacenter_name": "My Datacenter",                                          `It is your vCenter Datacenter Name`
    "vcenter_cluster_name": "My Cluster",                                                `It is your vCenter Cluster Name`
    "vcenter_network": "VM Network",                                                     `It is your vCenter Network Name in which will be template created`
    "iso_datastore": "Datastore01",                                                      `It is your vCenter Datastore where you have your ISO`
    "iso_folder": "ISO",                                                                 `It is your folder where you uploaded your ISO`
    "iso_name": "rhel-8.3-x86_64-dvd",                                                   `It is name of your ISO file`
    "iso_cheksum": "30fd8dff2d29a384bd97886fa826fa5be872213c81e853eae3f9d9674f720ad0",   `This is sha256 iso checksum`
    "vm_cpu_count": "8",                                                                 `The number of your vm cpus`
    "vm_cpu_core_per_socket": "4",                                                       `The number of cpus in one socket if your server have 2 socket 8/2=4 if more 8/count=summ`
    "cpu_hot_plug": "true",                                                              `Hot plug feature for vm CPU`
    "vm_ram_in_mb": "4096",                                                              `RAM amount of your vm in mb`
    "vm_ram_reservation": "false",                                                       `Reservation all num of ram for your vm. Leave it false if you dont need`
    "vm_ram_hot_plug": "true",                                                           `Hot plug feature for your RAM`
    "vm_datastore_name": "Datastore01",                                                  `Datastore where vm files will stored ( vmx vmdk and etc)`
    "vm_boot_command": "<tab> inst.text inst.ks=cdrom:/dev/sr1:/ks.cfg <enter><wait>",   `Boot command dont change it`
    "vm_template_folder": "Template/RHEL",                                               `Folder where template will created`
    "vm_ssh_user": "root",                                                               `User which needs for ssh`
    "vm_ssh_password": "PASSWORD",                                                       `Password of user must be same with encrypted in kickstart file`
    "vm_template_name": "RHEL-8-TEMPLATE",                                               `The name of template what will be created`
    "kickstart_file_name": "ks.cfg",                                                     `The name of kickstart file`
    "disk_size_in_mb": "61440",                                                          `Amount of storage what will be provisioned to vm`
    "gues_os_type": "rhel8_64Guest",                                                     `Guest os type which in vCenter will be Red Hat Enterprise Linux 8(64)`
    "user_notes": "Made By Hashicorp Packer template written by Ziyaddin Mammadov",      `Special notes for template`
    "disk_label": "sr1"                                                                  `Disk label on which will be mounted kickstart`


 # Author Information #
* [Github](https://github.com/zmmdv/)
* [Instagram](https://www.instagram.com/z.mmd/)        

<p align="center"><img width="600" height="428" src="https://media.makeameme.org/created/devops.jpg"></p>

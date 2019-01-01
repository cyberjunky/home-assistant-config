
## Server Build

### Hardware used

- [Motherboard ECS H55-I](http://www.ecs.com.tw/ECSWebSite/Product/Product_Overview/EN/Motherboard/H55H-I%20-LL-V1-DO-1-RR-/Socket%201156-LL-Intel-RR-) [Download Manual](https://github.com/cyberjunky/home-assistant-config/blob/master/docs/server/ECS-H55H-IV10_manual.pdf)
- Processor Intel i5
- 16GB memory
- [ATA Samsung SSD 860 (500GB)](https://www.samsung.com/semiconductor/minisite/ssd/product/consumer/860evo/)
- [Seagate 2TB HDD](https://www.seagate.com/nl/nl/support/internal-hard-drives/desktop-hard-drives/barracuda-3-5/#specs) [Download Manual](https://github.com/cyberjunky/home-assistant-config/blob/master/docs/server/3-5-barracudaDS1900-11-1806NL-nl_NL.pdf)
- [Streacom FC8 EVO Case](https://streacom.com/products/fc8-evo-fanless-chassis/) [Download Manual](https://github.com/cyberjunky/home-assistant-config/blob/master/docs/server/Streacom-FC8-EVO.pdf)
- [PicoPSU 90 Watt](http://www.mini-box.com/picoPSU-90)

### OS

Ubuntu 18.04.1

First do a standard CLI only installation with only SSH server enabled.
Use full disk (Samsung SSD)

#### Configuration

```
$ sudo dpkg-reconfigure tzdata
$ sudo locale-gen
Current default time zone: 'Europe/Amsterdam'
Local time is now:      Mon Nov  5 20:18:05 CET 2018.
Universal Time is now:  Mon Nov  5 19:18:05 UTC 2018.
```

```
$ sudo parted 
GNU Parted 3.2
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print                                                            
Model: ATA Samsung SSD 860 (scsi)
Disk /dev/sda: 500GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  500GB   500GB   ext4
```

```
$ df -k
Filesystem     1K-blocks    Used Available Use% Mounted on
udev             3919924       0   3919924   0% /dev
tmpfs             789900    1184    788716   1% /run
/dev/sda2      479667880 6315104 448917220   2% /
tmpfs            3949492       0   3949492   0% /dev/shm
tmpfs               5120       0      5120   0% /run/lock
tmpfs            3949492       0   3949492   0% /sys/fs/cgroup
/dev/loop0         89088   89088         0 100% /snap/core/4917
tmpfs             789896       0    789896   0% /run/user/1000
```

```
$ sudo vi /etc/apt/sources.list
deb http://nl.archive.ubuntu.com/ubuntu/ bionic main restricted
deb http://nl.archive.ubuntu.com/ubuntu/ bionic-updates main restricted
deb http://nl.archive.ubuntu.com/ubuntu/ bionic universe
deb http://nl.archive.ubuntu.com/ubuntu/ bionic-updates universe
deb http://nl.archive.ubuntu.com/ubuntu/ bionic multiverse
deb http://nl.archive.ubuntu.com/ubuntu/ bionic-updates multiverse
deb http://nl.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu bionic-security main restricted
deb http://security.ubuntu.com/ubuntu bionic-security universe
deb http://security.ubuntu.com/ubuntu bionic-security multiverse
```

```
$ sudo cat <<EOF > ~/update.sh
#!/bin/bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get autoclean
sudo apt-get clean
sudo apt-get autoremove
>~/.bash_history
history -c
EOF

$ ./update.sh
```

Install lm-sensors to monitor temperatures, and ecryptfs to encrypt data on 2TB disk (NAS)

```
$ sudo apt install lm-sensors ecryptfs-utils
$ sudo sensors-detect
[Answer with default answers to find correct chip and bus]
```
```
$ sensors
acpitz-virtual-0
Adapter: Virtual device
temp1:        +30.0°C  (crit = +110.0°C)

coretemp-isa-0000
Adapter: ISA adapter
Core 0:       +38.0°C  (high = +89.0°C, crit = +105.0°C)
Core 2:       +35.0°C  (high = +89.0°C, crit = +105.0°C)
```

#### Install docker

```
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt-get install docker-ce
```

#### Install docker-compose

```
$ sudo su
# curl -L https://github.com/docker/compose/releases/download/1.23.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
# chmod +x /usr/local/bin/docker-compose
[ctrl-d]
```

```
$ sudo adduser docker
$ sudo usermod -aG sudo docker
$ su docker
$ sudo usermod -aG docker ${USER}
```

```
$ sudo cat <<EOF > /etc/environment
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"
TZ="Europe/Amsterdam"
EOF
```

*nginx-proxy-mananger*: is used as proxy server to connect hass (and other docker images I use but are outside this scope) to outside world,
it also automatically provision the used domains with lets encrypt certificates

*portainer*: for easy docker management GUI (you can also leave this out and use the portainer Hass.io addon, but I think it's better to have it outside of Hass.io)

*watchtower*: a tool which checks for image updates nightly and update them automatically.

*NOTE: I left out several images in yaml file below, these is just basic supporting software needed to prepare for Hass.io setup*

```
$ su docker
$ vi ~/docker/docker-compose.yml
---
version: '3'
services:
  nginx-proxy-manager:
    image: jlesage/nginx-proxy-manager
    container_name: nginx-proxy-manager
    ports:
      - "8181:8181"
      - "8080:8080"
      - "4443:4443"
    volumes:
      - nginx-proxy-manager:/config

  portainer:
    image: portainer/portainer
    container_name: portainer
    restart: always
    command: -H unix:///var/run/docker.sock
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    environment:
      - TZ=${TZ}

  watchtower:
    container_name: watchtower
    restart: always
    image: v2tec/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --schedule "0 0 4 * * *" --cleanup

volumes:
  nginx-proxy-manager:
  portainer-data:
----
```

Install docker images

```
$ docker-compose -f docker-compose.yml up -d
...
Creating nginx-proxy-manager   ... done
Creating portainer             ... done
Creating watchtower            ... done
```

They should be running now:

```
$ docker container list
CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS    
506a030c45          jlesage/nginx-proxy-manager              "/init"                  5 days ago          Up 5 days 
3043a97b9224        portainer/portainer                      "/portainer -H unix:…"   5 days ago          Up 5 days 
60a194ed6b10        v2tec/watchtower                         "/watchtower --sched…"   12 days ago         Up 5 days 
```

The portainer GUI is available at: http://[IP server]:9000

The nginx-proxy-manager GUI is available at: http://[IP server]:8181


### SMB Fileserver

I also use the server as a simple NAS sharing files over SMB.
It replaces my old Synology NAS with had 4 HDD's and had an Synology OS which was EOL, thus it didn't get any updates anymore.
The files are stored on the Seagate 2TB HDD mentioned above, it is put in sleep mode after an idle time to save power and keep temperatures down.
Backups are made to an external disk.

Here are some snippets describing the SMB server config.

#### Preparing the 2TB disk

```
$ sudo parted -l
$ sudo parted /dev/sdx mklabel gpt
$ sudo parted -a opt /dev/sdx mkpart primary ext4 0% 100%
$ sudo mkfs.ext4 -L shares /dev/sdx1

$ sudo mkdir /mnt/shares
$ sudo mount /dev/sdx1 /mnt/shares
```

Now we enrypt the contents

```
$ sudo mount -t ecryptfs /mnt/shares /mnt/shares
Passphrase: 
Select cipher: 
 1) aes: blocksize = 16; min keysize = 16; max keysize = 32
 2) blowfish: blocksize = 8; min keysize = 16; max keysize = 56
 3) des3_ede: blocksize = 8; min keysize = 24; max keysize = 24
 4) twofish: blocksize = 16; min keysize = 16; max keysize = 32
 5) cast6: blocksize = 16; min keysize = 16; max keysize = 32
 6) cast5: blocksize = 8; min keysize = 5; max keysize = 16
Selection [aes]: 
Select key bytes: 
 1) 16
 2) 32
 3) 24
Selection [16]: 
Enable plaintext passthrough (y/n) [n]: 
Enable filename encryption (y/n) [n]: 
Attempting to mount with the following options:
  ecryptfs_unlink_sigs
  ecryptfs_key_bytes=16
  ecryptfs_cipher=aes
  ecryptfs_sig=f735f5c2e9a1a147
WARNING: Based on the contents of [/root/.ecryptfs/sig-cache.txt],
it looks like you have never mounted with this key 
before. This could mean that you have typed your 
passphrase wrong.

Would you like to proceed with the mount (yes/no)? : yes
Would you like to append sig [xxxxxxxxxxxxx] to
[/root/.ecryptfs/sig-cache.txt] 
in order to avoid this warning in the future (yes/no)? : yes
Successfully appended new sig to user sig cache file
Mounted eCryptfs
```

After a reboot the disk contents is encrypted, I choose to decrypt it manually, and have this script as a note.

```
$ cat <<EOF > ~/mount.sh <<EOF
#!/bin/bash
echo "mount -t ecryptfs /mnt/shares /mnt/shares"
EOF
$ chmod +x mount.sh
```

Install and configure hdparm to put disk in sleep mode after idle timeout.

```
$ sudo install hdparm
```

Find out disk's UUID

```
$ sudo blkid
```

Add below snippet to end of file.

```
$ vi /etc/hdparm.conf
command_line {
    hdparm -S 25 /dev/disk/by-uuid/<your disk uuid>
}

$ sudo hdparm -S 25 /dev/disk/by-uuid/<your disk uuid>
/dev/disk/by-uuid/<your disk uuid>:
 setting standby to 25 (2 minutes + 5 seconds)

$ sudo hddtemp  /dev/sdb
/dev/sdb: ST2000DM006-2DM164: drive is sleeping
```

#### Install SAMBA

```
$ sudo apt-get install samba
$ sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
$ sudo vi /etc/samba/smb.conf
---
[global]
        netbios name = SERVER
        workgroup = HERSCHEL
        server role = standalone server
        name resolve order = bcast host
        domain master = no
        case sensitive = no
        auto services = global
        interfaces = lo enp2s0
        bind interfaces only = yes
        disable netbios = yes
        smb ports = 139 445
        log file = /var/log/samba/smb.log
        max log size = 1000
        passdb backend = tdbsam
        obey pam restrictions = yes
        unix password sync = no
        pam password change = yes
        load printers = no
        force group = sambashare
        min protocol = SMB2

[documents]
        path = /mnt/shares/documents
        browseable = yes
        read only = no
        delete readonly = yes
        directory mask = 0777
        valid users = @sambashare

[audio]
        path = /mnt/shares/audio
        browseable = yes
        read only = no
        valid users = @sambashare

[photo]
        path = /mnt/shares/photo
        browseable = yes
        read only = no
        valid users = @sambashare

[video]
        path = /mnt/shares/video
        browseable = yes
        read only = no
        valid users = @sambashare

[backup]
        path = /mnt/shares/backup
        browseable = no
        read only = no
        valid users = @sambashare
---
```

```
$ sudo smbpasswd -a ron
Storing account ron with RID 1000
Enabled user ron.

$ sudo cd /mnt/shares/
$ sudo mkdir audio backup documents photo video

$ sudo usermod -aG sambashare ron
$ sudo chgrp -R sambashare /mnt/shares
$ sudo chmod 2770 /mnt/shares
$ sudo systemctl restart smbd.service
```

This ends the basic setup, checkout the Hass.io setup [here](https://github.com/cyberjunky/home-assistant-config/blob/master/README.md#home-assistant).

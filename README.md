# abap-1909-undocked

Three months ago SAP has released new Developer Edition [SAP ABAP Platform 1909](https://blogs.sap.com/2021/02/15/sap-abap-platform-1909-developer-edition-available-soon/comment-page-1/). The euphoria is great: new features, has HANA as DB and on top of all runs in Docker. Super easy to install on Linux, Windows and macOS, even for newbies. Yeah!

Looking at the requirements the euphoria begins to disappear. 16GB RAM and 150GB Disk for Docker alone. On Windows and macOS Docker runs in a VM and the overhead is much bigger than on Linux. I didn't measured it precisely, but have the feeling it is not much smaller than running VMware or VirtualBox, if at all. Additional to that if you want to change some properties of the container after you run it, you have to *docker rm* it and run it with the new parameters again, which basically means, you start from the very beginning. Docker on Windows comes with other restrictions e.g. you can't just *bridge* the network.

Being unhappy with Docker on WSL2 and running it in Docker in VMware makes little sense to me so I decided to undock it. With success. Below the steps I did.


1. Use [docker save](https://docs.docker.com/engine/reference/commandline/save/) to save the image outside Docker. The file is 60GB
```
G:\ABAP1909>docker save -o abaptrial_1909 store/saplabs/abaptrial:1909
```

2. Install [Oracle Linux 8.3](https://yum.oracle.com/oracle-linux-isos.html). I used the Full ISO. 

*	a. Make a **minimal** Software Selection. 
*	b. Set hostname to **vhcala4hci**
*	c. Make sure you have enough disk space in **/**. If you intend to copy the file exported in step 1 you will need ~190GB. I used the *Shared Folders* feature in VMware so I needed 130GB for Linux. It is even more than the required by SAP, but this is temporary. At the end the space used will be below 60GB. 
*	d. **DO NOT Create addition user**

3. After Linux starts update it and install some addition packages. *uuidd tcsh libnsl libaio libatomic* are  required by SAP. *python3* is for the undocking. 
```
yum update -y
yum install -y uuidd tcsh libnsl libaio libatomic python3 nano net-tools open-vm-tools
```

4. Start uuid, disable firewall and SElinux
```
systemctl start uuidd && systemctl enable uuidd
systemctl stop firewalld && systemctl disable firewalld
sed -i s/^SELINUX=.*$/SELINUX=disabled/ /etc/selinux/config
```

5. Make entry in hosts. Replace **10.10.10.110** with your IP.
```
echo "10.10.10.110 vhcala4hci" >> /etc/hosts
```

6. Reboot Linux.

7. I used *Shared Folders* so mounted them
```
/usr/bin/vmhgfs-fuse .host:/ /mnt/hgfs -o subtype=vmhgfs-fuse,allow_other,umask=022
```

8. After mounting the file from step 1 in my case it is here
```
/mnt/hgfs/trial/abaptrial_1909
```

9. Create working directory and cd to it
```
mkdir /abaptrial && cd /abaptrial
```

10. Get [**undocker.py**](https://github.com/larsks/undocker) and put it in */abaptrial*

11. The examples of *undocker.py* are in form *docker save busybox | undocker -o busybox -v*. Because of this the *undocker.py* expects the container name to be *store/saplabs/abaptrial:1909*. If we supply abaptrial_1909 the script will fail. We can edit the script to suit our needs but I used this workaround.
```
mkdir -p store/saplabs
ln -s /mnt/hgfs/trial/abaptrial_1909 store/saplabs/abaptrial:1909
```

12. Still in **/abaptrial** execute the following. It will take some time. Take a coffee (or two). The script first copies the content of the Docker Image to **/tmp** and then to the current folder, therefor the extra space.
```
python3 undocker.py store/saplabs/abaptrial:1909
```

13. Remove old HANA executables and traces
```
rm -rf hana/shared/HDB/exe/linuxx86_64/hdb_old
rm -rf hana/shared/HDB/exe/linuxx86_64/HDB_2.00.012.02.1506602370_f3eed173a5f0431baee8be0c50f3a88b67840452

rm -f hana/shared/HDB/HDB02/vhcala4hci/trace/*
rm -f hana/shared/HDB/HDB02/vhcala4hci/trace/DB_HDB/*
```

14. Move the SAP HANA and ABAP AS installation folders
```
mv home/* /home
mv hana /.
mv opt/* /opt
mv sapmnt /.
mv usr/sap /usr/.
mv var/lib/hdb /var/lib/.
mv etc/init.d/sapinit /etc/rc.d/init.d/.
mv usr/local/bin/* /usr/local/bin/.
```

15. Copy the SAP entries for **/etc/services**
```
grep sapmsA4H etc/services >> /etc/services
grep "SAP System" etc/services >> /etc/services
```

**In the following 2 steps the group and user IDs are important as these have to mach with the permisions of the undocked files.**

16. Create groups
```
groupadd -g 493 sapsys
groupadd -g 1000 hdbshm
groupadd -g 1001 sapinst
groupadd -g 492 sccgroup
```

17. Create users (ignore - useradd: warning: the home directory already exists.)
```
useradd -m -d /home/a4hadm -g 493 -c "SAP System Administrator" -s /bin/csh -u 1001 a4hadm
useradd -m -d /opt/sap/scc -g 492 -c "SAP scc admin" -s /bin/false -u 494 sccadmin
useradd -m -d /home/sapadm -g 493 -c "SAP Local Administrator" -s /bin/false -u 495 sapadm
useradd -m -d /usr/sap/HDB/home -g 493 -c "SAP HANA Database System Administrator" -s /bin/sh -u 1000 hdbadm
```

18. Set limits and kernel parameters
```
echo "*    hard    nofile    1048576" >> /etc/security/limits.conf
echo "*    soft    nofile    1048576" >> /etc/security/limits.conf
echo "vm.max_map_count=2147483647" >> /etc/sysctl.conf
echo "fs.file-max=20000000" >> /etc/sysctl.conf
echo "fs.aio-max-nr=18446744073709551615" >> /etc/sysctl.conf
echo "kernel.shmmni=32768" >> /etc/sysctl.conf

sysctl -p
```

19. Reboot

20. Start and enable **sapinit**
```
service sapinit start
systemctl enable sapinit
```

21. With fingers crossed start HANA DB.
```
su - hdbadm -c "HDB start"
```

22. After HANA starts switch to user a4hadm and check the Hardware Key
```
su - a4hadm
saplikey pf=/sapmnt/A4H/profile/A4H_D00_vhcala4hci -get
```
Should be something like this:
```
vhcala4hci:a4hadm 51> saplikey pf=/sapmnt/A4H/profile/A4H_D00_vhcala4hci -get
SAP License Key Administration  -  Copyright (C) 2003 - 2016 SAP AG

System ID. . . . : A4H
Hardware Key . . : Z0131444143        (of this computer)
Installation No. : DEMOSYSTEM
System No. . . . : 000000000850577307
Release. . . . . : 777
Software products: NetWeaver_HDB
```

23. [Get new license](https://go.support.sap.com/minisap/#/minisap). Put it in the a4hadm home folder and execute
```
saplikey pf=/sapmnt/A4H/profile/A4H_D00_vhcala4hci -install A4H_Multiple.txt
```
Or do it as it was before through the GUI with user SAP*  after starting SAP AS (step 25).

24. Check the license
```
vhcala4hci:a4hadm 52> saplikey pf=/sapmnt/A4H/profile/A4H_D00_vhcala4hci -show
SAP License Key Administration  -  Copyright (C) 2003 - 2016 SAP AG

List of installed License Keys:
==========================================

1. License Key:
------------------------------------------
System                : A4H
Hardware Key          : Z0131444143
Software Product      : NetWeaver_HDB
Software Product Limit: 2147483647
Type of License Key   : permanent
Installation Number   : DEMOSYSTEM
System Number         : 000000000850577307
Begin of Validity     : 20210507
End   of Validity     : 20210808
Last successful check : 20210513
Validity              : valid

2. License Key:
------------------------------------------
System                : A4H
Hardware Key          : Z0131444143
Software Product      : Maintenance_HDB
Software Product Limit: 2147483647
Type of License Key   : permanent
Installation Number   : DEMOSYSTEM
System Number         : 000000000850577307
Begin of Validity     : 20210507
End   of Validity     : 20210808
Last successful check : 00000000
Validity              : valid

------------------------------------------
2 license keys listed.
```

25. Still as **a4hadm** start SAP Application Server
```
sapcontrol -nr 0 -function StartSystem
```
or as **root**
```
su - a4hadm -c "sapcontrol -nr 0 -function StartSystem"
```

26. For stopping the system I'm using the following script.
```
[root@vhcala4hci ~]# cat stopsap
#!/bin/bash
su - a4hadm -c "sapcontrol -nr 0 -function StopSystem"
n=36
echo "Waiting for SAP to stop"
while (( $n > 10 ))
do
        n="$(ps -ef | grep a4h | wc -l)"
        sleep 5
done
echo "SAP stopped"
su - hdbadm -c "HDB stop"
```

27. Now if you are on VMware shrink the space the VM uses. This will take a lot of time, too.
```
vmware-toolbox-cmd disk shrink /
```
After the shrink the VMware occupies only 55GB disk space. 

Enjoy.


Additional tests. 

Tried with 12GB RAM. HANA doesn't start. 

With 13GB HANA and AS do start, but in GUI you get only short dumps.

```
Category               Resource bottleneck
Runtime Errors         DBIF_REPO_SQL_ERROR
Date and Time          08.05.2021 20:59:42 (UTC)

SQL error 129 while accessing program "LAGS_TBOM_CONTENT_GAP5_3$01" part "SRC
".

Database error text: "transaction rolled back by an internal error:
 AttributeEngine: error reading file"
```

With 14GB it seems to work. 



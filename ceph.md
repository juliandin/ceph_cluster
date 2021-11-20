
Console access: 
| Hostname | User | PW |
| ------ | ------ | ------ |
| server1 | ubuntu | Server1
| server2 | ubuntu | Server2
| server3 | ubuntu | Server3

### Roles/model: 

| Hostname | Role | NIC1 |
| ------ | ------ | ------ |
| server1 | Monitor, Management, OSD | 192.168.161.63
| server2 | OSD |192.168.161.58
| server3 | OSD |192.168.161.59

---
## Disks configuration
[![x](https://i.imgur.com/y7xQhRA.png)]()
---

### ~/.ssh/config 
```sh
Host server1 
    Hostname server1 
    User ubuntu 

Host server2 
    Hostname server2 
    User ubuntu 

Host server3 
    Hostname server3 
    User ubuntu 
```
 
```sh
chmod 600 ~/.ssh/config 
```

### /etc/hosts 

```sh
192.168.161.63 server1 
192.168.161.58 server2 
192.168.161.59 server3 
192.168.180.14 client 
```

### On all three machines: 
```sh
lvcreate -L 200G -n shared-space ubuntu-vg 

lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT 
loop0                       7:0    0  55.4M  1 loop /snap/core18/1944 
loop1                       7:1    0  31.1M  1 loop /snap/snapd/10707 
loop2                       7:2    0  69.9M  1 loop /snap/lxd/19188 
sda                         8:0    0 410.1G  0 disk 
├─sda1                      8:1    0     1M  0 part 
├─sda2                      8:2    0     1G  0 part /boot 
└─sda3                      8:3    0 409.1G  0 part 
  ├─ubuntu--vg-ubuntu--lv 253:0    0   200G  0 lvm  / 
  └─ubuntu--vg-shared--lv 253:1    0   200G  0 lvm 
 ```
 
```sh
ssh-keygen –t rsa –b 4096 
ssh-copy-id server2 
ssh-copy-id server3 
```
And vice-versa keys if needed 

### Add NOPASSWD line in /etc/sudoers so that admin group members can use sudo without password (this can be changed later, but for scripts it is useful)
```sh
%admin ALL=(ALL) NOPASSWD:ALL
```

-------------------- 
# Mgmt server (Server1)

Install all needed packages. It will ask for serverX ubuntu users password.

```sh
for NODE in server1 server2 server3 
do 
    ssh -t $NODE "sudo apt update;sudo apt -y install ceph" 
done 
```


```sh
uuidgen 
```

### /etc/ceph/ceph.conf 

```sh
[global] 
# Public network 
public network = 192.168.161.0/24 

# UUID generated above 
fsid = 77f0cde8-ef40-4a05-a82d-22318329d598 

# IP address of Monitor Daemon 
mon host = 192.168.161.63  # server1

# Hostname of Monitor Daemon 
mon initial members = server1 

osd_pool_default_size = 3 
osd_pool_default_min_size = 2 

[mon.node01] 
# specify Hostname of Monitor Daemon 
host = server1 
# specify IP address of Monitor Daemon 
mon addr = 192.168.161.63  # server1 
```

---

### Generate secret key for Cluster monitoring 
```sh
ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *' 
```
  
### Generate secret key for Cluster admin 
```sh
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *' 
```
  
### Generate key for bootstrap 
```sh
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r' 
```
  
### Import generated key 
```sh
ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring 
ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring 
```

### Generate monitor map 
```sh
FSID=$(grep "^fsid" /etc/ceph/ceph.conf | awk {'print $NF'}) 
NODENAME=$(grep "^mon initial" /etc/ceph/ceph.conf | awk {'print $NF'}) 
NODEIP=$(grep "^mon host" /etc/ceph/ceph.conf | awk {'print $NF'}) 
monmaptool --create --add $NODENAME $NODEIP --fsid $FSID /etc/ceph/monmap 
```
  
### Create a directory for Monitor Daemon 
```sh
mkdir /var/lib/ceph/mon/ceph-server1 
```

### Assosiate key and monmap to Monitor Daemon 
```sh
ceph-mon --cluster ceph --mkfs -i $NODENAME --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring 
chown ceph. /etc/ceph/ceph.* 
chown -R ceph. /var/lib/ceph/mon/ceph-server1 /var/lib/ceph/bootstrap-osd 
systemctl enable --now ceph-mon@$NODENAME 
```
  
### Enable Messenger v2 Protocol (should be enabled by default) and enable Placement Groups auto scale module (should be enabled by default) 
```sh
ceph mgr module enable pg_autoscaler 
ceph mon enable-msgr2 
```
  
### Create a directory for Manager Daemon 
```sh
mkdir /var/lib/ceph/mgr/ceph-server1 
``` 
  
### Create auth key 
```sh
ceph auth get-or-create mgr.$NODENAME mon 'allow profile mgr' osd 'allow *' mds 'allow *' 
ceph auth get-or-create mgr.server1 | tee /etc/ceph/ceph.mgr.admin.keyring 

cp /etc/ceph/ceph.mgr.admin.keyring /var/lib/ceph/mgr/ceph-server1/keyring 

chown ceph. /etc/ceph/ceph.mgr.admin.keyring 
chown -R ceph. /var/lib/ceph/mgr/ceph-server1 

systemctl enable --now ceph-mgr@$NODENAME 
```

### Check monitor
```sh
root@server1:~# ceph -s
  cluster:
    id:     51c50f43-8460-45eb-8c76-182ccf9c324e
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum server1 (age 23s)
    mgr: server1(active, starting, since 0.444756s)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
 ```

---

## OSDs creation: 
Create somewhere script. /etc/ceph/ for example.

```sh
#!/bin/bash 
for NODE in server1 server2 server3 
do 
    if [ ! ${NODE} = "server1" ] 
    then 
        scp /etc/ceph/ceph.conf ${NODE}:/home/ubuntu/ceph.conf 
        ssh -t ubuntu@${NODE} sudo mv /home/ubuntu/ceph.conf /etc/ceph/ 

        scp /etc/ceph/ceph.client.admin.keyring ${NODE}:/home/ubuntu/ceph.client.admin.keyring 
        ssh -t ubuntu@${NODE} sudo mv /home/ubuntu/ceph.client.admin.keyring /etc/ceph/ 

        scp /var/lib/ceph/bootstrap-osd/ceph.keyring ${NODE}:/home/ubuntu/ceph.keyring 
        ssh -t ubuntu@${NODE} sudo mv /home/ubuntu/ceph.keyring /var/lib/ceph/bootstrap-osd/ 
    ssh -t ubuntu@$NODE \ 
    "sudo chown ceph. /etc/ceph/ceph.*; \ 
    sudo chown ceph. /var/lib/ceph/bootstrap-osd/ceph.keyring; \
    sudo ceph-volume lvm create --data ubuntu-vg/shared-space" 
    fi 
done 
```

```sh
chmod +x osd_create.sh
```
And run it.

### Check OSD
```sh
root@server1:/etc/ceph# ceph -s
  cluster:
    id:     51c50f43-8460-45eb-8c76-182ccf9c324e
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum server1 (age 40m)
    mgr: server1(active, since 40m)
    osd: 3 osds: 3 up (since 2m), 3 in (since 2m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 597 GiB / 600 GiB avail
    pgs:     1 active+clean


root@server1:/etc/ceph# ceph osd df
ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP  META   AVAIL    %USE  VAR   PGS  STATUS
 2    hdd  0.19530   1.00000  200 GiB  1.0 GiB  2.5 MiB   0 B  1 GiB  199 GiB  0.50  1.00    1      up
 1    hdd  0.19530   1.00000  200 GiB  1.0 GiB  2.5 MiB   0 B  1 GiB  199 GiB  0.50  1.00    1      up
 0    hdd  0.19530   1.00000  200 GiB  1.0 GiB  2.5 MiB   0 B  1 GiB  199 GiB  0.50  1.00    1      up
                       TOTAL  600 GiB  3.0 GiB  7.5 MiB   0 B  3 GiB  597 GiB  0.50
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```
Looks good.
 
-------------------- 

# Client 

### Install ceph on client machine
```sh
sudo ssh -t student@client "sudo apt update; sudo apt -y install ceph-common"
```
### Transfer required files to Client Host from server1 (You can not SCP it straight into ceph folder - no permissions, need sudo.)
```sh
sudo scp /etc/ceph/ceph.conf student@client:/home/student
ssh student@client sudo mv /home/student/ceph.conf /etc/ceph/ceph.conf
sudo scp /etc/ceph/ceph.client.admin.keyring student@client:/home/student
ssh student@client sudo mv /home/student/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
ssh student@client "sudo chown ceph. /etc/ceph/ceph.*" 
```

### Create default RBD pool [rbd] 

>A RADOS Block Device (RBD) is software that facilitates the storage of block-based data in the open source Ceph distributed storage system. Ceph RBD facilitates the storage of disk images, snapshots and storage volumes.] 
128 - Placement groups (PGs) are an internal implementation detail of how Ceph distributes data. You can allow the cluster to either make recommendations or automatically tune PGs based on how the cluster is used by enabling pg-autoscaling - Done.

```sh
sudo su
```


```sh
ceph osd pool create rbd 128 
```

### Enable Placement Groups auto scale mode 
```sh
ceph osd pool set rbd pg_autoscale_mode on 
```
 
### Initialize the pool 
```sh
rbd pool init rbd 
```
  
```sh
root@grupp3vm:/etc/ceph# ceph osd pool autoscale-status
POOL                     SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE
device_health_metrics      0                 3.0        600.0G  0.0000                                  1.0       1              on
rbd                        0                 3.0        600.0G  0.0000                                  1.0     128          32  on
```

### Create a block device with 600G 
```sh
rbd create --size 600G --pool rbd rbd01 

root@grupp3vm:/etc/ceph# rbd ls -l
NAME   SIZE     PARENT  FMT  PROT  LOCK
rbd01  600 GiB            2
```
  
### Map the block device 
```sh
rbd map rbd01 
```

### Confirm that everything looks fine 
```sh
root@grupp3vm:/etc/ceph# rbd showmapped
id  pool  namespace  image  snap  device
0   rbd              rbd01  -     /dev/rbd0
```

### Format with XFS 
```sh
mkfs.xfs /dev/rbd0 
```
  
### Finally mount  
```sh
mount /dev/rbd0 /mnt 
```

### Check our precious work
```sh
root@grupp3vm:/home/student# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           395M  1.1M  394M   1% /run
/dev/sda2        16G  6.4G  8.6G  43% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       89M   89M     0 100% /snap/core/7270
/dev/loop1      100M  100M     0 100% /snap/core/10958
tmpfs           395M     0  395M   0% /run/user/1000
/dev/rbd0       600G  4.3G  596G   1% /mnt

cd /mnt
```
Downloaded some useless files (wget http://ipv4.download.thinkbroadband.com/1GB.zip)
And copied it a few times.
```sh
root@grupp3vm:/mnt# ll
total 10475772
drwxr-xr-x  2 root root        165 May  1 18:15 ./
drwxr-xr-x 20 root root       4096 May  1 15:11 ../
-rw-r--r--  1 root root 1073741824 May 30  2008 1GB.zip
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip1
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip2
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip3
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip4
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip5
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip6
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip7
-rw-r--r--  1 root root 1073741824 May  1 18:15 1GB.zip8
-rw-r--r--  1 root root 1063510016 May  1 18:15 1GB.zip9

root@grupp3vm:/home/student# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           395M  1.1M  394M   1% /run
/dev/sda2        16G  6.4G  8.6G  43% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0       89M   89M     0 100% /snap/core/7270
/dev/loop1      100M  100M     0 100% /snap/core/10958
tmpfs           395M     0  395M   0% /run/user/1000
/dev/rbd0       600G   15G  586G   3% /mnt
```

We can even see some actions being done on the client side.
```sh
root@server1:/home/ubuntu# ceph -s
  cluster:
    id:     51c50f43-8460-45eb-8c76-182ccf9c324e
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum server1 (age 10m)
    mgr: server1(active, since 86m)
    osd: 3 osds: 3 up (since 48m), 3 in (since 48m)

  data:
    pools:   2 pools, 33 pgs
    objects: 1.28k objects, 4.9 GiB
    usage:   18 GiB used, 582 GiB / 600 GiB avail
    pgs:     33 active+clean

  io:
    client:   139 KiB/s rd, 45 MiB/s wr, 103 op/s rd, 41 op/s wr
```

### Add line into /etc/fstab on client machine
```sh
/def/rbd0 /mnt xfs defaults 0 0 
```

---

##### To stop/start ceph-volume/osd: 
```sh
systemctl stop ceph-osd@0 
ceph-volume lvm zap --osd-id 0 
```
##### List cluster pools:  
```sh
ceph osd lspools 
```

##### To delete Block devices or Pools in our case: 
```sh
rbd unmap /dev/rbd/rbd/rbd01
rbd rm rbd01 -p rbd
ceph osd pool delete rbd rbd --yes-i-really-really-mean-it
(and in case it still does not want to be deleted add following line below and rerun)
ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
```

### Used sources:
1. https://docs.ceph.com/
2. https://alanxelsys.com/ceph-hands-on-guide/ 
3. https://www.howtoforge.com/tutorial/how-to-install-a-ceph-cluster-on-ubuntu-16-04/ 
4. https://www.server-world.info/en/note?os=Ubuntu_20.04&p=ceph15&f=1 
5. https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/administration_guide/the-ceph-volume-utility 
6. And a lot of useless YouTube videos.

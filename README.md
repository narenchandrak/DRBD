# DRBD + Pacemaker & Corosync MySQL Cluster Centos7

![DRBD With Pacemaker](Resources\1.png)





# DRBD

### LVM partition setup

**Both Nodes**

Create a empty partition

```bash
fdisk /dev/vdb
```

<img src="Resources\DRBD\1.png" style="zoom:150%;" />

Create LVM partition

```bash
pvcreate /dev/vdb1
vgcreate vg00 /dev/vdb1
lvcreate -l 95%FREE -n drbd-r0 vg00
```

<img src="Resources\DRBD\2.png" style="zoom: 200%;" />

View LVM partition after creation

```bash
pvdisplay
vgdisplay
lvdisplay
```

Look in "/dev/mapper/" find the name of your LVM disk

```bash
ls /dev/mapper/
```

OUTPUT:

```
control vg00-drbd--r0
```

**You will use "vg00-drbd--r0" in the "mysql.res" file in the below steps

### DRBD Installation

Install the DRBD package

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install -y https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
yum info *drbd* | grep Name
yum install -y drbd90-utils drbd90-utils-sysvinit kmod-drbd90
modprobe drbd
echo drbd > /etc/modules-load.d/drbd.conf
semanage permissive -a drbd_t
```

Edit the DRBD config and add the to hosts it will be connecting to (DB1 and DB2)

```bash
cat /etc/drbd.conf
```

<img src="Resources\DRBD\3.png" style="zoom:150%;" />

```bash
vim /etc/drbd.d/global_common.conf
```

```
# DRBD is the result of over a decade of development by LINBIT.
# In case you need professional services for DRBD or have
# feature requests visit http://www.linbit.com

global {
        usage-count no;

        # Decide what kind of udev symlinks you want for "implicit" volumes
        # (those without explicit volume <vnr> {} block, implied vnr=0):
        # /dev/drbd/by-resource/<resource>/<vnr>   (explicit volumes)
        # /dev/drbd/by-resource/<resource>         (default for implict)
        udev-always-use-vnr; # treat implicit the same as explicit volumes

        # minor-count dialog-refresh disable-ip-verification
        # cmd-timeout-short 5; cmd-timeout-medium 121; cmd-timeout-long 600;
}

common {
        handlers {
                # These are EXAMPLE handlers only.
                # They may have severe implications,
                # like hard resetting the node under certain circumstances.
                # Be careful when choosing your poison.

                # pri-on-incon-degr "/usr/lib/drbd/notify-pri-on-incon-degr.sh; /usr/lib/drbd/notify-emergency-reboot.sh; echo b > /proc/sysrq-trigger ; reboot -f";
                # pri-lost-after-sb "/usr/lib/drbd/notify-pri-lost-after-sb.sh; /usr/lib/drbd/notify-emergency-reboot.sh; echo b > /proc/sysrq-trigger ; reboot -f";
                # local-io-error "/usr/lib/drbd/notify-io-error.sh; /usr/lib/drbd/notify-emergency-shutdown.sh; echo o > /proc/sysrq-trigger ; halt -f";
                # fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
                split-brain "/usr/lib/drbd/notify-split-brain.sh root";
                out-of-sync "/usr/lib/drbd/notify-out-of-sync.sh root";
                # before-resync-target "/usr/lib/drbd/snapshot-resync-target-lvm.sh -p 15 -- -c 16k";
                # after-resync-target /usr/lib/drbd/unsnapshot-resync-target-lvm.sh;
                # quorum-lost "/usr/lib/drbd/notify-quorum-lost.sh root";
        }

        startup {
                # wfc-timeout degr-wfc-timeout outdated-wfc-timeout wait-after-sb
                degr-wfc-timeout 60;
                outdated-wfc-timeout 30;
                wfc-timeout 20;
        }

        options {
                # cpu-mask on-no-data-accessible

                # RECOMMENDED for three or more storage nodes with DRBD 9:
                # quorum majority;
                # on-no-quorum suspend-io | io-error;
                auto-promote yes;
        }

        disk {
                # size on-io-error fencing disk-barrier disk-flushes
                # disk-drain md-flushes resync-rate resync-after al-extents
                # c-plan-ahead c-delay-target c-fill-target c-max-rate
                # c-min-rate disk-timeout
                on-io-error detach;
        }

        net {
                # protocol timeout max-epoch-size max-buffers
                # connect-int ping-int sndbuf-size rcvbuf-size ko-count
                # allow-two-primaries cram-hmac-alg shared-secret after-sb-0pri
                # after-sb-1pri after-sb-2pri always-asbp rr-conflict
                # ping-timeout data-integrity-alg tcp-cork on-congestion
                # congestion-fill congestion-extents csums-alg verify-alg
                # use-rle
                protocol C;
                allow-two-primaries no;
                cram-hmac-alg sha1;
                shared-secret "Redhat@123";
                after-sb-0pri discard-zero-changes;
                after-sb-1pri discard-secondary;
                after-sb-2pri disconnect;
                rr-conflict disconnect;
        }
}
```



```bash
vim /etc/drbd.d/r0.res
```

```
resource r0 {

device /dev/drbd0;
disk /dev/mapper/vg00-drbd--r0;

on db1.example.com {
address     192.168.122.96:7788;
meta-disk   internal;
node-id     0;
}

on db2.example.com {
address     192.168.122.133:7788;
meta-disk   internal;
node-id     1;
}
}
```

**Both Nodes**

Cluster nodes to initialize the r0 resource

```bash
drbdadm create-md r0
drbdadm up r0
systemctl start drbd
lsblk 
```

**On DB1**

**While** we start `DRBD` service, it runs in secondary mode by default. Execute the following command to change the mode to primary on one of the cluster node which Initializes Device Synchronization.

```bash
drbdadm primary r0 --force
```

Execute the following command to view real time device Synchronization status.

```bash
watch -n .2 drbdsetup status -vs	
```

**Both Nodes**

```bash
drbdadm status
```

<img src="Resources\DRBD\4.png" style="zoom:150%;" />

**On DB1**

Once the Synchronization is completed, create `filesystem` on primary node and mount the `DRBD` volume.

```bash
mkfs.xfs /dev/drbd0
mount -t xfs /dev/drbd0 /mnt/
df -h
```

Put some data on mounted `DRBD` partition and check the data on other node.

```bash
cp -r /etc/ /mnt/
ls /mnt/
```

<img src="Resources\DRBD\5.png" style="zoom:150%;" />

Change `db1` to secondary mode and `db2` to primary mode, mount the `DRBD` volume and check whether data is available.

**On DB1**

```bash
umount /mnt/
drbdadm secondary r0
drbdadm status
```

<img src="Resources\DRBD\6.png" style="zoom:200%;" />

**On DB2**

```bash
drbdadm primary r0
drbdadm status
mount /dev/drbd0 /mnt/
ls /mnt/
```

**Note:** As we see on the above test that, the data is available on both cluster nodes.

<img src="Resources\DRBD\7.png" style="zoom:200%;" />

Now stop and disable the `DRBD` service if started because this service will be managed through cluster.

**On DB1**

```bash
systemctl stop drbd
systemctl disable drbd
```

**On DB2**

```bash
umount /mnt/
drbdadm secondary r0
systemctl stop drbd
```





# MariaDB

**Both Nodes**

Install the following package on cluster nodes to setup `MariaDB` server.

```bash
yum install -y mariadb-server mariadb
```

Start `DRBD` service on cluster nodes, make one of the cluster node to primary and then mount the `DRBD` volume on `/var/lib/mysql` directory.

```
systemctl start drbd
drbdadm up r0
```

**On DB1**

```bash
drbdadm primary r0
drbdadm status
mount -t xfs /dev/drbd0 /var/lib/mysql/
chown -R mysql:mysql /var/lib/mysql/
```

Start `MariaDB` service and execute the following command on `DRBD` volume mounted node to configure `mysql` secure installation.

**On DB1**

```bash
systemctl start mariadb.service
ls /var/lib/mysql
mysql_secure_installation
```

<img src="Resources\DRBD\8.png" style="zoom: 200%;" />

Test mysql database on cluster nodes

```bash
mysql -u root -p
	show databases;
	create database test1;
	show databases;
	exit
```

<img src="Resources\DRBD\9.png" style="zoom:150%;" />

Stop `mariadb` service on `db1` and unmount `/var/lib/mysql` mount point directory. Now change the `DRBD` resource mode from primary to secondary on `db1` and make `db2` to primary then mount the `DRBD` volume and start `mariadb` service to test the database.

```bash
systemctl stop mariadb.service
umount /var/lib/mysql/
drbdadm secondary r0
drbdadm status
```

**On DB2**

```bash
drbdadm primary r0
drbdadm status
mount -t xfs /dev/drbd0 /var/lib/mysql/
ls /var/lib/mysql/
systemctl start mariadb.service
mysql -u root -p
	show databases;
	create database test2;
	show databases;
	exit
```

**Note:** As we see on the above test that, we are able to create and view database on both cluster nodes.

<img src="Resources\DRBD\11.png" style="zoom:150%;" />

<img src="C:\Users\Naren Chandra\Desktop\DRBD\Resources\DRBD\12.png" style="zoom:150%;" />

Now stop both `DRBD` and `mariadb` service on cluster nodes and start `pacemaker cluster` service.

```bash
systemctl stop mariadb.service
umount /var/lib/mysql/
drbdadm secondary r0
drbdadm status
systemctl stop drbd
```

**On DB1**

```bash
drbdadm primary r0
systemctl stop drbd
```



# Pacemaker

### Pacemaker Install

**ON BOTH NODES**

Install PaceMaker and Corosync

```bash
yum install -y pcs pacemaker corosync resource-agents fence-agents-all psmisc policycoreutils-python bash-completion vim
```

Authenticate as the hacluster user

```bash
echo "Redhat@123" | passwd hacluster --stdin
```

Start and enable the service

```bash
systemctl start pcsd
systemctl enable pcsd
```

**ON DB1**

Test and generate the Corosync configuration

```bash
pcs cluster auth db1 db2 -u hacluster -p Redhat@123
pcs cluster setup --start --name mycluster db1 db2
```

**ON BOTH NODES**

Start the cluster

```bash
systemctl start corosync
systemctl enable corosync
pcs cluster start --all
pcs cluster enable --all
```

Verify Corosync installation

Master should have ID 1 and slave ID 2

```bash
corosync-cfgtool -s
```

### Pacemaker cluster resources

**ON DB1**

Create a new cluster configuration file

One handy feature of `pcs` that it has the ability to queue up several changes into a file and commit those changes atomically. Here we will configure all the resource in a raw XML file.

```bash
pcs cluster cib mycluster
```

Disable the Quorum & STONITH policies in your cluster configuration file

```bash
pcs -f /root/mycluster property set no-quorum-policy=ignore
pcs -f /root/mycluster property set stonith-enabled=false
```

**Note:** The cluster is nothing without `stonith`. The fencing mechanism in a cluster ensures that when a node does not run any resources then it fences the affected node to save from the data corruption.

**Note:** A cluster quorum required when more than half of the nodes are online. This does not make sense in a two-node cluster.

The stickiness prevent the resources from moving after recovery as it usually increases downtime. Execute following command to set stickiness value.

```bash
pcs -f /root/mycluster resource defaults resource-stickiness=300
```

Create a `drbd_mysql` resource for the `DRBD` volume resource, and an additional clone the resource `drbd_mysql` to allow the resource to run on both cluster nodes at the same time.

```bash
pcs -f /root/mycluster resource create drbd_mysql ocf:linbit:drbd drbd_resource=r0 op monitor interval=10s
pcs -f /root/mycluster resource master drbd_master_slave drbd_mysql master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
```

**Note**
**master-max**: how many copies of the resource can be promoted to master status.
**master-node-max**: how many copies of the resource can be promoted to master status on a single node.
**clone-max**: how many copies of the resource to start. Defaults to the number of nodes in the cluster.
**clone-node-max**: how many copies of the resource can be started on a single node.
**notify**: when stopping or starting a copy of the clone, tell all the other copies beforehand and when the action was successful.

Create `mysql` filesystem resource.

```bash
pcs -f /root/mycluster resource create mysql_fs ocf:heartbeat:Filesystem device=/dev/drbd0 directory=/var/lib/mysql fstype=xfs
```

Create `mysql` service resource.

```bash
pcs -f /root/mycluster resource create mysql-server ocf:heartbeat:mysql binary="/usr/bin/mysqld_safe" config="/etc/my.cnf" datadir="/var/lib/mysql" pid="/var/lib/mysql/run/mariadb.pid" socket="/var/lib/mysql/mysql.sock" additional_parameters="--bind-address=0.0.0.0" op start timeout=60s op stop timeout=60s op monitor interval=20s timeout=30s
```

Create `mysql` Virtual IP resource.

```bash
pcs -f /root/mycluster resource create mysql_vip ocf:heartbeat:IPaddr2 ip="192.168.122.100" cidr_netmask=32 op monitor interval=10s
```

Bind the resources in to `mysql-group` group.

```bash
pcs -f /root/mycluster resource group add mysql-group mysql_fs mysql_vip mysql-server
```

Set the constraint order and colocation for cluster resources and list the constraint order.

```bash
pcs -f /root/mycluster constraint colocation add mysql-group with drbd_master_slave INFINITY with-rsc-role=Master
pcs -f /root/mycluster constraint order promote drbd_master_slave then start mysql-group
pcs -f /root/mycluster constraint
```

Execute the following command to commit above changes.

```bash
pcs cluster cib-push mycluster
```

Validate the cluster configuration file and check the cluster status.

```bash
crm_verify -L
pcs status
```

<img src="Resources\DRBD\13.png" style="zoom:150%;" />

# Cluster FailOver Test

**ON DB1**

Execute the following to view cluster status.

```bash
crm_mon -r1
```

Execute the following commands on resource running node to test `mysql` database.

```bash
mysql -u root -p
	show databases;
	create database test3;
	show databases;
	exit
```

Execute following command on cluster nodes to view `DRBD` resource status.

**Both Nodes**

```bash
drbdadm status
```

**Note**
As we see on the above test that, we are able to view the previously created database and as well as able to create new database.
The `DRBD` resource state is also running primary mode on cluster resource running node and secondary node on other cluster node.

Now reboot `db1` and check the cluster status, you would see the resource are moving `db1` to `db2` within few seconds

**ON DB1**

```bash
poweroff
```

<img src="Resources\DRBD\14.png" style="zoom:150%;" />

**ON DB2**

```bash
drbdadm status
crm_mon -r1
mysql -u root -p
	show databases;
	create database test4;
	show databases;
	exit
```

<img src="Resources\DRBD\15.png" style="zoom:150%;" />

Now start cluster service manually on `db1` and check the `DRBD` resource status.

**Both Nodes**

```bash
drbdadm status
pcs status
```

<img src="Resources\DRBD\17.png" style="zoom:150%;" />
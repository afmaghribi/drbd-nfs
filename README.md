# Exec all these steps as root user

# Exec on drbd-1 and drbd-2 vm

## Add DRBD repository

Manually added. Not need to execute if already using vagrant cloud-init

```
add-apt-repository ppa:linbit/linbit-drbd9-stack 
add-apt-repository “deb https://ppa.launchpadcontent.net/linbit/linbit-drbd9-stack/ubuntu focal main”  
apt update -y 
```

## Install packages

Manually install. Not need to execute if already using vagrant cloud-init

```
apt install -y drbd-utils drbd-dkms pacemaker pcs nfs-kernel-server
```

## Prepare nfs disk

Create LVM in additional disk (non-root)

```
parted -s -a opt /dev/vdb mklabel gpt mkpart primary 0% 100%
set 1 lvm on 
pvcreate /dev/vdb1
vgcreate vg_drbd /dev/vdb1
lvcreate -n nfs_data -l 100%FREE vg_drbd
```

## Configura DRBD

Config `/etc/drbd.d/global_common.conf`

```
mv /etc/drbd.d/global_common.conf /etc/drbd.d/global_common.conf.default
nano /etc/drbd.d/global_common.conf

---
global {
  usage-count yes;
}
common {
  net {
    protocol C;
    timeout 10;
    ko-count 1;
    ping-int 1;
  }
}
---
```

Config drbd resources. `/etc/drbd.d/drbd_nfs.res`

```
nano /etc/drbd.d/drbd_nfs.res
---
resource "drbd_nfs" {
  device "/dev/drbd1"; # Device to be created
  disk "/dev/vg_drbd/nfs_data"; # LV created before
  meta-disk internal;

  options {
    quorum majority;
    on-no-quorum suspend-io; # Allows Pacemaker to unmount the filesystem and to demote the DRBD resource to secondary role
  }

  on "drbd-1" {
    address 144.144.144.11:7003;
    node-id 0;
  }

  on "drbd-2" {
    address 144.144.144.12:7003;
    node-id 1;
  }

  connection-mesh {
    hosts "drbd-1" "drbd-2";
  }
}
---
```

Apply drpd resource cluster (?), exec on `drbd-1` and `drbd-2`

```
drbdadm create-md drbd_nfs
drbdadm up drbd_nfs
drbdadm status drbd_nfs  # Both nodes statuses will become inconsistent and secondary

drbdsetup down drbd_nfs # to force delete resource, if error
```

Exec only on `drbd-1`

```
drbdadm new-current-uuid --clear-bitmap drbd_nfs/0
drbdadm primary --force drbd_nfs

drbdadm status drbd_nfs # Both nodes statuses must be UpToDate now
mkfs.xfs /dev/drbd1
```

## Test Operation DRDB NFS

Exec on `drbd-1`

```
// Create mountpoint
mkdir -p /mnt/drbd_nfs
chmod 777 -R /mnt/drbd_nfs

// Set Filesystem and mount disk
mount /dev/drbd1 /mnt/drbd_nfs

// Create testing file
echo "Gw pernah singgah disini" > /mnt/drbd_nfs/test_file.txt
ls /mnt/drbd_nfs/
cat /mnt/drbd_nfs/test_file.txt

// Unmount and set as secondary
umount /mnt/drbd_nfs/
drbdadm secondary drbd_nfs
```

Exec on `drbd-2`

```
// Create mountpoint
mkdir -p /mnt/drbd_nfs
chmod 777 -R /mnt/drbd_nfs

// Set Filesystem and mount disk
mount /dev/drbd1 /mnt/drbd_nfs

// Check file
ls /mnt/drbd_nfs/
cat /mnt/drbd_nfs/test_file.txt

// Unmount and set as secondary
umount /mnt/drbd_nfs/
drbdadm secondary drbd_nfs
```


## Setup NFS Server


```
// Setup NFS shared directory
chown nobody:nogroup /mnt/drbd_nfs
chmod 777 /mnt/drbd_nfs

// Backup default
mv /etc/exports /etc/exports.default

// Setup Exports file
nano /etc/exports
---
/mnt/drbd_nfs/ 144.144.144.0/24(rw,sync,no_subtree_check,no_root_squash)
---

// Apply exports
exportfs -a

// Restart service
systemctl restart nfs-kernel-server
systemctl restart nfs-server
```

## Configure pacemaker and corosync

Exec on `drbd-1` and `drbd-2`

```
pcs cluster destroy
systemctl enable pcsd.service --now
echo hacluster:ezpzpassword | chpasswd #Set password for setup pcs cluster
```

Exec only on `drbd-1`

```
pcs host auth drbd-1 drbd-2
pcs cluster setup ha-drbd-nfs drbd-1 drbd-2
pcs cluster enable --all
pcs cluster start --all
pcs status
```

Exec on `drbd-1` and `drbd-2`

```
systemctl enable pacemaker --now
pcs status # Status online
```

Create configuration file for pacemaker using crm. Exec on `drbd-1` only.

```
nano drbd-nfs.conf

---
node 1: drbd-1
node 2: drbd-2
primitive virtip_drbd_nfs IPaddr2 \
        params \
            ip=144.144.144.10 \                                             # VIP
            cidr_netmask=32 \
        op monitor interval=0s timeout=40s \
        op start interval=0s timeout=20s \
        op stop interval=0s timeout=20s
primitive ha_drbd_nfs ocf:linbit:drbd \
        params \
            drbd_resource=drbd_nfs \                                        # Drbd resource Name
        op monitor timeout=20 interval=21 role=Slave \
        op monitor timeout=20 interval=20 role=Master
primitive ha_nfs_exports exportfs \
        params \
            clientspec="144.144.144.0/24" \                                 # VIP
            directory="/mnt/drbd_nfs" \                                     # Mount point
            fsid=1003 unlock_on_stop=1 options="rw,mountpoint,insecure,sync,no_subtree_check,no_root_squash" \
        op monitor interval=15s timeout=40s \
        op_params OCF_CHECK_LEVEL=0 \
        op start interval=0s timeout=40s \
        op stop interval=0s timeout=120s
primitive ha_nfs_filesystem Filesystem \
        params \
            device="/dev/drbd1" \                                           # Drbd device Name
            directory="/mnt/drbd_nfs" \                                     # Mount Point
            fstype=xfs \                                                    # Device filesystem type
            run_fsck=no \
        op monitor interval=15s timeout=40s \
        op_params OCF_CHECK_LEVEL=0 \
        op start interval=0s timeout=60s \
        op stop interval=0s timeout=60s
primitive nfsserver systemd:nfs-server \
        op monitor interval=30s
ms ms_ha_drbd_nfs ha_drbd_nfs \
        meta master-max=1 master-node-max=1 \
        clone-node-max=1 clone-max=2 notify=true \
        target-role="Promoted" \
        promotable=true
clone cl-nfsserve nfsserver \
        meta interleave=true
colocation co_ha_nfs inf: \
        virtip_drbd_nfs \
        ms_ha_drbd_nfs:Master \
        ha_nfs_exports \
        ha_nfs_filesystem
property cib-bootstrap-options: \
        have-watchdog=false \
        dc-version=2.0.3-4b1f869f0f \
        cluster-infrastructure=corosync \
        cluster-name=ha-drbd-nfs \
        stonith-enabled=false \
        last-lrm-refresh=1667889252 \
        no-quorum-policy=ignore
---

// Backup current
crm configure show > drbd-nfs.conf.bckp

// Apply config
crm configure load replace drbd-nfs.conf

// Clean
pcs resource cleanup

// Check status
crm status
pcs status
```

## Testing access from client

Install nfs client

```
apt install -y nfs-common
```

Try mount

```
mkdir -p /mnt/nfs-client
mount -t nfs 144.144.144.10:/mnt/drbd_nfs /mnt/nfs-client

// Check directory content
ls /mnt/nfs-client/
cat /mnt/nfs-client/test_file.txt 

// Add new file
echo "gw juga pernah singgah disini" > /mnt/nfs-client/test_file2.txt
ls /mnt/nfs-client/
```

Try change primary to `drbd-2`. Exec on `drbd-1`

```
crm node standby
crm status # Online drbd-2
pcs status
```
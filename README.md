## Create GFS2 HA Cluster in RHEL 9 - Step-by-Step
This is just a basic example on how to create a GFS2 cluster filesystem in RHEL 9 HA cluster with pacemaker/corosync. This environment is running in a simple KVM host with two VMs being used as the newly created cluster. Adjust the steps according to your own environment.

### Create two similar VMs . Create an empty shareable raw image and attach in both VMs in QEMU/KMV/Libvirt without formatting it

## Enable repositories
### On each VM node

    # subscription-manager repos \
    --enable=ansible-automation-platform-2.2-for-rhel-9-x86_64-rpms \
    --enable=rhel-9-for-x86_64-appstream-rpms \
    --enable=rhel-9-for-x86_64-baseos-rpms \
    --enable=rhel-9-for-x86_64-highavailability-rpms \
    --enable=rhel-9-for-x86_64-resilientstorage-rpms \
    --enable=rhel-9-for-x86_64-supplementary-rpms \
    --enable=satellite-client-6-for-rhel-9-x86_64-rpms
    # dnf update -y

## Create a pacemaker cluster (https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_high_availability_clusters/assembly_creating-high-availability-cluster-configuring-and-managing-high-availability-clusters)

    # dnf install pcs pacemaker fence-agents-all lvm2-lockd gfs2-utils dlm pcp pcp-pmda-gfs2 pcp-zeroconf -y
    # firewall-cmd --permanent --add-service=high-availability
    # firewall-cmd --add-service=high-availability
    # firewall-cmd --reload
    # passwd hacluster
    # systemctl start pcsd.service
    # systemctl enable pcsd.service
    # pcs host auth gfsa.example.local gfsb.example.local
    # pcs cluster setup hacluster --start gfsa.example.local gfsb.example.local
    # pcs cluster enable --all

## Configuring fencing device (https://access.redhat.com/solutions/917833)
### On KVM host

    # yum install fence-virt fence-virtd fence-virtd-libvirt fence-virtd-multicast fence-virtd-serial -y
    # mkdir -p /etc/cluster
    # dd if=/dev/urandom of=/etc/cluster/fence_xvm.key bs=4k count=1
    # scp /etc/cluster/fence_xvm.key gfsa.example.local:/etc/cluster/
    # scp /etc/cluster/fence_xvm.key gfsb.example.local:/etc/cluster/
    # fence_virtd -c
    # systemctl start fence_virtd
    # systemctl enable fence_virtd
    # firewall-cmd --permanent --add-port=1229/tcp
    # firewall-cmd --reload

### On each VM node

    # rpm -q fence-virt
    # fence_xvm -o list
    # pcs stonith create xvmfence fence_xvm key_file=/etc/cluster/fence_xvm.key

### On one of the VM nodes

    # pcs stonith create hafence-gfs fence_xvm pcmk_host_map="gfsa.example.local:gfsa gfsb.example.local:gfsb" key_file=/etc/cluster/fence_xvm.key pcmk_delay_max=15 --force
    # pcs stonith show --full

## Configure GFS2 with the pacemaker cluster (https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/configuring_and_managing_high_availability_clusters/index#assembly_configuring-gfs2-in-a-cluster-configuring-and-managing-high-availability-clusters)
### On both nodes of the cluster, set the use_lvmlockd configuration option in the /etc/lvm/lvm.conf file to use_lvmlockd=1
### On node A

    # pcs property set no-quorum-policy=freeze
    # pcs resource create dlm --group locking ocf:pacemaker:controld op monitor interval=30s on-fail=fence
    # pcs resource clone locking interleave=true
    # pcs resource create lvmlockd --group locking ocf:heartbeat:lvmlockd op monitor interval=30s on-fail=fence
    # pcs status --full
    # vgcreate --shared shared_vg /dev/vdb

### On node B

    # lvmdevices --adddev /dev/vdb
    # vgchange --lock-start shared_vg

### On node A

    # lvcreate --activate sy -L1G -n shared_lv shared_vg
    # lvs
    # vgs
    # mkfs.gfs2 -j2 -p lock_dlm -t hacluster:gfs2-demo /dev/shared_vg/shared_lv
    # pcs resource create sharedlv --group shared_vg ocf:heartbeat:LVM-activate lvname=shared_lv vgname=shared_vg activation_mode=shared vg_access_mode=lvmlockd
    # pcs resource clone shared_vg interleave=true
    # pcs constraint order start locking-clone then shared_vg-clone
    # pcs constraint colocation add shared_vg-clone with locking-clone
    # lvs
    # pcs resource create sharedfs --group shared_vg ocf:heartbeat:Filesystem device="/dev/shared_vg/shared_lv" directory="/mnt/gfs" fstype="gfs2" options=noatime op monitor interval=10s on-fail=fence
    # mount | grep gfs2
    # pcs status --full

## Example output of a final configuration

    # pcs status --full
    Cluster name: hacluster
    Cluster Summary:
      * Stack: corosync
      * Current DC: gfsa.example.local (1) (version 2.1.2-4.el9-ada5c3b36e2) - partition with quorum
      * Last updated: Wed Aug 10 16:47:30 2022
      * Last change:  Wed Aug 10 16:02:31 2022 by root via cibadmin on gfsa.example.local
      * 2 nodes configured
      * 9 resource instances configured
    
    Node List:
      * Online: [ gfsa.example.local (1) gfsb.example.local (2) ]
    
    Full List of Resources:
      * hafence-gfs	(stonith:fence_xvm):	 Started gfsa.example.local
      * Clone Set: locking-clone [locking]:
        * Resource Group: locking:0:
          * dlm	(ocf:pacemaker:controld):	 Started gfsa.example.local
          * lvmlockd	(ocf:heartbeat:lvmlockd):	 Started gfsa.example.local
        * Resource Group: locking:1:
          * dlm	(ocf:pacemaker:controld):	 Started gfsb.example.local
          * lvmlockd	(ocf:heartbeat:lvmlockd):	 Started gfsb.example.local
      * Clone Set: shared_vg-clone [shared_vg]:
        * Resource Group: shared_vg:0:
          * sharedlv	(ocf:heartbeat:LVM-activate):	 Started gfsa.example.local
          * sharedfs	(ocf:heartbeat:Filesystem):	 Started gfsa.example.local
        * Resource Group: shared_vg:1:
          * sharedlv	(ocf:heartbeat:LVM-activate):	 Started gfsb.example.local
          * sharedfs	(ocf:heartbeat:Filesystem):	 Started gfsb.example.local
    
    Migration Summary:
    
    Tickets:
    
    PCSD Status:
      gfsa.example.local: Online
      gfsb.example.local: Online
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled

    # mount | grep gfs2
    /dev/mapper/shared_vg-shared_lv on /mnt/gfs type gfs2     (rw,noatime,seclabel,rgrplvb)
    
    # pvs
      PV         VG        Fmt  Attr PSize    PFree
      /dev/vda2  VG_01     lvm2 a--    <9.00g    0 
      /dev/vdb   shared_vg lvm2 a--  1020.00m    0 
      
    # vgs
      VG        #PV #LV #SN Attr   VSize    VFree
      VG_01       1   2   0 wz--n-   <9.00g    0 
      shared_vg   1   1   0 wz--ns 1020.00m    0 
      
    # lvs
      LV        VG        Attr       LSize    Pool Origin Data%  Meta%  Move Log     Cpy%Sync Convert
      root      VG_01     -wi-ao----   <8.00g                                                    
      swap      VG_01     -wi-ao----    1.00g                                                    
      shared_lv shared_vg -wi-ao---- 1020.00m       
                                                   
    # lsblk
    NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sr0                    11:0    1 1024M  0 rom  
    vda                   252:0    0   10G  0 disk 
    ├─vda1                252:1    0    1G  0 part /boot
    └─vda2                252:2    0    9G  0 part 
      ├─VG_01-root        253:0    0    8G  0 lvm  /
      └─VG_01-swap        253:1    0    1G  0 lvm  [SWAP]
    vdb                   252:16   0    1G  0 disk 
    └─shared_vg-shared_lv 253:2    0 1020M  0 lvm  /mnt/gfs

## Testing

### On node A
    # cd /mnt/gfs/
    # echo "GFS2 is working" > test.txt

### On node B

    # cat /mnt/gfs/test.txt
    # rm -rf /mnt/gfs/test.txt
    # ls

### On node A

    # ls /mnt/gfs

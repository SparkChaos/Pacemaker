## Prerequisites for Pacemaker


The following commands should be done on both servers.

Firstly, we have to configure the firewall to not interfere with the setup of pacemaker. To do this, issue the following:

```
systemctl stop firewalldsystemctl disable firewalld
```

  

The next step is to disable SELinux from also interfering with the setup of pacemaker, you can disable it by issuing this simple one-liner:

```
sed -i --follow-symlinks 's/^SELINUX=.*/SELINUX=disabled/g' /etc/sysconfig/selinux
```

  

It is recommended to add NTP into your environment and pull from a local stratum server so that the time does not drift and cause pacemaker issues, you can do so by visiting:

```
http://www.pool.ntp.org/zone/europe
```

  

and then by issuing:

```
echo "server servername iburst" >> /etc/ntpd.confecho "server servername iburst" >> /etc/ntpd.conf
```

  

In addition to NTP, it is also recommended to implement SSH keys between your cluster members, you can do this by issuing the following commands on each member:

```
ssh-keygen -b 4096 ssh-copy-id root@yourserver
```

Now that you have interfering services disabled, clocks are synced and communications are secure between the two nodes, you can now begin with installing pacemaker dependencies.

  

To do so, issue the following:

```
yum install nfstest.noarch  net-tools gawk.x86_64  grep.x86_64 gzip.x86_64  rsync.x86_64  rsyslog.x86_64  sed.x86_64  unzip.x86_64  which.x86_64  bc.x86_64  expect.x86_64  httpd.x86_64  nmap.x86_64  sysstat.x86_64  tcpdump.x86_64  lsof.x86_64  sysfsutils.x86_64  lldpad.x86_64  sg3_utils.x86_64   sudo.x86_64   tar.x86_64    genisoimage.x86_64  libmemcached.x86_64  mutt.x86_64   perf.x86_64   perftest.x86_64  traceroute.x86_64      wget.x86_64   logrotate.x86_64  curl.x86_64 ethtool.x86_64  aide bind-utils jpackage-utils glibc-devel.i686  libgcc.i686  sg3_utils.x86_64   sg3_utils-libs.i686 libsysfs.x86_64 libsysfs.i686
```

  

We will now tell the OS not to use the LVM metadata caching daemon. You can keep this enabled if you want data to be read from a cache for speed, but, accuracy is preferred:

```
sed -i --follow-symlinks 's/^use_lvmetad=.*/use_lvmetad=0/g' /etc/lvm/lvm.conf
```

  

If you are currently using volume groups (vgs), you will need to put an exception into /etc/lvm/lvm.conf, to do this, do the following:

```
vgs
```

  

Now you should be presented with a list of volume groups. Note down each one (if applicable) and enter this into /etc/lvm/lvm.conf:

```
volume_list = [ 'vg_root', 'anyothervg' ]
```

  

Next step is to get dracut configured on both systems to boot the hosts up faster. You can read more about dracut by issuing  _man dracut (or visit: https://linux.die.net/man/8/dracut_ ), issue the following:

```
cd /cp -rp boot /tmp/boot_imagesecho "dracut –H –f  /boot/initramfs-$(uname –r).img   $(uname –r)" >> /tmp/dracut_cmd.out
```

  

Then execute the script that you wrote to in /tmp:

```
chmod +x /tmp/dracut_cmd.outsh /tmp/dracut_cmd.out
```

  

Once done, reboot your servers:

```
shutdown -r now
```

## Cluster Setup in Pacemaker

The following commands will need to be done on both servers after you have completed the prerequisites (if you have not, go to:  [Prerequisites for Pacemaker](https://slimwiki.com/milamber/pacemaker-setup)).

  

Once your servers have rebooted and the prompts are accessible again, you will need to ensure that you have the Resilient Storage repo enabled in Red Hat (CentOS - ignore), you can do this by issuing:

```
subscription-manager repos --enable=rhel-rs-for-rhel-7-server-rpms
```

  

You may need to add the aforementioned repo first, you can do this using the below information:

```
Repo ID:   rhel-rs-for-rhel-7-server-eus-rpms
Repo Name: Red Hat Enterprise Linux Resilient Storage (for RHEL 7 Server) - Extended Update Support (RPMs)
Repo URL:  https://cdn.redhat.com/content/eus/rhel/server/7/$releasever/$basearch/resilientstorage/os
Enabled:   1
```

  

```
vi /etc/yum.repos.d/redhat.repo
```

  

Insert the below:

```
[rhel-rs-for-rhel-7-server-rpms]name = Red Hat Enterprise Linux Resilient Storage (for RHEL 7 Server) (RPMs)baseurl = https://cdn.redhat.com/content/eus/rhel/server/7/$releasever/$basearch/resilientstorage/osenabled = 1
```

  

Once this is done, update your system repos and install the cluster necessities:

```
yum -y update; yum install -y pacemaker pcs resource-agents fence-agents-all device-mapper-multipath
```

  

Once done, we will enable friendly names on a service called multipath which detects multiple paths to devices for failover reasons:

```
mpathconf --enable --user_friendly_names y
```

  

Enable and start the multipath daemon:

```
systemctl enable multipathdsystemctl start multipathd
```

  

We will then pass a new password to a user that was just installed called  _hacluster,_ this allows the cluster to communicate using this username:

```
echo somepassword | passwd --stdin hacluster
```

  

We will also change the password expiry of the username to 99,999 days:

```
chage -I -1 -m 0 -M 99999 -E -1  hacluster
```

  

Finally, reboot the machine:

```
reboot
```

  

Once your machines have come back, issue the following on both:

```
systemctl start pcsd.servicesystemctl status pcsd.servicesystemctl enable pcsd.service
```

  

We will now continue using just one member.  **The following does not need to be inserted on both servers, only one that we will use as a "master" if you like.**

  

We will authorise the hacluster user against all members of the cluster:

```
pcs cluster auth <member1_short_hostname> <member2_short_hostname> -u hacluster -p password_you_set --force
```

  

We will now setup the cluster using a cluster name of our choice:

```
pcs cluster setup --name <cluster_name_here> <member1_short_hostname> <member2_short_hostname>
```

  

Enable and start the cluster:

```
pcs cluster enable --allpcs cluster start --all
```

  

Start and enable the corosync and pacemaker services:

```
systemctl start corosync.servicesystemctl start pacemaker.servicesystemctl enable corosync.servicesystemctl enable pacemaker.service
```

  

Issue another reboot and ensure that when the nodes are back that the cluster has started:

```
reboot
```

  

Once the machines are back again and are accessible, issue the following and ensure everything looks OK:

```
pcs cluster statussystemctl status corosync.servicesystemctl status pacemaker.servicesystemctl status pcsd.service
```

  

Next we will have a look at the current rings on the cluster by issuing:

```
corosync-cfgtool -s
```

  

You should have a similar output to:

```
Printing ring status.Local node ID 1RING ID 0        id      = MainIP        status  = ring 0 active with no faultsRING ID 1        id      = 127.0.0.1        status  = ring 1 active with no faults
```

  

If you want an additional private network between the two hosts, you must edit /etc/corosync/corosync.conf and add them in. We will also add rrp_mode whilst we in the file:

  

```
vi /etc/corosync/corosync.conf
```

  

Add the following in the totem section:

```
rrp_mode: passive
```

  

Add the following in the nodelist section:

```
nodelist {    node {        ring0_addr: node01 # Aliased in /etc/hosts        ring1_addr: node01_priv # Aliased in /etc/hosts        nodeid: 1    }    node {        ring0_addr: node02 # Aliased in /etc/hosts        ring1_addr: node02_priv # Aliased in /etc/hosts        nodeid: 2    }}
```

  

Add the following in the quorum section:

```
wait_for_all: 1
```

  

You will notice the above has _priv for the additional address. You can add this into /etc/hosts to simplify your IP changes if it changes in the future:

```
vi /etc/hosts
```

  

Append the following the bottom of the file, changing 10.0.0.* to whatever your IPs are:

```
10.0.0.10  node01_priv10.0.0.11  node02_priv
```

  

Let us s verify that everything is in order by issuing:

```
crm_verify -L -V
```

  

We will also disable STONITH (**S**hoot  **T**he  **O**ther  **N**ode  **I**n  **T**he  **H**ead) for the time being:

```
pcs property set stonith-enabled=false
```

  

Let us confirm it is off:

```
pcs property show stonith-enabled
```

  

Should result in the following output:

```
Cluster Properties: stonith-enabled: false
```

  

To aid in proper failover down the line, lets issue the resource-stickiness command so that we can define a weight for the cluster members:

```
pcs resource defaults resource-stickiness=200
```

  

Congratulations! You now have a working cluster.


## Creating LVM Resources in Pacemaker

  
In order to start adding resources into a pacemaker cluster, we will first have to start by creating a volume group from our physical volumes.

  

To obtain a list of devices we can use for volume group creation, issue the following command on both servers, output should be identical in a shared storage environment:

```
multipath -ll
```

  

You should then see an output and a line(s) starting with the following:

```
mpatha (3600009700001962918394533030329486)
```

  

Let us now take however many of the mpath_x_ we want to use, apply it into a volume group (VG) creation command and making sure to tag it with pacemaker:

```
vgcreate yournewvg /dev/mapper/mpatha  --addtag pacemaker --config 'activation { volume_list = [ "@pacemaker" ] }'
```

  

Let us have a look and see if pacemaker tags have been added by issuing:

```
vgs -a -o+tags
```

  

Now that the tag is visible, lets add a new logical volume (LV) with a pacemaker tag also. You can repeat the below for however many LVs you wish to create:

```
lvcreate -addtag pacemaker -LXG -n lv_yournewlv yournewvg --config 'activation { volume_list = [ "@pacemaker" ] }'
```

  

Lets confirm and see if the pacemaker tags have been added to the LVs by issuing:

```
lvs -a -o+tags
```

  

Next you need to make the filesystem type on your new logical volume, you can do this by issuing:

```
mkfs -t ext4 -m 1 -v /dev/yournewvg/lv_yournewlv
```

  

Now let us remove the pacemaker tag from the newly created LV:

```
lvchange -an yournewvg/lv_yournewlv --deltag pacemaker
```

  

Let us confirm again that the tags have gone by issuing:

```
lvs -a -o+tags
```

  

Now finally let us remove the pacemaker tag from the VG:

```
vgchange  --deltag pacemaker  yournewvg
```

  

Now we add the completed VG into the pacemaker cluster:

```
pcs resource create yournewvg LVM volgrpname=yournewvg exclusive=true --group whateveryourclusternameis
```

  

Let us see if the resource is now showing as it should be:

```
pcs resource show
```

  

Now we will create mount-points so that we can mount our LVs when they are added to the pacemaker cluster:

```
mkdir -p /some/path/to/lv/mount
```

  

Now we add the LVs to the pacemaker cluster much like the VG:

```
pcs resource create yournewlv_lv Filesystem device=/dev/yournewvg/lv_yournewlv  directory=/some/path/to/lv/mount fstype=ext4  --group whateveryourclusternameis
```
### Adding a Virtual IP to Pacemaker configuration.


Adding a Virtual IP (vIP) is fairly straightforward thanks to Pacemaker.

  

Decide which IP you would like to give to your cluster, it should be a vIP that can reside in the same subnet as your main interface.

  

To find all IPs in your system, issue:

```
ip addr
```

  

Notice how I have not issued  _ifconfig_ in the above statement. Ifconfig will not show all interfaces like ip addr will, so be sure to use ip addr!

  

Once you have decided which IP to use as your vIP, you can put it into pacemaker as a resource by issuing the following command:

```
pcs resource create yournewipname1 ocf:heartbeat:IPaddr ip=yourIP cidr_netmask=24 op monitor interval=60s  --group whateveryourclusternameis
```

  

We will also want to make sure that the IP is the first resource started on your cluster, you can ensure that this is the case by issuing:

```
pcs resource group add whateveryourclusternameis yournewipname1 --before yournewvg
```

  

Let us also give it a quick clean to make sure things are in order:

```
pcs resource cleanup yournewipname1
```

### Creating a new Power-fencing resource in Pacemaker


>   
> This section will have two types of power fencing that can be done.  
> We will look at the first option which is for Virtual Machines  **only** in a VMware environment.  
> The second option will be using IPMI over LAN for Physical Servers.

  

### VMware Environment

  

> This section is only for VMware Servers!

  

Let us start by issuing a command to allow resources to run on any node by default:

```
pcs property set symmetric-cluster=true
```

  

The next command will invoke a fencing agent called  _fence_vmware_soap_ which will communicate with VMware and output Virtual machines in a list, you will have to grep for your machines to ensure they are there:

```
/usr/sbin/fence_vmware_soap -z -l 'username' -p 'password' -a server_or_vcenter -o list --ssl-insecure | egrep "(node_name1|node_name2)"
```

  

You should then get a response similar to the output below:

```
node_name1, 428fd921-9483-948e-b8a2-0133d4e936anode_name2, 847ae112-3383,288f-28ae-0133f4de98e
```

  

Perform the same on the other server and you should have the exact same output. If you do not, double-check your command.

  

If everything is OK, let us now go ahead and create the pacemaker resource using the following command:

```
pcs stonith create 
```

  

Let us confirm the resource has been created by issuing:

```
pcs stonith show
```

  

You are done with fencing using  _fence_vmware_soap,_ you can test it using the following command on either server. We will attempt to fence node1 from node2

  

```
[user@node2]$ pcs stonith fence node1
```

  

You should then see an output that shows the node being fenced successfully:

  

```
Node: node1 fenced
```

  

You can repeat the same for node2 from node1 to confirm everything works as expected.

  

### Physical Server Environment

  

> This section is only for Physical Servers!

  

Let us start by issuing a command to allow resources to run on any node by default:

```
pcs property set symmetric-cluster=true
```

  

The next command will invoke the  _fence_ipmilan_ agent to confirm connectivity to the IPMI device on the physical motherboard:

```
fence_ipmilan -a yourNodesPhysicalRemoteControllersIP 
```

  

Depending on what brand of server you have, you may have to tell it to run under a specific user, e.g. OPERATOR, you can do this by appending the command with -L OPERATOR

```
fence_ipmilan -a 
```

  

If the output is similar to the one below, you will be good to proceed with the next part:

```
Status: ON
```

  

We will now create the pacemaker resource using the following command:

  

```
pcs stonith create node1 fence_ipmilan pcmk_host_list="node1" ipaddr="yourNodesPhysicalRemoteControllersIP" login="username" passwd="password" lanplus=1 power_wait=4
```

  

Ensure that you also do it for the second node:

```
pcs stonith create node2 fence_ipmilan pcmk_host_list="node1" ipaddr="yourNodesPhysicalRemoteControllersIP" login="username" passwd="password" lanplus=1 power_wait=4
```

  

> If you have a different privilege level that you need to add on the above command you can either use  _privlvl=_**usertype** or you can use the below to update the pacemaker resource:

```
pcs stonith update node1 privlvl=operatorpcs stonith update node2 privlvl=operator
```

  

Once that has been added, we can confirm that the resource has been created by issuing:

```
pcs stonith show
```

  

You are now done with fencing using IPMI over LAN. You can now test the fencing by issuing the following command. We will attempt to fence node1 from node2:

```
[user@node2]$ pcs stonith fence node1
```

  

You should then see an output that shows the node being fenced successfully:

  

```
Node: node1 fenced
```

  

You can repeat the same for node2 from node1 to confirm everything works as expected.

## Destroying a Cluster

Whilst a cluster is ideal for most scenarios, there may come a point where a cluster is no longer needed for a variety of reasons. In this section, we will removing a pacemaker cluster from an active set of nodes and reverting them back to their original state prior to our initial build out.

  

Firstly there needs to be some prep-work before we can remove the cluster in order to preserve some settings (e.g. LV mounts).

Issue the following command take note of the output:

```
df -kh
```
Once you have taken a note of the filesystems, you will want to issue another command to obtain everything that is mounted in another fashion to show what filesystem types you are using for each filesystem:

```
mount
```

  

You should then get a similar output to below, take note of your output and add it to your documentation:

```
/dev/mapper/rootvg-data on /data type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/rootvg-home on /home type xfs (rw,nodev,relatime,attr2,inode64,noquota)
/dev/sda1 on /boot type xfs (rw,relatime,attr2,inode64,noquota)/dev/mapper/rootvg-tmp on /tmp type xfs (rw,nosuid,nodev,noexec,relatime,attr2,inode64,noquota)
/dev/mapper/rootvg-var on /var type xfs (rw,nosuid,nodev,relatime,attr2,inode64,noquota)
/dev/mapper/rootvg-var_tmp on /var/tmp type xfs (rw,nosuid,nodev,noexec,relatime,attr2,inode64,noquota)
/dev/mapper/rootvg-opt on /opt type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/rootvg-var_log on /var/log type xfs (rw,relatime,attr2,inode64,noquota)
```

  

Finally, before we can begin, you will have to issue a pacemaker command to take a copy of what was active, resource-wise, in the cluster before removing it, this can be accomplished by issuing:

```
pcs resources show --full
```

  

You will then get, depending on how many resources you have, a list of resources:

```
Group: yourGroupName Resource: yourIpName (class=ocf provider=heartbeat type=IPaddr) Attributes: cidr_netmask=24 ip=yourIP1 Operations: monitor interval=30s (yourIpName-monitor-interval-30s)             start interval=0s timeout=20s (yourIpName-start-interval-0s)             stop interval=0s timeout=20s (yourIpName-stop-interval-0s)Resource: yourVgName (class=ocf provider=heartbeat type=LVM)
   Attributes: exclusive=true volgrpname=yourVgName
   Operations: monitor interval=60s timeout=60s (yourVgName-monitor-interval-60s)
               start interval=0s timeout=30 (yourVgName-start-interval-0s)
               stop interval=0s timeout=30 (yourVgName-stop-interval-0s)
Resource: yourLvName (class=ocf provider=heartbeat type=Filesystem)   Attributes: device=/dev/yourVgName/yourLvName directory=yourLvMountpoint fstype=yourLvFilesystemType
   Operations: monitor interval=20 timeout=40 (yourLvName-monitor-interval-20)
               start interval=0s timeout=60 (yourLvName-start-interval-0s)
               stop interval=0s timeout=60 (yourLvName-stop-interval-0s)
```

  

Once you have obtained all this information and recorded it separately, we can start removing pacemaker.

  

Firstly, issue the following command in order to stop the pacemaker cluster entirely:

```
pcs cluster stop --all
```

  

You should then see pacemaker turning off pacemaker, corosync, etc., until you are returned to the prompt. Issue the following command in order to verify pacemaker is off:

```
pcs status
```

  

You should then be greeted with the following error:

```
Error: cluster is not currently running on this node
```

  

You may now proceed to destroy the cluster entirely by issuing:

```
pcs cluster destroy
```

  

Once this has finished, you will no longer have a cluster running on your environment.

  

There are a few more things that are needed in order to fully clean up the environment and have it running fully as a single node.

You will have to remove the changes that were made in the  [Prerequisites for Pacemaker](https://slimwiki.com/milamber/pacemaker-setup)  section, notably the LVM configuration changes. Edit  _/etc/lvm/lvm.conf_ (ideally backing it up prior) with these changes:

  

Comment out:

```
volume_list = [ "yourRootVg", "yourOtherVgIfApplicable" ]
```

  

Find and modify the following parameter to equal to 1:

```
use_lvmetad = 1
```

  

Finally, regenerate the boot images using dracut by doing the following:

```
cd /
cp -rp boot /tmp/boot_images # You are advised to append the date to boot_images folder
echo "dracut –H –f  /boot/initramfs-$(uname –r).img   $(uname –r)" >> /tmp/dracut_cmd.out
chmod +x /tmp/dracut_cmd.out 
```
> Verify the file has hyphens and not dots for each flags. Causing an error here can cause your server not to boot.
```
sh /tmp/dracut_cmd.out
```
You will need to wait for the dracut command to finish and then finally issue:

```
reboot
```

  

Wait for the server to reboot and you are done!

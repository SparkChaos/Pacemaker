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

  

You may now proceed onto the  [Creating LVM Resources in Pacemaker](https://slimwiki.com/milamber/creating-lvm-resources-in-pacemaker)  section.

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

  

You may now go to the next step, [Clustering Setup for Pacemaker]()  while waiting for your servers to reboot.

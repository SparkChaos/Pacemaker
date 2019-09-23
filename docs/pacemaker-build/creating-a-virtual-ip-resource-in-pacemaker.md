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

  

You are done! Now move onto the next step which is  [Creating a new Power Fencing resource in Pacemaker]().

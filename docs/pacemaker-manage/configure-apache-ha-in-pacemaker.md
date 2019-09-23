### Configure Apache HA on Pacemaker

  
Once we have a cluster setup, we will add the Apache (httpd) service to Pacemaker so that it can failover between two nodes.

  

We have to firstly install httpd and then create a status page before enabling the resource in Pacemaker.

  
Firstly, we go and install Apache:

```
yum -y install httpd wget
```

  

Once this has installed, we will need to permit the service through the firewall:

```
firewall-cmd --permanent --add-service={http,https}firewall-cmd --reload
```

  

We need to enable the Apache status URL so that the health of the Apache instance can be monitored:

```
vi /etc/httpd/conf.d/status.conf
```

  

Add this to the configuration file:

```
<Location /server-status>
SetHandler server-status
Require local
</Location>
```

  

Once the configuration file has been configured, you can test this by issuing:

```
wget -O - http://yourServerName/server-status
```

  

Finally, we will add the resource into Pacemaker by issuing the following command:

```
pcs resource create yourWebSiteName ocf:heartbeat:apache configfile=/path/to/httpd/conf statusurl="http://yourserver/server-status" op monitor interval=1min
```

  

On a side note, if you would like to ensure the resources runs on the same node as the virtual IP, we can do so by using  _pcs constraint._ 
Let us issue the following command to ensure it starts with the vIP resource:

```
pcs constraint colocation add yourWebSiteName with yourClusterIP INFINITY
```

  

We will also put in a second constraint so that the Virtual IP will start  _before_ the Apache resource since Apache depends on the Virtual IP:

```
pcs constraint order yourClusterIP then yourWebSiteName 
```

  

You are done!

  

N.B. If you have a webserver that is more powerful than the other corresponding member, you can issue the following  _pcs contraint_ command so that Pacemaker prefers one node over another:

```
pcs contraint location yourWebSite prefers nodeName=100
```

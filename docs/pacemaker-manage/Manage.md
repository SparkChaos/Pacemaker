# Configuring Resources on Pacemaker

## Configure Apache HA on Pacemaker

  
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


N.B. If you have a webserver that is more powerful than the other corresponding member, you can issue the following  _pcs contraint_ command so that Pacemaker prefers one node over another:

```
pcs contraint location yourWebSite prefers nodeName=100
```

## Configure Oracle HA for Pacemaker

>   
> In order to do service fail-over on a Red Hat HA Cluster, you need the following prerequisites, which should have been done if you have been using the Full Clustering Build. These requirements are:

1.  Virtual IP for each Oracle Instance;
2.  Mount point(s) created for each Oracle instance;
3.  Unique user created by each Oracle instance;
4.  Unique listener name setup for each Oracle instance;
5.  Listener.ora file should have a SID name as the binding IP and entry in /etc/hosts; and
6.  Identification of what group you want the resource to be part of in the Red Hat HA.

  

If your environment satisfies all of the above, we can begin!

  

Firstly, we will want to create the Oracle resource in Pacemaker:

  

```
pcs resource create yourOracleResourceName oracle sid=yourOracleSID home=yourOracleHomeLocation user=yourOracleUser --group yourClusterResourceGroup
```

  

Lets check that the resource has successfully started:

  

```
pcs resource show
```

  

If everything looks OK and your resource is now at the bottom of the resource table, let us move on and create the listener resource for Oracle:

  

```
pcs resource create yourOracleListenerResourceName oralsnr sid=yourOracleSID home=yourOracleHomeLocation listener=yourOracleListenerName --group yourClusterResourceGroup
```

  

Let us check the resource and make sure it is listed and started by issuing:

  

```
pcs resource show
```


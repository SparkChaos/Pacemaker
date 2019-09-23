### Configure Oracle HA for Pacemaker

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

  

You are done!

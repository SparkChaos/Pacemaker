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

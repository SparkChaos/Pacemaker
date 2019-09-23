### Creating LVM Resources in Pacemaker

  
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

  

You are done with adding LVM resources to Pacemaker! You can now move onto the next step which is  [Creating a Virtual IP Resource in Pacemaker]().

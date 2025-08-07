One way we can provide clustered storage to our Incus cluster is via LINSTOR.  LINSTOR is a clustered storage layer that uses DRBD to replicate block devices across nodes.  For this example, let's assume you have the LINSTOR Satellite and Controller, OpenJDK, and DRBD installed on each node.  The Satellite nodes will provide a storage device to the pool and the controller node will manage the pool.  For detailed information consult the excellent documentation on the LINBIT website.  This LINSTOR cluster runs on the same nodes that make up the Incus cluster.  That way Incus can keep instance storage local.

After getting the needed packages installed (this will vary per distribution), enable and then start the LINSTOR Satellite service **on all nodes**. 

```bash
# Set the service's rc file as executable
chmod +x /etc/rc.d/rc.linstor-satellite
# Start the service via its rc file
/etc/rc.d/rc.linstor-satellite start
```

You can verify that the LINSTOR Satellite service started correctly by examining its log file.  You can use a command like this to display the log to the screen.  Errors with your OpenJDK installation may appear at this point.

```bash
cat /var/log/linstor/satellite
```

Now, let's choose one node at random to start with.  This node will become the first controller node.  All nodes will be satellite nodes.  On the chosen node, start the controller service.  This will also be the node you will need to issue all `linstor` commands from for now.  For the example, let's use `node4`

```bash
# Set the service's rc file as executable
chmod +x /etc/rc.d/rc.linstor-controller
# Start the service via its rc file
/etc/rc.d/rc.linstor-controller start
```

Similarly to the satellite service, you can examine the log of the controller service to make sure it started correctly.  Errors with Python or the python_linstor module may appear at this point.

```bash
# The LINSTOR log location may be different on your system
cat /var/log/linstor/controller
```

We can also now use the `linstor` command to verify it can communicate with the LINSTOR services.  On the same controller node you were just on, run the following command.  You should get an empty list of nodes and no error messages.

```bash/etc
linstor node list
```

The response should be similar to this:

```text
╭─────────────────────────────────────╮
┊ Node ┊ NodeType ┊ Addresses ┊ State ┊
╞═════════════════════════════════════╡
╰─────────────────────────────────────╯
```

Continuing on the same controller node, add the controller node itself to the LINSTOR cluster.  We are indicating this is a combined node - is hosts both the satellite and the controller service.  Make sure the LINSTOR node name matches the Incus node name (`node4.example.net`) in this case.

```
linstor node create node4.example.net 10.1.31.174 --node-type combined
```

You'll get a lot of information back.  Look for any failures, those will need to be corrected before the LINSTOR cluster will be healthy.  There are a few lines in the output that are worth examining:

```text
   ...

    New node 'node4.example.net' registered.

   ...

    Node 'node4.example.net' authenticated
Details:
    Supported storage providers: [diskless, lvm, lvm_thin, zfs, zfs_thin, file, file_thin, remote_spdk, ebs_init, ebs_target]
    Supported resource layers  : [drbd, luks, nvme, cache, storage]
   ...
```

Note that the node has registered and authenticated to the cluster.  You will also see a list of supported storage providers.  This may vary depending on your use case, but you probably want one or more of `lvm, lvm_thin, zfs, zfs_thin` for an Incus cluster.  We'll use `zfs_thin` in this example for thin provisioning on top of ZFS.  Verify the node list again:

```bash
linstor node list
```

Let's add the other nodes as satellite nodes.  For now, they will only host the satellite service.  You should get a confirmation message similar to the first node after each new node is added.  Make sure your required storage driver, in our case `zfs_thin`, is in the supported storage driver list.  

```
linstor node create node5.example.net 10.1.31.175 --node-type satellite
# Confirmation output will follow, similar to previous step
linstor node create node6.example.net 10.1.31.177 --node-type satellite
# Confirmation output will follow, similar to previous step
```

Should a node fail registration or not have the supported storage driver, you should delete the node from the cluster using the below command.  Fix the node as needed and then re-create the node.  You will only run this command if you need to delete a broken node

```
# Don't use unless needed
#linstor node delete node5.example.net
#
```

Once all satellite nodes are added, you should examine the node list again.  Make sure all nodes are healty.  Any unhealthy nodes should be fixed or removed at this point.  

```bash
linstor node list
# This can be shortened to 'linstor n l'
```

On a healthy cluster with two satellite-only and one combined node, you should see an output like this:

```text
╭───────────────────────────────────────────────────────────────────╮
┊ Node              ┊ NodeType  ┊ Addresses                ┊ State  ┊
╞═══════════════════════════════════════════════════════════════════╡
┊ node4.example.net ┊ COMBINED  ┊ 10.1.31.174:3366 (PLAIN) ┊ Online ┊
┊ node5.example.net ┊ SATELLITE ┊ 10.1.31.175:3366 (PLAIN) ┊ Online ┊
┊ node6.example.net ┊ SATELLITE ┊ 10.1.31.177:3366 (PLAIN) ┊ Online ┊
╰───────────────────────────────────────────────────────────────────╯
```

Now you need to prepare your storage space on the nodes.  This can be an empty disk or a partition on each node.  Make sure the disk or partition is ready prior to running the following commands. In this example, there is an emtpy partition called `/dev/nvme0n1p3` on each node.  Back on the controller node, create the storage pool with the following commands.  The entire storage pool across all nodes will be called `linstor-pool` in LINSTOR.  The local ZFS pool on each node will be called `zfs-pool`.  These names are arbitrary and are your choice.  These names are chosen here to clarify if they represent the ZFS pool, the LINSTOR pool, or the Incus storage pool.

```bash
# There is a still a single contoller, run these commands there
# zfsthin specifies a thinly allocated ZFS pool
linstor physical-storage create-device-pool --pool-name zfs-pool --storage-pool linstor-pool zfsthin node4.example.net /dev/nvme0n1p3
# Verify that you have SUCCESS
linstor physical-storage create-device-pool --pool-name zfs-pool --storage-pool linstor-pool zfsthin node5.example.net /dev/nvme0n1p3
# Verify that you have SUCCESS
linstor physical-storage create-device-pool --pool-name zfs-pool --storage-pool linstor-pool zfsthin node6.example.net /dev/nvme0n1p3
```

Once all the nodes are added, there is nothing else to do.  The LINSTOR pool should be online and ready to connect to the Incus cluster.  To verfify the LINSTOR storage pool is ready for use, run the following.

```bash
linstor storage-pool list
# This can be shortened to 'linstor sp l'
```

You should see an output similar to below.  Notice that all nodes show a state of "Online"

```text
╭───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool          ┊ Node              ┊ Driver   ┊ PoolName ┊ FreeCapacity ┊ TotalCapacity ┊ CanSnapshots ┊ State ┊ SharedName                             ┊
╞═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ DfltDisklessStorPool ┊ node4.example.net ┊ DISKLESS ┊          ┊              ┊               ┊ False        ┊ Ok    ┊ node4.example.net;DfltDisklessStorPool ┊
┊ DfltDisklessStorPool ┊ node5.example.net ┊ DISKLESS ┊          ┊              ┊               ┊ False        ┊ Ok    ┊ node5.example.net;DfltDisklessStorPool ┊
┊ DfltDisklessStorPool ┊ node6.example.net ┊ DISKLESS ┊          ┊              ┊               ┊ False        ┊ Ok    ┊ node6.example.net;DfltDisklessStorPool ┊
┊ linstor-pool         ┊ node4.example.net ┊ ZFS_THIN ┊ zfs-pool ┊   192.81 GiB ┊       199 GiB ┊ True         ┊ Ok    ┊ node4.example.net;linstor-pool         ┊
┊ linstor-pool         ┊ node5.example.net ┊ ZFS_THIN ┊ zfs-pool ┊   192.81 GiB ┊       199 GiB ┊ True         ┊ Ok    ┊ node5.example.net;linstor-pool         ┊
┊ linstor-pool         ┊ node6.example.net ┊ ZFS_THIN ┊ zfs-pool ┊   192.81 GiB ┊       199 GiB ┊ True         ┊ Ok    ┊ node6.example.net;linstor-pool         ┊
╰───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

Now, we need to tell Incus about the LINSTOR cluster.  This is done by changing the Incus configuration to point to the LINSTOR controller node.

```
incus config set storage.linstor.controller_connection=http://node4.example.net:3370
```

Finally, create the storage pool in Incus that is backed by a resource group on the LINSTOR cluster.  First, the pool needs created on each node.  Then the LINSTOR pool is set and the pool is created across the cluster.  Incus will create a resource group with the same name of the storage pool in Incus.  In this case, the resource group will be called `incus-pool`.

```bash
# Configure storage on each node
incus storage create incus-pool linstor --target node4.example.net
incus storage create incus-pool linstor --target node5.example.net
incus storage create incus-pool linstor --target node6.example.net
# Configure storage across the cluster
incus storage create incus-pool linstor linstor.resource_group.storage_pool=linstor-pool
```

Now, let's look at the resource group created by Incus in the LINSTOR cluster.

```bash
linstor resource-group list
# This can be shortened to 'linstor rg l'
```

LINSTOR will show the resource groups.  There should be a resource group with the same name as the Incus storage pool.  `incus-pool` in this case.

```textile
╭─────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceGroup ┊ SelectFilter                 ┊ VlmNrs ┊ Description                     ┊
╞═════════════════════════════════════════════════════════════════════════════════════════╡
┊ DfltRscGrp    ┊ PlaceCount: 2                ┊        ┊                                 ┊
╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
┊ incus-pool    ┊ PlaceCount: 2                ┊        ┊ Resource group managed by Incus ┊
┊               ┊ StoragePool(s): linstor-pool ┊        ┊                                 ┊
╰─────────────────────────────────────────────────────────────────────────────────────────╯
```



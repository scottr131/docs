# Time Synchronization

As with any cluster software, time synchronization is critical.  Make sure NTP is enabled and configured to use the same NTP server as the other nodes.

```bash
chmod +x /etc/rc.d/rc.ntpd
```

# Software Installation

## Additional Slackware Packages

If the D (development) software set was not installed, then a few packages from that set will need installed to support Ceph.

```bash
# GNU Fortran library, Required by SciPy
installpkg gcc-gfortran-15.2.0-x86_64-1.txz
# LUA library, Required by RADOSGW
installpkg lua-5.4.8-x86_64-1.txz
# Python and friends, Required by Ceph
installpkg python3-3.12.11-x86_64-1.txz
installpkg python-pip-25.2-x86_64-1.txz
installpkg python-setuptools-80.9.0-x86_64-1.txz
```

## Runtime Prerequisites

Some additional software is needed to support running Ceph on the system.  This list contains only the runtime software required, even more supporting software is required to build Ceph on Slackware.

```bash
installpkg rdma-core-59.0-x86_64-1_SBo.txz libnbd-1.22.4-x86_64-1_SBo.txz googletest-1.17.0-x86_64-1_SBo.txz benchmark-1.9.4-x86_64-1_SBo.txz snappy-1.2.2-x86_64-1_SBo.txz oath-toolkit-2.6.13-x86_64-1_SBo.txz numactl-2.0.19-x86_64-1_SBo.txz lttng-ust-2.14.0-x86_64-1_SBo.txz babeltrace-1.5.11-x86_64-1_SBo.txz boost-1.89.0-x86_64-1_SBo.txz thrift-0.22.0-x86_64-1_SBo.txz rabbitmq-c-0.15.0-x86_64-1_SBo.txz librdkafka-2.11.1-x86_64-1_SBo.txz bcrypt-4.3.0-x86_64-1-unsafe.txz
```

There are some Python packages required that are not currently packaged as Slackware packages.  These will get packaged later.  For now, install these from `pip`.

```bash
pip install scipy cherrypy jsonpatch python-dateutil cryptography
```

## Install Ceph and Create Directories and User

For now, all the Ceph components are packaged up in a single Slackware package.  Therefore, this package can be used to deploy and type of Ceph node.  This may be split into packages for manager, monitor, OSD, etc.  

```bash
Installpkg ceph-20.1.0-x86_64-test2.txz
```

Ceph will need its own user.  Create a `ceph` group and add a `ceph` user.  In addition, there are some directories in `/etc` and `/var` that need created.  The directories in `/var` need to be owned by `ceph` since the daemons will be running as that user.

```bash
## Create Ceph user (ceph)
groupadd -r ceph
useradd --system -g ceph -c "Ceph" -m -d /home/ceph ceph
# Create Ceph directories
mkdir -p /etc/ceph
mkdir -p /var/lib/ceph/bootstrap-osd
mkdir -p /var/log/ceph
mkdir -p /var/run/ceph
# Make sure ceph user owns the /var directories
chown -R ceph:ceph /var/lib/ceph
chown -R ceph:ceph /var/log/ceph
chown -R ceph:ceph /var/run/ceph
```

# Bootstrap Cluster

The entire cluster will start off with a single monitor.  This monitor will need to be configured with a unique ID for the cluster as well as other various settings.  These settings are typically saved in a configuration file; that will be created here.  Also, the Ceph components authenticate with keys, so a keyring needs generated for the admin user and for the monitor component. 

## Create configuration file

### Global Configuration

Typically, Ceph clusters use a configuration file named after the cluster.  A default Ceph cluster is called `ceph`, therefore we’ll use `ceph.conf` as the configuration file.  The cluster needs a unique identifier, and `uuidgen` can be used to get that.

```bash
# Overwrites existing config
echo "[global]" > /etc/ceph/ceph.conf
echo "fsid = `uuidgen`" >> /etc/ceph/ceph.conf
```

### Initial Monitor Configuration

Ceph needs to know about this initial monitor that we are creating.  This is done by adding the host’s name and IP address to the configuration file.  For this example, the initial monitor node’s hostname is `node4` and its IP address is `172.31.254.14`.

```bash
echo "mon_initial_members = node4" >> /etc/ceph/ceph.conf
echo "mon_host = 172.31.254.14" >> /etc/ceph/ceph.conf
```

## Create Cluster Keyring

This process creates a keyring for the cluster, generates a key for the monitor, and then specifies the capabilities of the monitor.  This will be shown as multiple steps first to make clear what is happening.  The multiple commands can then be combined into a single command for convenience.  These are created in `/tmp` as they will be imported into the initial monitor daemon later.

```bash
# Create the empty cluster keyring in the /tmp directory.
ceph-authtool /tmp/ceph.mon.keyring --create-keyring
# Generate a new key for the entity named `mon.`.
ceph-authtool /tmp/ceph.mon.keyring -n mon. --gen-key 
# Set the capabilities for `mon.`
ceph-authtool /tmp/ceph.mon.keyring -n mon. --cap mon 'allow *'
```

These commands can be combined into a single command.  You only need to run the previous commands or this command – not both.

```bash
#ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

## Create Administrator Keyring

Next is a similar process for the administrator keyring.  This single command will create a keyring at `/etc/ceph/ceph.client.admin.keyring`, generate a new key for an entity called `client.admin`, and grant full capabilities on that entity.

```bash
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *'
```

## Create bootstrap-osd Keyring

The bootstrap-osd keyring is used by the monitor to provision new OSDs.  First the directory is created.  Then a single command creates a new bootstrap-osd keyring at `/var/lib/ceph/bootstrap-osd/ceph.keyring`, generates a new key for an entity called `client.bootstrap-osd`, and grants some limited capabilities to the entity.

```bash
# Create the expected directory
mkdir -p /var/lib/ceph/bootstrap-osd
# Generate the keyring, new key, and grant capabilities
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'
```

## Merge Keyrings

Now the administrator keyring and the bootstrap-osd keyring need to be added to the cluster (monitor) keyring created earlier.  Also make sure the keyring file is owned by the ceph user.

```bash
# Merge admin keyring
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
# Merge bootstrap-osd keyring
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
# Make sure keyring is owned by ceph
chown ceph:ceph /tmp/ceph.mon.keyring
```

## Generate Monitor Map

The monitor map contains a list of all the monitors in the cluster.  A new monitor map will need to be generated containing this single monitor node.  The node name, IP address, and fsid need to match the `/etc/ceph/ceph.conf` file generated earlier.  This initial monitor map is also generated in /tmp as it will be imported into the monitor daemon later.

```bash
# This is run as ceph user so that user owns the file
sudo -u ceph monmaptool --create --add node4 172.31.254.14 --fsid 1ff986af-e0c2-4338-b219-cba29a19659d /tmp/monmap
```

## Create Monitor Data Directory

The monitor daemon will store its data in a directory named after the cluster and monitor - `<cluster_name>-<monitor_host_name>`.  That directory is located under `/var/lib/ceph/mon`.  Create that directory and populate it.  These operations are done as the ceph user to make sure filesystem permissions are correct.  Any permissions errors that occur here need to be resolved before proceeding.

```bash
# Create the monitor’s data directory (as the ceph user)
sudo -u ceph mkdir -p /var/lib/ceph/mon/ceph-node4
# Populate the monitor daemon with the monitor map and keyring
sudo -u ceph ceph-mon --mkfs -i node4 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```

## Finalize Configuration File

There are additional settings that can be configured in the configuration file.  At a minimum, it should look similar to this:

```text
[global]
fsid = a7f64266-0894-4f1e-a635-d0aeaca0e993
mon_initial_members = node4
mon_host = 172.31.254.14
public_network = 172.31.254.0/24
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd_pool_default_size = 3
osd_pool_default_min_size = 2
osd_pool_default_pg_num = 333
osd_crush_chooseleaf_type = 1
```

## Verify File Permissions

Since the various ceph components will run as the user `ceph` and not as `root` the various files just created need to be owned by ceph.  The easiest way to do this is to just reset the ownership information on all effected files.  

```bash
chown -R ceph:ceph /var/lib/ceph
chown -R ceph:ceph /var/log/ceph
chown -R ceph:ceph /var/run/ceph
chown ceph:ceph /etc/ceph/ceph.client.admin.keyring
chown ceph:ceph /etc/ceph/ceph.conf
# From RH docs – what is rbdmap?
chown ceph:ceph /etc/ceph/rbdmap
```

## Start the Monitor

Manually start the monitor. This will be added to an rc file later.  Add a `-f` to keep the process running in the foreground.
```bash
/usr/bin/ceph-mon -c /etc/ceph/ceph.conf -i node4
```


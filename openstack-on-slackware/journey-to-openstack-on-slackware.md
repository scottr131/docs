# The Setup

For this setup, I’ll have 2 PCs.  Each one has an integrated ethernet adapter `eth0` and a USB network adapter `eth1`.  To start, the `eth0` adapters are connected to each other and the internet via a switch.  The `eth1` adapters are connected to each other on a separate network via a VLAN or independent switch.

# The Operating System

I’ll install Slackware on both systems.  Slackware is divided into software sets.  The general recommendation is to install all the sets, but that’s a lot of software.  For both nodes, I’ll install A, AP, L, N, X.  I installed X just so I have a very minimal GUI that I may want to use in the future.

I will need to add these packages from the D disk set since Python and Perl are required for OpenStack and system administration scripts.  These are the official Slackware packages.

```
perl-5.42.0-x86_64-1.txz

python3-3.12.11-x86_64-1.txz

python-setuptools-80.9.0-x86_64-1.txz

python-pip-25.1.1-x86_64-1.txz
```

Speaking of scripts, since Perl wasn't installed during the initial install, I’ll need to run this command to update the CA certificate store.  This is a quirk of Slackware that has bitten me many times and will cause certificate errors everywhere if this is not done.

```
update-ca-certificates --fresh
```

# Account Information

First, I'll make a list of all the passwords I will need for the various accounts on the controller.  There are a lot of accounts in several systems, so a list is helpful.  These aren't great passwords were only used once to create this document and shouldn’t be used for anything.   I won’t create any accounts yet.

| Password Name      | Description                                     | System    | Value                            |
|:------------------ |:----------------------------------------------- |:--------- |:-------------------------------- |
| Database password  | Root password for the database                  | mariadb   | c3a611b04ca3364ea76b0361beecec3f |
| ADMIN_PASS         | Password of user admin                          | —         | 816da0afea288e1a5f10e33273babf40 |
| CINDER_DBPASS      | Database password for the Block Storage service | mariadb   | 203d1a95f0175a90616fe99ef8ce3503 |
| CINDER_PASS        | Password of Block Storage service user cinder   | openstack | 357ddf10f755eacab0c6f82387d0fe09 |
| DASH_DBPASS        | Database password for the Dashboard             | mariadb   | 45a14c5a967c4d6071ff79cac474cb36 |
| DEMO_PASS          | Password of user demo                           | —         | a6d9321f107fa296600c14e9cb643995 |
| GLANCE_DBPASS      | Database password for Image service             | mariadb   | d58dd6f24f21a57ba038978d5c13d734 |
| GLANCE_PASS        | Password of Image service user glance           | openstack | 1fe41b4c9815f8d85dcdc23f18459f81 |
| KEYSTONE_SYS_PASS  | Password of Keystone system user                | system    | db8d443b195a4f78efb7b4f6ffd3714c |
| KEYSTONE_DBPASS    | Database password of Identity service           | mariadb   | bc64aae27ced33198049416c3314daf2 |
| METADATA_SECRET    | Secret for the metadata proxy                   | —         | aeb8f614f3e12c1606454c5545b5286d |
| NEUTRON_DBPASS     | Database password for the Networking service    | mariadb   | d2c4b330bffb9935adca43a6d0a2e3d9 |
| NEUTRON_PASS       | Password of Networking service user neutron     | openstack | 9934ab3b3c944ac42ece8f561d1d0284 |
| NOVA_DBPASS        | Database password for Compute service           | mariadb   | 620a08b6d8f5e4980635ff1bf77a60eb |
| NOVA_SYS_PASS      | Password of Nova system user                    | system    | aeedcb013334fd0c3c9a780c73b653ed |
| NOVA_PASS          | Password of Compute service user nova           | openstack | 0d1a80eabc688896e34e14bca481e354 |
| PLACEMENT_DBPASS   | Database password for Placement service         | mariadb   | 39bfe16e71ae78a8f93ed88e38c47750 |
| PLACEMENT_SYS_PASS | Password of Placement system user               | system    | dace769fe6c4047b0e9e4cfbb70b9c23 |
| PLACEMENT_PASS     | Password of Placement service user placement    | openstack | 6eafd81693f885359d83282e6bedce76 |
| RABBIT_PASS        | Password of RabbitMQ user openstack             | rabbitmq  | 62a319a0482bbb2775a1843925864fc8 |




# Software Package Installation

Now, I'll install a few pieces of software that aren't part
of the OpenStack ecosystem but are used by OpenStack on the controller nodes.  For now, I’m just installing these
components.  I’ll configure them later.  These packages are from my SlackBuilds scripts.

#### etcd

Etcd can be used by OpenStack components for configuration
data storage.  I don’t have any
components that currently use this but may as the OpenStack cluster is expanded.

```bash
installpkg etcd-3.6.4-x86_64-1_SBo.txz
```

#### memcached

Memcached is used across various OpenStack components.

```bash
installpkg memcached-1.6.39-x86_64-1_SBo.txz
```

#### rabbitmq

RabbitMQ Server is used across various OpenStack components.
```bash
installpkg otp_src-27.3.4.2-x86_64-1_SBo.txz
installpkg elixir-1.18.4-x86_64-1_SBo.txz
installpkg rabbitmq-server-4.1.3-x86_64.txz
# Binary will be at
#/usr/local/lib/erlang/lib/rabbitmq_server-4.1.3/sbin/rabbitmq-server
```

#### mod_wsgi
Mod_wsgi is compiled as an Apache module for WSGI.  This is used to serve APIs by various components on the controller.
```bash
installpkg mod_wsgi-5.0.2-x86_64-1_SBo.txz
```

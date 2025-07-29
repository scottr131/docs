### Using Incus

Now that Incus is installed, you need to configure it.  The general configuration process involves adding Incus users to a particular group and then actually configuring Incus itself using the `incus` command.  This is probably a good time to introduce a couple of important programs.

- incusd - This is the actual Incus daemon (server process) itself.  This program runs in the background and manages resources such as instances (containers and virtual machines), storage, and networks.

- incus - This is the command line utility that you will use to interact with Incus.  There are a lot of commands, but it's not necessary to remember then all as `incus` has an excellent built-in help system.  In addition, the commands generally follow a form of `incus <nouns> <verb> <flags> <properties>`.  Describe the item you want to take action on, the action you want to take, and the details of that action.  If you want to take action on an instance, omit the the noun.

### Add trusted local user(s)

By default, only `root` can access Incus.  To allow other local users full access to Incus, add them to the `incus-admin` group.  This should only be done for administrative users who need full control over Incus.  Other users can be added later.  Consult your distribution specific documentation to add a user to the `incus-admin` group, but it can generally be done with a command like:

```bash
# Replace your-user-name with your actual user name
sudo usermod -a -G incus-admin your-user-name
```

### Initialize Incus

At this point, Incus has just been installed and needs to be configured.  In order to get right to using Incus, let's start with a minimal configuration.  This is a simple setup that's great for developers or exploring Incus for the first time.  The configuration stores everything as files on disk and creates a network for instances.

```bash
# Creates a minimal Incus configuration
# WARNING - THIS WILL OVERWRITE ANY EXISTING CONFIG
incus admin init --minimal
```

At this point, Incus should be running.  Let's examine some things.  Incus runs instances (virtual machines and containers), so let's see if anything is running.

```bash
incus list
```

You will see an output similar to below.  There is nothing listed.  That makes sense because we haven't started any instances.  

```

```

We'll start some instances in a minute.  Before that, let's look at the storage that is available to Incus.

```bash
incus storage list
```

You will see something similar to the output below.  This indicates that we have a storage pool called `default` available for use.  This storage pool uses the `dir` driver, which means that it stores everything as files on the root filesystem.  This is fine for small tests, but we'll use a better storage driver later.

```bash
+---------+--------+-------------+---------+---------+
|  NAME   | DRIVER | DESCRIPTION | USED BY |  STATE  |
+---------+--------+-------------+---------+---------+
| default | dir    |             | 1       | CREATED |
+---------+--------+-------------+---------+---------+
```

Let's look at the `default` storage pool in detail.  Note the name of the storage 
pool just happens to be `default` in this case, but a pool named `default` is not required.

```
incus storage show default
```

Incus will output some information about the storage pool.  This information is formatted in YAML.

```yaml
config:
  source: /var/lib/incus/storage-pools/default
description: ""
name: default
driver: dir
used_by:
- /1.0/profiles/default
status: Created
locations:
- none
```

We'll store the first instance in that pool.  Now, let's look at networks available to the instances.

```bash
oot@node1:~# incus network list
```

You'll see something like this.  The list on your system will almost certainly look different because it includes all network interfaces on the host system, not just the Incus networks.

```
+----------------+----------+---------+---------------+---------------------------+-------------+---------+---------+
|      NAME      |   TYPE   | MANAGED |     IPV4      |           IPV6            | DESCRIPTION | USED BY |  STATE  |
+----------------+----------+---------+---------------+---------------------------+-------------+---------+---------+
| br-int         | bridge   | NO      |               |                           |             | 0       |         |
+----------------+----------+---------+---------------+---------------------------+-------------+---------+---------+
| eth0           | physical | NO      |               |                           |             | 0       |         |
+----------------+----------+---------+---------------+---------------------------+-------------+---------+---------+
| wlan0          | physical | NO      |               |                           |             | 0       |         |
+----------------+----------+---------+---------------+---------------------------+-------------+---------+---------+
```

Notice the 'incusbr0' network.

```yaml
config:
  ipv4.address: 10.9.221.1/24
  ipv4.nat: "true"
  ipv6.address: fd42:4f22:bfd1:a93d::1/64
  ipv6.nat: "true"
description: ""
name: incusbr0
type: bridge
used_by:
- /1.0/profiles/default
managed: true
status: Created
locations:
- none
project: default
```

### Running Instances

Now that you are a little more familiar with Incus and the resources available, let's start some instances.  In Incus and instances is a container or a virtual machine.  You can start a container with the below command.  

```bash
incus launch images:alpine/3.22 one
```

Don't expect much output.  If the container starts successfully, Incus will show you that it is downloading the image and starting the cotainer named `one`.  To understand that command better, let's look at the results.

```bash
incus list
```

Like before, Incus will show you the running instances.  This time, there is an instance to see.  It's called `one`.  That's the name you gave it with the `incus launch` command.  This name is a unique identifier you choose to indentify the instance. 

```
+------+---------+---------------------+------+-----------+-----------+
| NAME |  STATE  |        IPV4         | IPV6 |   TYPE    | SNAPSHOTS |
+------+---------+---------------------+------+-----------+-----------+
| one  | RUNNING | 10.56.100.72 (eth0) |      | CONTAINER | 0         |
+------+---------+---------------------+------+-----------+-----------+
```

In addition to the name, Incus shows if the instance is running or stopped, it's IP address, and if the instance is a container or a virtual machine.  So where did Incus get this container image from?  In the `incus launch` command, you specified `images:alpine/3.22`.  The part before the colon (`images` in this case) specifies a remote server.  In Incus, remote servers can other Incus servers, HTTPS sources, or OCI/Docker style registries.  Let's see what remote servers are configured.

```bash
incus remote list
```

Incus will show you a list of all currently configured remote servers.  Sicne this is a new installation of Incus, you should only see your local Incus server (called `local`) and the linuxcontainers.org remote server (called `images`).  Incus downloaded the container image from the remote server named `images` which is configured to connect to imges.linuxcontainers.org.

```
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
|      NAME       |                URL                 |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC | GLOBAL |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
| images          | https://images.linuxcontainers.org | simplestreams | none        | YES    | NO     | NO     |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
| local (current) | unix://                            | incus         | file access | NO     | YES    | NO     |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
```

The default `images` server contains LXC containers.  These are called system containers by Incus because they typically contain an entire operating system.  This is the default type for Incus containers.  Incus can also run application containers.  Application containers typically contain just the files needed to run whatever application is in the container.  Docker containers in the OCI format are application containers.  Let's add the Docker registry to Incus in order to run a Docker container.

```bash
incus remote add docker https://docker.io --protocol=oci
```

With this command, you added a remote server named `docker`.  This name you choose and will use it to refer to this remote server.  The remote server has a URL of `https://docker.io` and is an OCI server (other options are available, such as simplestreams or incus).  For testing, start a busybox instance from the Docker registry.

```bash
incus launch docker:busybox two
```



```bash
incus launch images:alpine/3.22 three --vm -c limits.cpu=1 -c limits.memory=1GiB -c security.secureboot=false
incus launch docker:alpine:3.22 four
```



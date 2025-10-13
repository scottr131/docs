## Set Up Control Plane

Let's start with a node named 'control'.  This has the base installation of Slackware-current as described elsewhere.

### Add User to Sudo

```sh
#su -
# Enter root password
#visudo
# Line 125, uncomment
#usermod -a -G wheel myuser
```

### Configure NTP

> [!CAUTION]
> Time synchronization is very important in a cluster.
> DO NOT SKIP THIS SECTION!  Your cluster will break
> in weird ways.

```sh
#vi /etc/ntp.conf
# Add your ntp server or
# uncomment one from pool
```

Now, make sure the NTP server is enabled.  Then start it.

```sh
sudo chmod +x /etc/rc.d/rc.ntpd
/etc/rc.d/rc.ntpd start
```

### Enable IP Forwarding

Kubernetes will need to use IP forwaring for the networking.  Enable that on startup, and go ahead and enable it now.

```sh
sudo chmod +x /etc/rc.d/rc.ip_forward
sudo /etc/rc.d/rc.ip_forward start
```

### Install Packages

Install Kubernetes, containerd, runc, and other supporting packages.

```sh
sudo installpkg cni-plugins-1.8.0-x86_64.txz containerd-2.1.4-x86_64.txz cri-tools-1.34.0-x86_64.txz \
    kubernetes-1.34.1-x86_64.txz nerdctl-2.1.6-x86_64.txz runc-1.3.2-x86_64.txz
```

### Install Startup/Shutdown Scripts

Download the startup scripts and the scripts the start the start and stop the startup scripts.  This will overwrite any existing files (like `/etc/rc.d/rc.local`).

```sh
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.containerd
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.kubelet
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.local
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.local_shutdown

sudo cp rc.containerd rc.kubelet rc.local rc.local_shutdown /etc/rc.d/

sudo chmod -x /etc/rc.d/rc.kubelet
sudo chmod +x /etc/rc.d/rc.containerd /etc/rc.d/rc.local /etc/rc.d/rc.local_shutdown
```

### Configure CGroups Version

Slackware-current ships with CGroups version 1 enabled.  This can be changed to version 2 for Kubernetes.

```sh
# Either edit /etc/default/cgroups or
sudo sed -i 's/^CGROUPS_VERSION=1$/CGROUPS_VERSION=2/' /etc/default/cgroups
```

### Configure Containerd

The containerd package doesn't include a configuration file.  This will add a basic configuration for containerd.

```sh
sudo mkdir -p /etc/containerd
containerd config default > config.toml
sudo cp config.toml /etc/containerd/

```

### Configure crictl

It's a good time to go ahead and configure crictl to be aware of the containerd endpoint.  This creates a minimal configuration file.

```sh
cat > crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF
sudo cp crictl.yaml /etc/
```

The cgroups change will require a reboot.  containerd should also come up after a reboot.  It's time to reboot the sytem and verify.

```sh
sudo reboot
```

Once the system comes back up, verify that `/sys/fs/cgroup` is mounted as `cgroup2`.  Also, make sure the `containerd` process shows up.  If so, check its log.

```sh
mount | grep cgroup
echo "  ---"
ps ax | grep containerd
echo "  ---"
tail /var/log/containerd.log
```

## Initialize Cluster (on control)

Now it's time to start building the cluster.  This is done on the control plane node.  First, make a config file for kubeadm to tell it to use the cgroupfs cgroupDriver (it defaults to systemd in Kubernetes 1.22 and later)

```sh
cat > kubeadm-config.yaml << EOF
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
kubernetesVersion: v1.34.1
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: cgroupfs
EOF
```

```sh
sudo kubeadm init --config kubeadm-config.yaml
```

Once this says it is waiting for a healthy kubelet, bootstrap the kubelet.

```sh
sudo chmod +x /etc/rc.d/rc.kubelet
sudo /etc/rc.d/rc.kubelet bootstrap
```

After a few minutes, you should recieve a message that the control plane has initialized successfully.  Save the `kubeadm join` command for later.

Per the instructions from kubeadm, copy the admin kubeconfig to your user account.

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install Calico Networking

Kubernetes needs a networking layer to function.  There are several options, but Calico works well for this configuration.

```sh
curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

## Set Up Worker Node

Let's configure the first worker node - `worker1`.  This has the base installation of Slackware-current as described elsewhere.  Additional worker nodes are configured the same way.

### Add User to Sudo

```sh
#su -
# Enter root password
#visudo
# Line 125, uncomment
#usermod -a -G wheel myuser
```

### Configure NTP

> [!CAUTION]
> Time synchronization is very important in a cluster.
> DO NOT SKIP THIS SECTION!  Your cluster will break
> in weird ways.

```sh
#vi /etc/ntp.conf
# Add your ntp server or
# uncomment one from pool
```

Now, make sure the NTP server is enabled.  Then start it.

```sh
sudo chmod +x /etc/rc.d/rc.ntpd
/etc/rc.d/rc.ntpd start
```

### Enable IP Forwarding

Kubernetes will need to use IP forwaring for the networking.  Enable that on startup, and go ahead and enable it now.

```sh
sudo chmod +x /etc/rc.d/rc.ip_forward
sudo /etc/rc.d/rc.ip_forward start
```

### Install Packages

Install Kubernetes, containerd, runc, and other supporting packages.

```sh
sudo installpkg cni-plugins-1.8.0-x86_64.txz containerd-2.1.4-x86_64.txz cri-tools-1.34.0-x86_64.txz \
    kubernetes-1.34.1-x86_64.txz nerdctl-2.1.6-x86_64.txz runc-1.3.2-x86_64.txz
```

### Install Startup/Shutdown Scripts

Download the startup scripts and the scripts the start the start and stop the startup scripts.  This will overwrite any existing files (like `/etc/rc.d/rc.local`).

```sh
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.containerd
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.kubelet
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.local
wget https://github.com/scottr131/linux/raw/refs/heads/main/slackware/etc/rc.d/rc.local_shutdown

sudo cp rc.containerd rc.kubelet rc.local rc.local_shutdown /etc/rc.d/

sudo chmod -x /etc/rc.d/rc.kubelet
sudo chmod +x /etc/rc.d/rc.containerd /etc/rc.d/rc.local /etc/rc.d/rc.local_shutdown
```

### Configure CGroups Version

Slackware-current ships with CGroups version 1 enabled.  This can be changed to version 2 for Kubernetes.

```sh
# Either edit /etc/default/cgroups or
sudo sed -i 's/^CGROUPS_VERSION=1$/CGROUPS_VERSION=2/' /etc/default/cgroups
```

### Configure Containerd

The containerd package doesn't include a configuration file.  This will add a basic configuration for containerd.

```sh
sudo mkdir -p /etc/containerd
containerd config default > config.toml
sudo cp config.toml /etc/containerd/

```

### Configure crictl

It's a good time to go ahead and configure crictl to be aware of the containerd endpoint.  This creates a minimal configuration file.

```sh
cat > crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF
sudo cp crictl.yaml /etc/
```

The cgroups change will require a reboot.  containerd should also come up after a reboot.  It's time to reboot the sytem and verify.

```sh
sudo reboot
```

Once the system comes back up, verify that `/sys/fs/cgroup` is mounted as `cgroup2`.  Also, make sure the `containerd` process shows up.  If so, check its log.

```sh
mount | grep cgroup
echo "  ---"
ps ax | grep containerd
echo "  ---"
tail /var/log/containerd.log
```

## Join Cluster

Now it's time to join the worker node to the cluster.  The command was displayed when the control plane was created.  Run it now on the worker node.

```sh
sudo kubeadm join ...
```


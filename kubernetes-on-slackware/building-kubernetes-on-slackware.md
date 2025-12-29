# Build Kubernetes on Slackware

## Prepare Build Environment

### Remove GCC Go and GCC Rust

These packages seem to interfere with the official Go and Rust implementations.

```sh
sudo removepkg gcc-go-15.2.0-x86_64-1.txz
sudo removepkg gcc-rust-15.2.0-x86_64-1.txz
```

### Set Environment Properties

These may need to be adjusted for your enviroment.

```sh
export MYARCH=x86_64
export MYTHREADS=16
export MYCFLAGS="-march=x86-64 -mtune=generic -pipe -O2 -fPIC"
export MYCCFLAGS="-march=x86-64 -mtune=generic -pipe -O2 -fPIC"
```

## Build Prerequisites

### containerd

```sh
export MYPKG=containerd
export MYVERSION=2.2.1
export MYURL=https://github.com/containerd/containerd/archive/refs/tags/v2.2.1.tar.gz
```

#### Download and Extract

Download the source code from GitHub and rename it to match the desired naming scheme.  Then extract the source.

```sh
wget ${MYURL}
mv v${MYVERSION}.tar.gz ${MYPKG}-${MYVERSION}.tar.gz
tar xf ${MYPKG}-${MYVERSION}.tar.gz
```

#### Compile

containerd has an included Makefile and can be compiled with `make`.

```sh
cd ${MYPKG}-${MYVERSION}
make VERSION=${MYVERSION} REVISION=1 BUILDTAGS="seccomp apparmor cri"
```

> [!NOTE]
> If containerd is not built from the Git repository,
> the build flags must be included for proper version
> information to be included in the resulting binaries.
> 
> If the build flags are not set, you may get errors
> similar to:
> 
> ```
> E1228 22:33:40.139900    1587 kuberuntime_manager.go:280] "Get runtime version  failed" err="get remote runtime typed version failed: not all fields are set in VersionResponse (missing RuntimeVersion)"
> E1228 22:33:40.139944    1587 run.go:72] "command failed" err="failed to run Kubelet: failed to create kubelet: get remote runtime typed version failed: not all fields are set in VersionResponse (missing RuntimeVersion)"
> ```

#### Package

containerd will install via `make install` and will install to the folder specified to by DESTDIR

```sh
# Create destination folder
sudo mkdir -p /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}
# Do the Install
sudo make install DESTDIR=/root/installs/${MYPKG}-${MYVERSION}-${MYARCH} VERSION=${MYVERSION}

# Create a package
sudo bash -c "cd /root/installs/${MYPKG}-${MYVERSION}-${MYARCH} && makepkg --linkadd y --chown y /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz"
```

#### Install the resulting package

```sh
sudo installpkg /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz
```

### cri-tools

```sh
export MYPKG=cri-tools
export MYVERSION=1.34.0
export MYURL=https://github.com/kubernetes-sigs/cri-tools/archive/refs/tags/v1.34.0.tar.gz
```

#### Download and Extract

Download the source code from GitHub and rename it to match the desired naming scheme.  Then extract the source.

```sh
wget ${MYURL}
mv v${MYVERSION}.tar.gz ${MYPKG}-${MYVERSION}.tar.gz
tar xf ${MYPKG}-${MYVERSION}.tar.gz
```

#### Compile

cri-tools has an included Makefile and can be compiled with `make`.

```sh
cd ${MYPKG}-${MYVERSION}
make -j $MYTHREADS
```

#### Package

cri-tools will install via `make install` and will install to the folder specified to by DESTDIR

```sh
# Create destination folder
sudo mkdir -p /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}
# Do the Install
sudo make install DESTDIR=/root/installs/${MYPKG}-${MYVERSION}-${MYARCH}

# Create a package
sudo bash -c "cd /root/installs/${MYPKG}-${MYVERSION}-${MYARCH} && makepkg --linkadd y --chown y /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz"
```

#### Install the resulting package

```sh
sudo installpkg /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz
```

### cni-plugins

```sh
export MYPKG=cni-plugins
export MYVERSION=1.8.0
export MYURL=https://github.com/containernetworking/plugins/archive/refs/tags/v1.8.0.tar.gz
```

#### Download and Extract

Download the source code from GitHub and rename it to match the desired naming scheme.  Then extract the source.

```sh
wget ${MYURL}
mv v${MYVERSION}.tar.gz ${MYPKG}-${MYVERSION}.tar.gz
tar xf ${MYPKG}-${MYVERSION}.tar.gz
mv plugins-${MYVERSION}/ cni-plugins-${MYVERSION}/
```

#### Compile

cni-plugins has an included build script for Linux.  It will build the binaries in the `bin` subdirectory of the source directory.

```sh
cd ${MYPKG}-${MYVERSION}
./build_linux.sh
```

#### Package

cni-plugins binaries will need to be manually copied into the `/opt/cni/bin` folder of the package folder.

```sh
# Create destination folder
sudo mkdir -p /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}/opt/cni/bin
# Do the Install
sudo cp bin/* /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}/opt/cni/bin

# Create a package
sudo bash -c "cd /root/installs/${MYPKG}-${MYVERSION}-${MYARCH} && makepkg --linkadd y --chown y /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz"
```

#### Install the resulting package

```sh
sudo installpkg /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz
```

### runc

```sh
export MYPKG=runc
export MYVERSION=1.3.2
export MYURL=https://github.com/opencontainers/runc/archive/refs/tags/v1.3.2.tar.gz
```

#### Download and Extract

Download the source code from GitHub and rename it to match the desired naming scheme.  Then extract the source.

```sh
wget ${MYURL}
mv v${MYVERSION}.tar.gz ${MYPKG}-${MYVERSION}.tar.gz
tar xf ${MYPKG}-${MYVERSION}.tar.gz
```

#### Compile

runc has an included Makefile and can be compiled with `make`.

```sh
cd ${MYPKG}-${MYVERSION}
make
```

#### Package

To install runc, the binary will be copied into `/usr/local/bin` in the package folder.

```sh
# Create destination folder
sudo mkdir -p /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}
# Do the Install
sudo make install DESTDIR=/root/installs/${MYPKG}-${MYVERSION}-${MYARCH}

# Create a package
sudo bash -c "cd /root/installs/${MYPKG}-${MYVERSION}-${MYARCH} && makepkg --linkadd y --chown y /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz"
```

#### Install the resulting package

```sh
sudo installpkg /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz
```

### kubernetes

```sh
export MYPKG=kubernetes
export MYVERSION=1.34.1
export MYURL=https://github.com/kubernetes/kubernetes/archive/refs/tags/v1.34.1.tar.gz
```

#### Download and Extract

Download the source code from GitHub and rename it to match the desired naming scheme.  Then extract the source.

```sh
wget ${MYURL}
mv v${MYVERSION}.tar.gz ${MYPKG}-${MYVERSION}.tar.gz
tar xf ${MYPKG}-${MYVERSION}.tar.gz
```

#### Compile

cri-tools has an included Makefile and can be compiled with `make`.

```sh
cd ${MYPKG}-${MYVERSION}
make -j $MYTHREADS
```

#### Package

cri-tools will install via `make install` and will install to the folder specified to by DESTDIR

```sh
# Create destination folder
sudo mkdir -p /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}/usr/local/bin
# Do the Install
sudo cp _output/local/bin/linux/amd64/* /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}/usr/local/bin

# Create a package
sudo bash -c "cd /root/installs/${MYPKG}-${MYVERSION}-${MYARCH} && makepkg --linkadd y --chown y /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz"
```

#### Install the resulting package

```sh
sudo installpkg /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz
```

### nerdctl

```sh
export MYPKG=nerdctl
export MYVERSION=2.1.6
export MYURL=https://github.com/containerd/nerdctl/archive/refs/tags/v2.1.6.tar.gz
```

#### Download and Extract

Download the source code from GitHub and rename it to match the desired naming scheme.  Then extract the source.

```sh
wget ${MYURL}
mv v${MYVERSION}.tar.gz ${MYPKG}-${MYVERSION}.tar.gz
tar xf ${MYPKG}-${MYVERSION}.tar.gz
```

#### Compile

containerd has an included Makefile and can be compiled with `make`.

```sh
cd ${MYPKG}-${MYVERSION}
make -j $MYTHREADS
```

#### Package

containerd will install via `make install` and will install to the folder specified to by DESTDIR

```sh
# Create destination folder
sudo mkdir -p /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}/usr/local/bin
# Do the Install
sudo cp _output/nerdctl /root/installs/${MYPKG}-${MYVERSION}-${MYARCH}/usr/local/bin/

# Create a package
sudo bash -c "cd /root/installs/${MYPKG}-${MYVERSION}-${MYARCH} && makepkg --linkadd y --chown y /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz"
```

#### Install the resulting package

```sh
sudo installpkg /root/packages/${MYPKG}-${MYVERSION}-${MYARCH}.txz
```

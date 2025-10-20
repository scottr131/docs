# Building OpenStack 2025.1 Epoxy on Slackware

## Set Up Build Environment

### Packages

Make sure package sets A, AP, D, K, L, N, TCL, and X are installed.

Remove GCC Go.  It seems to cause nothing but problems, and I can't find a situation where it is needed over the official Go.  Slackware-current contains official Go that will be the default when GCC Go is removed.

```sh
sudo removepkg gcc-go-15.2.0-x86_64-1
```

7zip is needed during build.  Install the binary and make sure it is called 7z and in the path.

## Build Third Party Supporting Software

### etcd

Source: <https://github.com/etcd-io/etcd>  
Build Instructions: <https://etcd.io/docs/v3.4/install/>
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/etcd>
Build time: 45 sec

This package provides the etcd configuration service.  It is used by **openstack**. **etcd** is built in Go via a *Makefile*.  The files from the `bin` directory need to be copied into the `usr/local/bin` directory in the package.

```sh
./scripts/build.sh

mkdir -p $PKG/usr/local/bin
cp bin/* $PKG/usr/local/bin/
```

### memcached

Source: <https://memcached.org/downloads>
Build Instructions: <https://memcached.org/downloads>
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/memcached>
Build time: 15 sec

This package provides the memcached distributed object caching system.  It is used by **openstack**. **memcached** uses a typical *Autoconf* build process.

```sh
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}

make install DESTDIR=$PKG
```

### erlang-otp

Source: <https://www.erlang.org/>
Build Instructions: <https://www.erlang.org/docs/27/system/install>
SlackBuild:  <https://github.com/scottr131/slackbuilds/tree/main/erlang-otp>
Build time: 3 min 30 sec

Tested with version 27.  **rabbitmq-server** is *not compatible with 28*.  This package provides the Erlang language and libraries.  It is used by **elixir** and **RabbitMQ server**. **erlang-otp** uses a typical *Autoconf* build process.

```sh
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}

make install DESTDIR=$PKG
```

### elixir

Source: <https://elixir-lang.org/>
Build Instructions: <https://elixir-lang.org/install.html#compiling-from-source>
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/elixir>
Build time:

This package provides the Elixir language on the Erlang VM.  **It requires erlang-otp to be installed prior to build.**  It is used by **openstack**. **elixir** provides a *Makefile* for its build process.

```sh

```

### rabbitmq-server

Source: <https://github.com/rabbitmq/rabbitmq-server>
Build Instructions:
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/>
Build time: 5 min (check)

This package provides a RabbitMQ messaging and streaming broker.  **It requires 7z, elixir, and erlang-otp to be installed prior to build.**  It is used by **openstack**. **rabbitmq-server** provides a *Makefile* for its build process.

### mod_wsgi

Source: <https://github.com/GrahamDumpleton/mod_wsgi>
Build Instructions: <https://github.com/GrahamDumpleton/mod_wsgi>
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/>
Build time: 14 min

### Qemu

I have an existing Slackbuild for this.

### OpenVSwitch

I have an existing Slackbuild for this.

### OVN

I have an existing Slackbuild for this.

### OpenStack Projects installed via pip

These are currently installed via pip, but I plan to integrate them with the other OpenStack components later.
stevedore
taskflow
cliff
castellan

### Pip Packages

cliff
typing_extensions
autopage
cmd2
cliff (requires PrettyTable and stevedore)
alembic (requires sqlalchemy (requires greenlet))
cryptography
iso8601
os_service_types
debtcollector (requires wrapt)
netaddr
tzdata
rfc3986
msgpack
pycadf
pyjwt
webob
python-dateutil
bcrypt
flask (requires werkzeug, itsdangerous, click, blinker)
flask-restful (requires pytz, aniso8601)
osprofiler (actually openstack component - where is source tarball URL? - req fasteners, oslo.concurrency)
jsonschema (rpds-py, attrs, referencing, jsonschema-specifications)
oauthlib
pysaml2 (requires elementpath, defusedxml, xmlschema, cryptography, pyopenssl)
scrypt
testresources
testscenarios (requires testtools)
eventlet
paste
pastedeploy
routes (requires repoze.lru)
prometheus-client
statsd
amqp (requires vine)
futurist
kombu
pyasn1 (per notes, keystone install guide)
pymysql (per notes)
castellan
cursive
httplib2
retrying
taskflow
WSME (requires simplegeneric)
microversion_parse
os-resource-classes
ovs (requires sortedcontainers, OpenVSwitch)
ovsdbapp (requires fixtures)
pyroute2
pecan
tooz (requires voluptuous)
ncclient (requires invoke, pynacl, paramiko)
setproctitle
websockify (requires redis, jwcrypto)

### OpenStack Packages

These provide the core OpenStack services.  When possible, they are downloaded from tarballs.openstack.org.  When that wasn't possible, they are downloaded from tarballs.opendev.org/openstack.  These are built with `setup.py`:

```sh
python3 setup.py install --root=$PKG
```

openstacksdk (decorator, dogpile-cache jmespath jsonpatch (req jsonpointer) requestsexceptions - keystoneauth1)
osc-lib ( - keystoneauth1 oslo-i18n osl-utils)
keystoneauth1
keystonemiddleware (req oslo-cache oslo-context oslo-log - pycadf pyjwt webob)
keystone (req keystonemiddleware bcrypt flask flask-restful jsonschema oauthlib osprofiler pysaml2 scrypt)
glance_store
glance (req oslo.reports, oslo.limit, python_barbicanclient, glance-store - castellan, cursive , httplib2, retrying,taskflow, wsme)
openstack_placement (aka placement, from notes needs microversion_parse  os-resource-classes)
os_ken (requires ncclient, ovs)
os_vif (requires ovsdbapp, pyroute2)
os_traits
os-win
neutron-lib (requires os-ken, os-traits - pecan, setproctitle)
neutron (requires neutron-lib, python-designateclient, python-novaclient - os-ken, os-vif, ovs, ovsdbapp, pecan, pyroute2, tooz)
os_brick
nova (requires os_brick - websockify)

### Oslo Packages

Oslo provides Python libraries containing code shared by OpenStack projects.  These libraries can be built and installed with `setup.py`

```sh
python3 setup.py install --root=$PKG
```

oslo-i18n
oslo-utils
oslo.config
oslo.serialization (requires msgpack)
oslo.cache
oslo.context
oslo.log (requires python-dateutil)
oslo.concurrency
oslo.db (requires testresources, testscenarios )
oslo.messaging (requires amqp, futurist, kombu, oslo-metrics, oslo-service)
oslo.middleware (requires statsd)
oslo.policy
oslo.upgradecheck
oslo.metrics (prometheus-client)
oslo.service (eventlet, paste, pastedeploy, routes, yappi)
oslo.reports
oslo.limit 
oslo.versionedobjects
oslo.privsep
oslo.rootwrap

# Python Clients

These provide Python client libraries for the various OpenStack services.  These can be built and installed with `setup.py` or the Python build module.  With the build module:

```sh
python3 -m build
pip install dist/*.whl --root=$PKG --root-user-action ignore
```

With setup.py:

```sh
python3 setup.py install --root=$PKG
```

python-openstackclient (requires rfc3986 olso-serialization oslo-config oslo-i18n oslo-utils openstacksdk keystoneauth1 python_cinderclient python_keystoneclient)
python_cinderclient
python_keystoneclient
python_designateclient
python_novaclient
python-barbicanclient
python-glanceclient
python-neutronclient
# runc

## Introduction

`runc` is a CLI tool for spawning and running containers on Linux according to the OCI specification.


## Building


In order to enable seccomp support you will need to install `libseccomp` on your platform.
> e.g. `libseccomp-devel` for CentOS, or `libseccomp-dev` for Ubuntu

```bash
# create a 'github.com/opencontainers' in your GOPATH/src
cd github.com/opencontainers
git clone https://github.com/opencontainers/runc
cd runc

make
sudo make install
```

You can also use `go get` to install to your `GOPATH`, assuming that you have a `github.com` parent folder already created under `src`:

```bash
go get github.com/opencontainers/runc
cd $GOPATH/src/github.com/opencontainers/runc
make
sudo make install
```

`runc` will be installed to `/usr/local/sbin/runc` on your system.

### Docker 
```
wget https://download.docker.com/linux/debian/dists/buster/pool/stable/amd64/docker-ce-cli_20.10.9~3-0~debian-buster_amd64.deb 
wget https://download.docker.com/linux/debian/dists/buster/pool/stable/amd64/docker-ce_20.10.9~3-0~debian-buster_amd64.deb 
wget https://download.docker.com/linux/debian/dists/buster/pool/stable/amd64/docker-compose-plugin_2.6.0~debian-buster_amd64.deb 

dpkg -i docker-ce_20.10.9~3-0~debian-buster_amd64.deb
dpkg -i docker-compose-plugin_2.6.0~debian-buster_amd64.deb
dpkg -i docker-ce-cli_20.10.9~3-0~debian-buster_amd64.deb

```
### RunC test

```
mkdir test-runc
cd test-runc
docker pull hello-world
docker export $(docker create hello-world) > hello-world.tar
mkdir rootfs
tar -C rootfs -xf hello-world.tar
runc spec
// sed -i 's;"sh";"/hello";' ` + specConfig + `
runc run test
runc list

runc checkpoint test01
runc list
runc restore test02
runc list


runc list
runc checkpoint --leave-running  --image-path  ./image --work-path ./work test
runc checkpoint --pre-dump --image-path  ./image --work-path ./work test

```


# Checkpoint & Restore Performance
Example one: kitex 

```
git clone https://github.com/cloudwego/kitex-examples.git
cd kitex-examples/
docker build -t kitex-examples .
cd ..
docker export $(docker create kitex-examples) > kitex-examples.tar
mkdir test-runc-kitex-examples
cd test-runc-kitex-examples
mkdir rootfs 
tar -C rootfs -xf ../kitex-examples.tar
runc spec 
runc run kitex-examples
```
Example two: ngnix
```
docker pull nginx
docker export $(docker create nginx) > nginx.tar
mkdir test-runc-nginx
cd test-runc-nginx
mkdir rootfs
tar -C rootfs -xf ../nginx.tar 
runc spec
runc run nginx
```
Example three: hello-world
```
docker pull hello-world
docker export $(docker create hello-world) > hello-world.tar
mkdir test-runc-hello-world
cd test-runc-hello-world
mkdir rootfs
tar -C rootfs -xf ../hello-world.tar 
runc spec
runc run hello-world
```

Example four: spark
```
docker pull apache/spark:latest
docker export $(docker create apache/spark:latest) > spark.tar
mkdir test-runc-spark
cd test-runc-spark
mkdir rootfs
tar -C rootfs -xf ../spark.tar 
runc spec
runc run spark

```

checkpoint and restore evaluation
```
runc list
runc checkpoint --leave-running  --image-path  ./image --work-path ./work test
runc restore test
```

## runc with network 

Should use 5.15 with CONFIG_RSEQ=y case PTRACE_GET_RSEQ_CONFIGURATION , https://lore.kernel.org/lkml/161598681294.398.14135404653803937904.tip-bot2@tip-bot2/

host上打上这个patch应该就可以了， 打完之后就没有类似错误了

set up network
```
sudo ip netns ls
sudo ip netns delete alpine_network
sudo ip netns add alpine_network
sudo ip link add name veth-host type veth peer name veth-alpine
sudo ip link set veth-alpine netns alpine_network
sudo ip link ls
sudo ip -netns alpine_network link ls

sudo ip netns exec alpine_network ip addr add 192.168.10.1/24 dev veth-alpine
sudo ip netns exec alpine_network ip link set veth-alpine up
sudo ip netns exec alpine_network ip link set lo up
sudo ip -netns alpine_network addr
sudo ip link set veth-host up
sudo ip route add 192.168.10.1/32 dev veth-host
sudo ip route
sudo ip netns exec alpine_network ip route add default via 192.168.10.1 dev veth-alpine

ping 192.168.10.1
```

tomcat example

```
$ mkdir -p tomcat_container/rootfs
$ cd tomcat_container
$ docker export $(docker run -d tomcat:9-jre11) |tar -C rootfs -xv
$ sudo apt install default-jdk
```

```
vim rootfs/usr/local/tomcat/bin/catalina.sh 
# add

export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```



```
$ ls -l /var/run/netns
```

Add below scripts to config.json
```
“namespaces”: [
 {
 “type”: “pid”
 },
 {
 “type”: “network”,
 “path”: “/var/run/netns/alpine_network”
 },
```


```
“args”: [
“/usr/local/tomcat/bin/catalina.sh”,”run”
 ]
```

```
“user”: {
 “uid”: 1000,
 “gid”: 1000
 }
“root”: {
 “path”: “rootfs”,
 “readonly”:false
 },
```

run tomcat
```
$ runc run tomcat
```


outside
```
$ curl http://192.168.10.1:8080
```

check & restore
```
runc checkpoint test

runc restore test

curl http://192.168.10.1:8080

runc list
# get PID
nsenter -t PID --net bash
ping IP_HOST

```


-  redis example

redis server on host

```
wget http://download.redis.io/releases/redis-6.0.5.tar.gz
tar xzf redis-6.0.5.tar.gz
cd redis-6.0.5
make
make test
make install

sudo ip route
vim redis.conf # add ip on host
redis-server redis.conf 
```

redis client in runc

```
root@n223-247-006:~/test-runc-redis# cat rootfs//usr/local/run.sh
#!/bin/bash
/usr/local/bin/redis-cli -h 172.17.0.1 # add ip on host

```

check and restore test

```
root@n223-247-006:~/CRIU-test/test-runc-redis# runc checkpoint --tcp-established test
216670124
root@n223-247-006:~/CRIU-test/test-runc-redis# runc restore --tcp-established test
start container %v
 1675229836359660200
restore %v
 1675229836420799355
```


debug

```
nsenter -t 416682 --net bash
ping # add ip on host
```

## Using runc


```bash
# run as root
cd /mycontainer
runc create mycontainerid

# view the container is created and in the "created" state
runc list

# start the process inside the container
runc start mycontainerid

# after 5 seconds view that the container has exited and is now in the stopped state
runc list

# kill container 
runc kill testcontainerid KILL


# now delete the container
runc delete mycontainerid
```



#### Rootless containers
`runc` has the ability to run containers without root privileges. This is called `rootless`. You need to pass some parameters to `runc` in order to run rootless containers. See below and compare with the previous version.

**Note:** In order to use this feature, "User Namespaces" must be compiled and enabled in your kernel. There are various ways to do this depending on your distribution:
- Confirm `CONFIG_USER_NS=y` is set in your kernel configuration (normally found in `/proc/config.gz`)
- Arch/Debian: `echo 1 > /proc/sys/kernel/unprivileged_userns_clone`
- RHEL/CentOS 7: `echo 28633 > /proc/sys/user/max_user_namespaces`

Run the following commands as an ordinary user:
```bash
# Same as the first example
mkdir ~/mycontainer
cd ~/mycontainer
mkdir rootfs
docker export $(docker create busybox) | tar -C rootfs -xvf -

# The --rootless parameter instructs runc spec to generate a configuration for a rootless container, which will allow you to run the container as a non-root user.
runc spec --rootless

# The --root parameter tells runc where to store the container state. It must be writable by the user.
runc --root /tmp/runc run mycontainerid
```

#### Supervisors

`runc` can be used with process supervisors and init systems to ensure that containers are restarted when they exit.
An example systemd unit file looks something like this.

```systemd
[Unit]
Description=Start My Container

[Service]
Type=forking
ExecStart=/usr/local/sbin/runc run -d --pid-file /run/mycontainerid.pid mycontainerid
ExecStopPost=/usr/local/sbin/runc delete mycontainerid
WorkingDirectory=/mycontainer
PIDFile=/run/mycontainerid.pid

[Install]
WantedBy=multi-user.target
```

## More documentation

* [cgroup v2](./docs/cgroup-v2.md)
* [Checkpoint and restore](./docs/checkpoint-restore.md)
* [systemd cgroup driver](./docs/systemd.md)
* [Terminals and standard IO](./docs/terminals.md)
* [Experimental features](./docs/experimental.md)



# Core Ops Tutorial

# Pre-requisites

Build etcdctl and fleetctl:

```
go get github.com/coreos/etcd/etcdctl
go get github.com/coreos/fleet/fleetctl
```

## Setting up a CoreOS cluster using Vagrant

We'll use Vagrant to set up a local CoreOS cluster with 3 nodes.

### Fetch and enter the repository

```
git clone https://github.com/blixtra/coreos-vagrant.git
cd coreos-vagrant
```

### Modify the cloud-init

```
cp user-data.sample user-data
cp config.rb.sample config.rb
```

For the user-data file, we need to replace the "<token>" placeholder with the discovery token obtian via the following command.

```
curl -w "\n" 'https://discovery.etcd.io/new?size=3'
```

You'll see here that we've specified the size of the cluster. This size is also what we need to enter at the top of the config.rb file.

Now we can start the cluster

```
vagrant up
vagrant status
```

should give us something like this

```
Current machine states:

core-01                   running (virtualbox)
core-02                   running (virtualbox)
core-03                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run 'vagrant status NAME'.
```

We should have 3 CoreOS instances running and they should be aware of each other.

Before we start exploring, we'll need the IP address of one of the nodes in the cluster.

```
vagrant ssh core-01
```

Now that we're in the box we can get the IP address.

```
ifconfig
```

Mine is 172.17.8.101

# Exploring

Let's set up a couple environment variables

```
# Note: this works in a fish shell
set -x FLEETCTL_ENDPOINT http://172.17.8.101:4001
set -x ETCDCTL_PEERS 172.17.8.101:4001
```

Let's take a look at our machines

```
fleetctl list-machines
```

This will give us a list of all the machines in our cluster.

```
MACHINE     IP            METADATA
2c337019... 172.17.8.102  -
ada3f69c... 172.17.8.101  -
e530512e... 172.17.8.103  -
```

## etcd

Creating a dir

```
etcdctl mkdir /test
```

Setting a value

```
etcdctl set /test/hello "Hello World!"
```

This key is now available from all nodes

You can run the following from another node to see the key change as it happens.

```
etcdctl watch /test/hello
```

## fleet

# Deploying a simple unit

We've got a very simple unit file available called hello.service

```
fleetctl submit hello.service
fleetctl list-unit-files
fleetctl list-units
```

We've not submitted the unit files but there's it's noe on a machine yet.

Let's now schedule it onto a machine

```
fleetctl load hello.service

```

It's still not running yet, though. Let's change that.

```
fleetctl start hello.service
```

# Deploying multiple instances

We do the above but use a templated unit file

```
fleetctl submit hello@.service
```

Now we can load multiple instances using the same unit file

```
fleetctl start hello@1.service
fleetctl start hello@2.service
```

# Running Global units

Often we want to run a service on each host. This is done using the Global-True X-Fleet option in the unit file.


```
fleetctl start global.service
```

## Debugging tools

```
/usr/bin/toolbox
```

```
yum install htop
```

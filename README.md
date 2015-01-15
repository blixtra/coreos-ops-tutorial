# Core Ops Tutorial

# Pre-requisites

Build etcdctl and fleetctl:

```
go get github.com/coreos/etcd/etcdctl
go get github.com/coreos/fleet/fleetctl
```

Build kubecfg:

```
git clone https://github.com/GoogleCloudPlatform/kubernetes
make
cp ./_output/local/bin/darwin/amd64/kubecfg ~/bin
```

Google Cloud SDK: https://cloud.google.com/sdk/

# Provisioning CoreOS, etcd, fleet, and flannel

## Control Node

```
gcloud compute instances create control \
--image-project coreos-cloud \
--image coreos-alpha-494-0-0-v20141108 \
--boot-disk-size 200GB \
--machine-type n1-standard-1 \
--can-ip-forward \
--scopes compute-rw \
--metadata-from-file user-data=control.yaml \
--zone us-central1-a
```

### Setup Client SSH Tunnels

```
gcloud compute instances list
```

#### etcd

```
ssh -f -nNT -L 4001:127.0.0.1:4001 core@<control-external-ip>
```

#### kubernetes

```
ssh -f -nNT -L 8080:127.0.0.1:8080 core@<control-external-ip>
```

#### fleet

```
export FLEETCTL_TUNNEL="<control-external-ip>"
```

### Update configs

```
sed -i "" -e 's/CONTROL-NODE-INTERNAL-IP/<control-internal-ip>/g' *.{json,yaml,service}
```

### Cluster Configuration

#### flannel

```
etcdctl --no-sync set /coreos.com/network/config '{"Network":"10.244.0.0/16"}'
```

## Worker Nodes

```
gcloud compute instances create node1 node2 node3 node4 node5 \
--image-project coreos-cloud \
--image coreos-alpha-494-0-0-v20141108 \
--boot-disk-size 200GB \
--machine-type n1-standard-1 \
--can-ip-forward \
--metadata-from-file user-data=node.yaml \
--zone us-central1-a
```

```
gcloud compute instances list
```

```
fleetctl list-machines
```

---

# Cross container networking with flannel

## View the subnet allocations in etcd 

```
etcdctl --no-sync ls / --recursive
```

## Communicate between two containers

### Terminal 1

```
gcloud compute ssh core@node1
```

```
docker run -t -i busybox /bin/sh -c 'ifconfig eth0 && nc -l -p 80'
```

### Terminal 2

```
gcloud compute ssh core@node2
```

Replace eth0-ip with the ip address from above.

```
docker run -t -i busybox /bin/sh -c 'nc <eth0-ip>:80'
```

---

# Logging with logentries.com

```
etcdctl --no-sync set /logentries.com/token <token>
```

```
fleetctl start journal-2-logentries.service
```

```
fleetctl list-units
```

---

# Sysadmin tools with Toolbox

## Terminal 1

```
gcloud compute ssh core@node1
```

```
/usr/bin/toolbox
```

```
yum install tcpdump
```

```
ip addr show flannel0
```

```
tcpdump -i flannel0
```

## Terminal 2

```
gcloud compute ssh core@node2
```

```
ping <node1-flannel0-ip>
```

---

# Kubernetes

## Installing Kubernetes with fleet

```
fleetctl start kube-kubelet.service 
fleetctl start kube-proxy.service
fleetctl start kube-apiserver.service
fleetctl start kube-controller-manager.service
fleetctl start kube-scheduler.service
fleetctl start kube-register.service
```

```
fleetctl list-units
```

```
kubecfg list minions
```

## Deploying applications

### Creating a replicationController

```
cat hello-stable-controller.json
```

```
kubecfg -c hello-stable-controller.json create replicationControllers
```

```
kubecfg list replicationControllers
kubecfg list pods
```

### Horizontally scaling pods

Edit: hello-stable-controller.json

```
"replicas": 4
```

#### Update the replication controller

```
kubecfg -c hello-stable-controller.json update replicationControllers/helloStableController
```

```
kubecfg list pods
```

### Creating and managing services

```
cat hello-service.json
```

```
kubecfg -c hello-service.json create services
```

```
gcloud compute firewall-rules create default-allow-hello --allow tcp:80
```

```
gcloud compute instances list
```

## Rolling Updates

### Send the Canary

```
cat hello-canary-controller.json 
```

```
kubecfg -c hello-canary-controller.json create replicationControllers
```

```
kubecfg list pods
```

```
gcloud compute instances list
```

```
while true; do curl http://<node-external-ip>; echo; sleep 2; done
```

### Rolling Update

#### Terminal 1

```
while true; do curl http://<node-external-ip>; echo; sleep 2; done
```

#### Terminal 2

```
kubecfg --image "quay.io/kelseyhightower/hello:2.0.0" rollingupdate helloStableController
```

#### Terminal 3

```
watch kubecfg list pods
```

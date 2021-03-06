# flannel

flannel (originally [rudder](http://comments.gmane.org/gmane.linux.coreos.devel/1683)) is an overlay network that gives a subnet to each machine for use with
Kubernetes. 

In Kubernetes every machine in the cluster is assigned a full subnet. The machine A
and B might have 10.0.1.0/24 and 10.0.2.0/24 respectively. The advantage of
this model is that it reduces the complexity of doing port mapping. The
disadvantage is that the only cloud provider that can do this is GCE.

## Theory of Operation

To emulate the Kubernetes model from GCE on other platforms we need to create
an overlay network on top of the network that we are given from cloud
providers. flannel uses the Universal TUN/TAP device and creates an overlay network
using UDP to encapsulate IP packets. The subnet allocation is done with the help
of etcd which maintains the overlay to actual IP mappings.

The following diagram demonstrates the path a packet takes as it traverses the
overlay network:

![Life of a packet](./packet-01.png)

## Building flannel

* Step 1: Make sure you have Linux headers installed on your machine. On Ubuntu, run ```sudo apt-get install linux-libc-dev```. On Fedora/Redhat, run ```sudo yum install kernel-headers```.
* Step 2: Git clone the flannel repo: ```https://github.com/coreos/flannel.git```
* Step 3: Run the build script: ```cd flannel; ./build```

Alternatively, you can build flannel in a docker container with the following command. Replace $SRC with the absolute path to your flannel source code:

```
docker run -v $SRC:/opt/flannel -i -t google/golang /bin/bash -c "cd /opt/flannel && ./build"
```

## Packaging flannel into Docker container
Once flannel has been built (see above), you can optionally put it into a Docker container.
The source tree contains Dockerfile at top level. Changed into that directory and replacing $TAG
with a tag for the image, run:

```
docker build -t $TAG .
```

## Configuration

flannel reads its configuration from etcd. By default, it will read the configuration
from ```/coreos.com/network/config``` (can be overridden via --etcd-prefix).
The value of the config should be a JSON dictionary with the following keys:

* ```Network``` (string): IPv4 network in CIDR format to use for the entire overlay network. This
is the only mandatory key.

* ```SubnetLen``` (number): The size of the subnet allocated to each host. Defaults to 24 (i.e. /24) unless
the Network was configured to be smaller than a /24 in which case it is one less than the network.

* ```SubnetMin``` (string): The beginning of IP range which the subnet allocation should start with. Defaults
to the first subnet of Network.

* ```SubnetMax``` (string): The end of the IP range at which the subnet allocation should end with. Defaults to
the last subnet of Network.

* ```Backend``` (dictionary): Type of backend to use and specific configurations for that backend.  The list
of available backends and the keys that can be put into the this dictionary are listed below. Defaults to
"udp" backend.

### Backends
* udp: use UDP to encapsulate the packets.
  * ```Type``` (string): ```udp```
  * ```Port``` (number): UDP port to use for sending encapsulated packets. Defaults to 8285

* alloc: only perform subnet allocation (no forwarding of data packets)
  * ```Type``` (string): ```alloc```

* vxlan: use in-kernel VXLAN to encapsulate the packets.
  * ```Type``` (string): ```vxlan```
  * ```VNI```  (number): VXLAN Identifier (VNI) to be used. Defaults to 1

### Example configuration JSON

The following configuration illustrates the use of most options.

```
{
	"Network": "10.0.0.0/8",
	"SubnetLen": 20,
	"SubnetMin": "10.10.0.0",
	"SubnetMax": "10.99.0.0",
	"Backend": {
		"Type": "udp",
		"Port": 7890
	}
}
```

### Firewalls
When using ```udp``` backend, flannel uses UDP port 8285 for sending encapsulated packets.
When using ```vxlan``` backend, kernel uses UDP port 8472 for sending encapsulated packets.
Make sure that your firewall rules allow this traffic for all hosts participating in the overlay network.

## Running

Once you have pushed configuration JSON to etcd, you can start flannel. If you published your
config at the default location, you can start flannel with no arguments. flannel will acquire a
subnet lease, configure its routes based on other leases in the overlay network and start
routing packets. Additionally it will monitor etcd for new members of the network and adjust
its routing table accordingly.

After flannel has acquired the subnet and configured the TUN device, it will write out an
environment variable file (```/run/flannel/subnet.env``` by default) with subnet address and
MTU that it supports.

## Key command line options

```
-etcd-endpoint="http://127.0.0.1:4001": etcd endpoint
-etcd-prefix="/coreos.com/network": etcd prefix
-iface="": interface to use (IP or name) for inter-host communication. Defaults to the interface for the default route on the machine.
-subnet-file="/run/flannel/subnet.env": filename where env variables (subnet and MTU values) will be written to
-v=0: log level for V logs. Set to 1 to see messages related to data path
```

## Zero-downtime restarts
When running in VXLAN mode, the kernel is providing the data path with flanneld acting as the control plane. As such, flanneld
can be restarted (even to do an upgrade) without disturbing existing flows. However, this needs to be done in few seconds as ARP
entries can start to timeout requiring the flanneld daemon to refresh them. Also, to avoid interruptions during restart, the configuration
must not be changed (e.g. VNI, --iface value).

## Docker integration

Docker daemon accepts ```--bip``` argument to configure the subnet of the docker0 bridge. It also accepts ```--mtu``` to set the MTU
for docker0 and veth devices that it will be creating. Since flannel writes out the acquired subnet and MTU values into
a file, the script starting Docker daemon can source in the values and pass them to Docker daemon:

```bash
source /run/flannel/subnet.env
docker -d --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```

Systemd users can use ```EnvironmentFile``` directive in the .service file to pull in ```/run/flannel/subnet.env```

## CoreOS integration

On CoreOS it is useful to add flannel configuration into .service file in the cloud-config as the following snippet demonstrates:

```
  - name: flannel.service
    command: start
    content: |
      [Unit]
      Requires=etcd.service
      After=etcd.service

      [Service]
      ExecStartPre=-/usr/bin/etcdctl mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'
      ExecStart=/opt/bin/flannel
```

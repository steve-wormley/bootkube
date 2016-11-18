## OpenStack Quickstart

### Configure Security Groups

The security group created here will be referenced later in this guide as k8s-sg, make a node of the name or ID if you choose a different name.

```
$ openstack security group create --description "Security group for k8s cluster" k8s-sg
```

Next, create the security group rules.

```
$ openstack security group rule create --proto tcp --dst-port 22 --src-ip 0.0.0.0/0 k8s-sg
$ openstack security group rule create --proto tcp --dst-port 443 --src-ip 0.0.0.0/0 k8s-sg
$ openstack security group rule create --proto tcp --dst-port 0-65535 --src-group k8s-sg k8s-sg
```

### Create a key-pair

```
$ openstack keypair create k8s-key > k8s-key.pem
$ chmod 400 k8s-key.pem
```

### Install Image

To find the latest CoreOS alpha/beta/stable images, please see the [CoreOS OpenStack Documentation](https://coreos.com/os/docs/latest/booting-on-openstack.html).

```
$ wget https://stable.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2
$ bunzip2 coreos_production_openstack_image.img.bz2
$ openstack image create --container-format bare --disk-format qcow2 --file coreos_production_openstack_image.img CoreOS
```

### Choose a network.

Choose an appropriate network for your cluster. It should have a router with Internet access and be able to allocate public IP addresses.

```
$ openstack network list
+--------------------------------------+----------+--------------------------------------+
| ID                                   | Name     | Subnets                              |
+--------------------------------------+----------+--------------------------------------+
| xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx | network1 | yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy |
...
```
Replace K8S_NET_ID with the ID or Name you want to use in the below commands.


### Launch Nodes

First we create a port for our new instance, this is needed so the instance has the public IP address assigned on boot.

```
$ openstack port create --network <K8S_NET_ID> k8s-node1-port
...
| id                    | x |
...
```
Make a note of the id for use below in <K8S_PORT_ID>


Next we will allocate a public IP for the port

```
$ openstack floating ip list
+--------------------------------------+---------------------+------------------+--------------------------------------+
| ID                                   | Floating IP Address | Fixed IP Address | Port                                 |
+--------------------------------------+---------------------+------------------+--------------------------------------+
| abcde0000000000000000000000000000000 | 10.100.10.17        | None             | None                                 |
```
If no floating IPs are available and unused then allocate a new IP. Replace public if the name of your floating pool differs.
Make a note of the ID for <K8S_PUBLIC_ID> and the Floating IP Address for <PUBLIC_IP>

```
$ openstack floating ip create public
...
| ip          | 1.2.3.4                 |
| id          | z                       |
...
```
Make a note of the ID for <K8S_PUBLIC_ID> and the IP for <PUBLIC_IP>

Assign the IP to your port where <K8S_PORT_ID> and <K8S_PUBLIC_ID> is the information from above

```
$ neutron floatingip-associate <K8S_PUBLIC_ID> <K8S_PORT_ID>
```

In the command below, replace <K8S_PORT_ID> with the port ID from above

```
$ openstack server create --image CoreOS --flavor m1.small --security-group k8s-sg --key-name k8s-key --nic port-id=<K8S_PORT_ID> k8s-node1
```

Verify the server has completed its build

```
$ openstack server show k8s-node1
...
| addresses                            | network1=4.5.6.7,1.2.3.4                                |
...
| status                               | ACTIVE                                                   |
...
```
Verify 'ACTIVE' for status and the <PUBLIC_IP> should be listed under addresses.


### Bootstrap Master

We can then use the public-ip to initialize a master node:

```
$ IDENT=k8s-key.pem ./init-master.sh <PUBLIC_IP>
```

After the master bootstrap is complete, you can continue to add worker nodes. Or cluster state can be inspected via kubectl:

```
$ kubectl --kubeconfig=cluster/auth/kubeconfig get nodes
```

### Add Workers

Run the `Launch Nodes` step for each additional node you wish to add, then using the public-ip, run:

```
IDENT=k8s-key.pem ./init-worker.sh <PUBLIC_IP> cluster/auth/kubeconfig
```

**NOTE:** It can take a few minutes for each node to download all of the required assets / containers.
 They may not be immediately available, but the state can be inspected with:

```
$ kubectl --kubeconfig=cluster/auth/kubeconfig get nodes
```

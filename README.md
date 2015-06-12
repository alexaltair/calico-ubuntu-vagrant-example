# Getting started with Calico on Docker

Calico provides IP connectivity between Docker containers on different hosts (as well as on the same host).

*In order to run this example you will need a 2-node Linux cluster with Docker and etcd installed and running.*  You can do one of the following.
* Set this up yourself, following these instructions: [Manual Cluster Setup](./ManualClusterSetup.md)
* Use [Calico Ubuntu Vagrant][calico-ubuntu-vagrant] to start a cluster in VMs on your laptop or workstation.

If you want to get started quickly and easily then we recommend just using Vagrant.

If you have difficulty, try the [Troubleshooting Guide](./Troubleshooting.md).

### A note about names & addresses
In this example, we will use the server names and IP addresses from the [Calico Ubuntu Vagrant][calico-ubuntu-vagrant] example.

| hostname   | IP address   |
|------------|--------------|
| ubuntu-01  | 172.17.8.100 |
| ubuntu-02  | 172.17.8.101 |

If you set up your own cluster, substitute the hostnames and IP addresses assigned to your servers.

## Starting Calico services<a id="calico-services"></a>

Once you have your cluster up and running, start calico on all the nodes

On ubuntu-01

    sudo ./calicoctl node --ip=172.17.8.100 --node-image=calico/node:libnetwork

On ubuntu-02

    sudo ./calicoctl node --ip=172.17.8.101 --node-image=calico/node:libnetwork

This will start a container. Check they are running

    docker ps

You should see output like this on each node

    vagrant@ubuntu-02:~$ docker ps -a
    CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS              PORTS                                            NAMES
    39de206f7499        calico/node:libnetwork   "/sbin/my_init"        2 minutes ago       Up 2 minutes                                                         calico-node
    5e36a7c6b7f0        quay.io/coreos/etcd      "/etcd --name calico   30 minutes ago      Up 30 minutes       0.0.0.0:4001->4001/tcp, 0.0.0.0:7001->7001/tcp   quay.io-coreos-etcd

## Creating networked endpoints

Now you can start any other containers that you want within the cluster, using normal docker commands. To get Calico to network them, first use the `docker network` command to create a network.

    docker network create --driver=calico net1
    docker network create --driver=calico net2
    docker network create --driver=calico net3

When you create a container add `--net <network name>` to specify the docker network they should be on.

So let's go ahead and start a few of containers on each host.

On ubuntu-01

    docker run --net net1 --name workload-A -tid busybox
    docker run --net net2 --name workload-B -tid busybox
    docker run --net net1 --name workload-C -tid busybox

On ubuntu-02

    docker run --net net3 --name workload-D -tid busybox
    docker run --net net1 --name workload-E -tid busybox

Now, check that A can ping C and E. You can get a containers IP by running

    docker inspect --format "{{ .NetworkSettings.IPAddress }}" <container name>

    docker exec workload-A ping -c 4 192.168.0.3
    docker exec workload-A ping -c 4 192.168.0.5

Also check that A cannot ping B or D:

    docker exec workload-A ping -c 4 192.168.0.2
    docker exec workload-A ping -c 4 192.168.0.4

By default, networks are configured so that their members can communicate with one another, but workloads in other networks cannot reach them.  B and D are in their own networks so shouldn't be able to ping anyone else.

Finally, to clean everything up (without doing a `vagrant destroy`), you can run

    sudo ./calicoctl reset


## IPv6
To connect your containers with IPv6, first make sure your Docker hosts each have an IPv6 address assigned.

On ubuntu-01

    sudo ip addr add fd80:24e2:f998:72d6::1/112 dev eth1

On ubuntu-02

    sudo ip addr add fd80:24e2:f998:72d6::2/112 dev eth1

Verify connectivity by pinging.

On ubuntu-01

    ping6 fd80:24e2:f998:72d6::2

Then restart your calico-node processes with the `--ip6` parameter to enable v6 routing.

On ubuntu-01

    sudo ./calicoctl node --ip=172.17.8.100 --ip6=fd80:24e2:f998:72d6::1 --node-image=calico/node:libnetwork

On ubuntu-02

    sudo ./calicoctl node --ip=172.17.8.101 --ip6=fd80:24e2:f998:72d6::2 --node-image=calico/node:libnetwork

Then, you can start containers with IPv6 connectivity by giving them an IPv6 address in `CALICO_IP`. By default, Calico is configured to use IPv6 addresses in the pool fd80:24e2:f998:72d6/64 (`calicoctl pool add` to change this).

On ubuntu-01

    docker run -e CALICO_IP=fd80:24e2:f998:72d6::1:1 --name workload-F -tid phusion/baseimage:0.9.16
    ./calicoctl profile add PROF_F_G
    ./calicoctl profile PROF_F_G member add workload-F

Note that we have used `phusion/baseimage:0.9.16` instead of `busybox`.  Busybox doesn't support IPv6 versions of network tools like ping.  Baseimage was chosen since it is the base for the Calico service images, and thus won't require an additional download, but of course you can use whatever image you'd like.

One ubuntu-02

    docker run -e CALICO_IP=fd80:24e2:f998:72d6::1:2 --name workload-G -tid phusion/baseimage:0.9.16
    ./calicoctl profile PROF_F_G member add workload-G
    docker exec workload-G ping6 -c 4 fd80:24e2:f998:72d6::1:1

[calico-ubuntu-vagrant]: https://github.com/Metaswitch/calico-ubuntu-vagrant-example
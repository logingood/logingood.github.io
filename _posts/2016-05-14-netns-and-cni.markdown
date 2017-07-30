---
layout: post
title:  "Experiments with container networking: Part 1"
date:   2016-05-14 20:35:28 +1000
categories: kubernetes cni
---
# Linux Network namespaces and CNI

{:toc}

## Introduction
[Container Network Interface](https://github.com/containernetworking/cni) is a great approach that allows to build container networking using different control and forwarding plane implementations available for Linux. In this post lets make quick introduction into Linux network namespaces and Container Network Interface.
First of all to get better understanding what is CNI we will look into linux containers network principles.

{% include image.html url="/images/NS-intro.png" description="Introduction into Network Namespaces" %}

This diagram represents the way how network namespaces are organised on the typical node which uses containerisation.

Usually there is a bridge interface that called docker0, kbr0, cni0 and etc. For simplicity we named it *bridge0*. This bridge interface links together interfaces which are named vethXXXXX. I guess you noticed that if you are running couple of docker containers on your linux box there are two vethXXXXX interfaces in your ```ip addr``` or ```ifconfig``` output.

This interfaces are representing a *root namespace* leg of the link between root and container namespaces. Lets look at the output below:

{% highlight bash %}
bridge mdb
#=> dev docker0 port vetheeed635 grp ff02::1:ff11:2 temp
#=> dev docker0 port veth78a60be grp ff02::1:ff11:3 temp
{% endhighlight %}
We have a docker0 bridge with two interfaces connected to it: ```vetheeed635``` and ```veth78a60be```.
To find out its opposite leg index, we can use ```ethtool```:
{% highlight bash %}
ethtool -S vetheeed635
#=> NIC statistics:
#=>     peer_ifindex: 7
ethtool -S veth78a60be
#=> NIC statistics:
#=>     peer_ifindex: 11
{% endhighlight %}
```peer_ifindex``` 7 and 11 are indexes of the eth0 interfaces inside docker containers. Using this indexes you can find out interface name and configuration from ```ip link``` output.
Inside a container, this vethXXXX interfaces will have a container namespace link leg. We can use ```ip link``` tool to print out information:
{% highlight bash %}
docker ps  --format "table {% raw %}{{.ID}}{% endraw %}"
#=> CONTAINER ID
#=> 4d0e8ada345c
#=> eed0e7bd4f9d

docker exec -ti 4d0e8ada345c bash -c "ip link"
#=> 11: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
{% endhighlight %}
We can check with ethtool interface index in the root namespace:
{% highlight bash %}
root@dcb084af2fbb:/# ethtool -S eth0
#=> NIC statistics:
#=>     peer_ifindex: 12
ip link show  veth78a60be
#=> 12: veth78a60be: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
{% endhighlight %}

## Exposing container namespaces

```ip tools``` allow us to execute ```ip``` command family under particular namespace. Docker by default does not expose it into /var/run/netns because directory because it is using own [libcontainer](https://github.com/opencontainers/runc/tree/master/libcontainer). Nevertheless we can do this creating a symlink from process network namespace to ```/var/run/netns``` directory. This directory should be created if does not exists.

{% highlight bash %}
docker inspect --format {% raw %}'{{.State.Pid}}'{% endraw %} eed0e7bd4f9d
#=> 7863
ln  -s /proc/7863/ns/net /var/run/netns/ns-7863
ip netns
#=> ns-7863
ip netns exec ns-7863 ip link
#=> 13: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

Having this information we can connect additional link inside the docker container:

{% highlight bash %}
ip link add eth0-ns-7863 type veth peer name veth-ns-7863
# create pair of interfaces veth-ns-7863 and eth0-ns-7863
ip link set eth0-ns-7863 netns ns-7863
# put eth0-ns-7863 inside network namespace 7863
ip netns exec ns-7863 ip link set dev eth0-ns-7863 down
ip netns exec ns-7863 ip link set dev eth0-ns-7863 name eth1
ip netns exec ns-7863 ip link set dev eth1 up
ip link set dev veth-ns-7863 up
# rename eth0-ns-7863 into eth1 and bring it up, together with veth-ns-7863
ip netns exec ns-7863 ip link 
# 16: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
{% endhighlight %}
Inside the container we can run ```ifconfig``` to check that we have actually built this link:

{% highlight bash %}
docker exec -ti  eed0e7bd4f9d ifconfig |grep eth
#=> eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02
#=> eth1      Link encap:Ethernet  HWaddr 92:cd:50:a0:19:53
{% endhighlight %}

## Container network interface

Container network interface [CNI](https://github.com/containernetworking/cni) is the approach that aims standardising of container networking. The main purpose of this project is to make container networking flexible, extensible and easy to use with different control and data plane implementations. Later I will show how easy is to use it. 

CNI has several plugin types. The most important of them are ```main``` and ```ipam``` (ip management). Main plugin does manipulations with network namespaces, e.g. create veth pair, linking it inside the container, connecting to a bridge and etc. IPAM plugin manages IP setting allocations.

If we look inside one of the scripts from here [https://github.com/containernetworking/cni](https://github.com/containernetworking/cni/blob/master/scripts/docker-run.sh#L8-L20) you will be surprised that it uses very similar principal that was described above.

We are starting docker container without network:
{% highlight bash %}
contid=$(docker run -d --net=none busybox:latest /bin/sleep 10000000)
{% endhighlight %}
Getting its PID:
{% highlight bash %}
pid=$(docker inspect -f {% raw %}'{{ .State.Pid }}'{% endraw %} $contid)
{% endhighlight %}
And figuring out the path of the namespace:
{% highlight bash %}
/proc/$pid/ns/net
{% endhighlight %}

Then we can execute our CNI plugin that will create an interface, assign IP address, mask and gateway. This all happens by means of golang library which is the part of CNI project [github.com/containernetworking/cni/pkg/ip](https://github.com/containernetworking/cni/tree/master/pkg/ip). Library itself uses [github.com/vishvananda/netlink](https://github.com/vishvananda/netlink), which is golang implementation of ```ip tools```.  

Finally the script connects network namespace of the first container to the new one, where all settings already configured and linked inside Root namespace in the way which is defined by particular CNI plugin:
{% highlight bash %}
docker run --net=container:$contid $@
{% endhighlight %}

## CNI configuration and binaries

CNI configuration is stored in ```/etc/cni/net.d/``` and you should name your files with preceding priority (e.g. 10,20) and .conf (e.g. /etc/cni/net.d/10-mynet.conf) extension. Here is the example of the configuration format that I used for [BaGPipe CNI plugin](https://github.com/murat1985/bagpipe-cni): 
{% highlight json %}
{
  "name": "mynet",
  "type": "bagpipe",
  "importrt": "64512:90",
  "exportrt": "64512:90",
  "isGateway": false,
  "ipMasq": false,
  "mtu": "1500", 
    "ipam": {
        "type": "host-local",
        "subnet": "10.1.2.0/24",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ]
    }
}
{% endhighlight %}

If you use CNI plugins together with [Kubernetes](http://kubernetes.io/) binaries should be stored in ```/opt/cni/bin```.

Arguments are passed in the plugins as environment variables:

{% highlight bash %}
CNI_COMMAND
CNI_NETNS
CNI_IFNAME
CNI_PATH
{% endhighlight %}

```CNI_COMMAND``` - add or delete. This commands creates or deletes network interface inside the container with all necessary operations.
```CNI_NETNS``` - namespace path e.g. ```/proc/$PID/ns/net```.
```CNI_IFNAME``` - the naming of network interface inside the container (usually ```eth0```).
```CNI_PATH``` - path to CNI plugin binaries, for Kubernetes it will be ```/opt/cni/bin```.

Plugins code worked pretty well for me with [golang compiler 1.6](https://tip.golang.org/doc/go1.6).

Here you can find set of functions that can be used to create veth pair [ip/link.go](https://github.com/containernetworking/cni/blob/master/pkg/ip/link.go). For example:
{% highlight golang %}
func SetupVeth(contVethName string, mtu int, hostNS *os.File) (hostVeth, contVeth netlink.Link, err error)
{% endhighlight %}
This function creates pair of linked interfaces similar as we did above with veth-ns-7863 and eth0-ns-7863.

Function ```ns.WithNetNS``` from  this package ```github.com/containernetworking/cni/pkg/ns```
allows to execute some code inside container's namespace that was passed as an argument.

Having all this and examples from [CNI github repository](ttps://github.com/containernetworking/cni) we can easily implement our own plugin.

First of all we should define ```NetConf struct```
{% highlight golang %}
 type NetConf struct {
  types.NetConf
  ImportRT string `json:"importrt"`
  ExportRT string `json:"exportrt"`
  IsGW     bool   `json:"isGateway"`
  IPMasq   bool   `json:"ipMasq"`
  MTU      int    `json:"mtu"`
} 
{% endhighlight %}

types.netConf will include standard struct parameters and we can define our own. In this particular example we added ImportRT and ExportRT (Import and Export route target for out BGP routes).

Next we can implement methods based on the code from one of the plugins:

{% highlight golang %}
func setupVeth(netns string, ifName string, mtu int) (contMacAddr string, hostVethName string, err error) {
  err = ns.WithNetNSPath(netns, false, func(hostNS *os.File) error {
    // create the veth pair in the container and move host end into host netns
    hostVeth, _, err := ip.SetupVeth(ifName, mtu, hostNS)
    if err != nil {
      return nil
    }
    hostVethName = hostVeth.Attrs().Name
    interfaces, _ := net.Interfaces()
    // Lookup MAC address of eth0 inside namespace
    for _, inter := range interfaces {
      if inter.Name != "lo" {
        contMacAddr = fmt.Sprintf("%v", inter.HardwareAddr)
      }
    }
    return nil
  })
  return contMacAddr, hostVethName, err
}
{% endhighlight %}

In this particular example we wrote function that allows us to execute  ```net.Interfaces()``` under the given network namespace. It is analog of the command: ```ip netns exec $NS_NAME ip link```
Then we can extract Hardware Address (MAC address) and the name of the root namespace leg of newly generated veth pair.

That is it for today, in [Part 2](http://murat1985.github.io/kubernetes/cni/2016/05/15/bagpipe-gobgp.html) we will discuss two interesting projects [BaGPipe BGP](https://github.com/Orange-OpenSource/bagpipe-bgp) and [goBGP](https://github.com/osrg/gobgp). We will use them together with CNI to build Kubernetes networks.

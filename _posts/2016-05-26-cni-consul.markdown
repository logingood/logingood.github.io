---
layout: post
title:  "Flannel network explanation and example"
date:   2016-05-26 21:24:00 +1000
categories: kubernetes cni consul
---

Default overlay network for Kubernetes is [Flannel](https://github.com/coreos/flannel), which uses [etcd](https://github.com/coreos/etcd) as a distributed KV store to keep track on IP assignment information. Every pod is being assigned with a uniq ip address that is stored as a KV record.

{% include image.html url="/images/Overlay-explanation.png" description="Flannel network explanation" %}
<br>

The networking model proposed by flannel implies that all nodes participating in the overlay network are residing in the flat ip space with a wide mask, e.g. /16. It means that ```flannel0``` interfaces on every node are assigned with ip address from this long network space. 

For this particular example flannel network would be described as the following JSON struct:
{% highlight json %}
{
  "Network": "10.1.0.0/16",
  "SubnetLen": 24,
  "Backend": { 
    "Type": "vxlan",
    "VNI": 12345
  } 
}
{% endhighlight %}

Network can be created with the etcd cli command:
{% highlight bash %}
etcdctl mk /lab/network/config '{"Network":"10.1.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan","VNI":12345}}'
{% endhighlight %}

Backend seems to be confusing parameter name because actually it describes overlay transport type.

Every node is  aware about others existence and networks using etcd information:

{% highlight bash %}
etcdctl ls /lab/network --recursive

/lab/network/config
/lab/network/subnets
/lab/network/subnets/10.1.3.0-24
/lab/network/subnets/10.1.60.0-24
{% endhighlight %}

And if we get values from subnets ```etcdctl get /lab/network/subnets/10.1.3.0-24```

{% highlight json %}
{
  "PublicIP":"192.168.33.21",
  "BackendType":"vxlan",
  "BackendData":{
    "VtepMAC":"d6:d5:43:cc:30:2a"
  }
}
{% endhighlight %}

There is a convention for a flannel interface ip address assignment. For example flannel interface of a 16 bit network gets ip address ```x.y.z.0/16``` and ```docker0``` interface gets ip address ```x.y.z.1/24```.
Number of flannel interface depends on the overlay network type. For ```vxlan``` it would be ```flannel.VNI``` and for ```udp``` it would be ```flannel0```
{% highlight bash %}
4: flannel.12345: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
    link/ether d6:d5:43:cc:30:2a brd ff:ff:ff:ff:ff:ff
    inet 10.1.3.0/16 scope global flannel.12345
{% endhighlight %}


For [CNI](https://github.com/containernetworking/cni/tree/master/plugins/ipam) case, ip allocation manager plugins currently support dhcp and host-local plugin types. DHCP plugin basically implements dhcp servers functionality for network containers. In this case container's interface ```eth0``` gets ip address by dhcp. That is probably can be a feasible solution. However having a KV backed to store ip-address information and keeping track on allocations adds extra flexibility and visibility to our container networks.

Flannel already supports [multi-network overlay topologies](https://github.com/coreos/flannel#multi-network-mode-experimental) but this feature is marked as experimental. Also it is possible to run flannel in the [controller mode](https://github.com/coreos/flannel#multi-network-mode-experimental). They call it client-server mode and it allows to have a dedicated flannel controller which coordinates container IP advertisements between other flannel network participants.

However flannel network requires further extension or proper L3 routing control plain on top of it. This would allow building more scalable topologies like described in this [post](murat1985.github.io/kubernetes/cni/2016/05/15/kubernetes.html).

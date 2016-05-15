---
layout: post
title:  "Experiments with container networking: Part 2"
date:   2016-05-15 10:43:28 +1000
categories: kubernetes cni
---
# Open source EVPN implementation: BaGPipe BGP

As a part of our discussion about container networking, CNI and Kubernetes lets have a look to a very intersting project [BaGPipe BGP](https://github.com/Orange-OpenSource/bagpipe-bgp).

This tool allows you to create a BGP speaker on the Linux machine that can advertise IP VPN and EVPN BGP routes, together with forwarding plane configuration. Implemented on Python it uses [ExaBGP](https://github.com/Exa-Networks/exabgp) code for talking on BGP protocol and own python implemented libraries to manipulate Linux network namespaces, VXLAN networks and MPLS networks based on [OvS](http://openvswitch.org/).

One of the main peculiarities of this project is the REST API that could be used for BGP route advertisements. 

BaGPipe BGP daemon is listening by default on ```127.0.0.1:8082``` and accepts POST and GET methods with json payload.

JSON payload format is pretty simple and straight forward:
{% highlight golang %}
  // JSON payload struct for BaGPipe BGP
  type Message struct {
    Import_rt        []string         `json:"import_rt"`
    Vpn_type         string           `json:"vpn_type"`
    Vpn_instance_id  string           `json:"vpn_instance_id"`
    Ip_address       string           `json:"ip_address"`
    Export_rt        []string         `json:"export_rt"`
    Local_port       *json.RawMessage `json:"local_port"`
    Readvertise      string           `json:"readvertise"`
    Gateway_ip       string           `json:"gateway_ip"`
    Mac_address      string           `json:"mac_address"`
    Advertise_subnet bool             `json:"advertise_subnet"`
  }

  // Linuxif Raw JSON object
  type LcPorts struct {
    LinuxIf string `json:"linuxif"`
  }
{% endhighlight %}

Apparently that is very easy to use this message format by arbitary external tool, especially by a CNI plugin. 

This project is only able to establish BGP session but doesn't accept incoming connections. That is why we require a BGP router to connect together different BaGPipe BGP nodes. This router should act as a Route Reflector in order to re-advertise iBGP rotes. The great example of such a router is [GoBGP](https://github.com/osrg/gobgp). This lightweight, powerful and easy to configure BGP daemon is written on Go.

# Looking into container networking at DC scale

Having this great tools we can create a network design that allows us to scale container networking at inter-DC level. BGP protocol allows to build highly scalable networks and particular NLRI types allow you to advertise virtual networks which aligns with service-oriented model. 

Containers became popular due to the paradigm of micro-services architecture. This approach applies to the networking part as well. Carriers and service providers are already using this model to separate different types of traffic or traffic of different subscribers into different isolated flows (VPNs). For MPLS networks it is implemented as L3/L2 VPNs and every VPN can be treated as a service.
For example, LTE traffic of modern mobile networks is a separate service, 3GPP S1-5, X2 interfaces could be transmitted as another VPN services and etc.

For container-based networks this way of treating things works really well and we can come up with the following topology:

{% include image.html url="/images/cont-DC.png" description="Multi-datacenter container network routed with BGP" %}

Different container colours represent different services. They could be connected together even they reside in different datacenters. For Kubernetes networks it would be POD groups that are grouped based on [PODs services](http://kubernetes.io/docs/user-guide/services/).

The question is how can we implement this service isolation. The answer is simple - we can use the same approach as utilised for MPLS networks. [VXLAN DCI Using EVPN](https://tools.ietf.org/html/draft-boutros-l2vpn-vxlan-evpn-04) implementation would be helpful. For this purpose BGP protocol has reserved AFI=25(L2VPN) and SAFI=70 (EVPN) NLRI types. This type of BGP routes allows us to advertise our containers network with a forwarding plane implemented as [VXLAN](https://tools.ietf.org/html/rfc7348).

# Quick introduction into VXLAN

VXLAN was initially developed to address the VLAN limitation issue in datacenters that are using virtualization. Apparently length of VLAN tag is only 12 bits and allows us to have a little bit more than forty hundred unique networks. Some technologies, like Q-in-Q, can add an outer tag and tackle this issue but adds additional complications and works only in Layer 2 networks. VXLAN provides a networking model that uses transport layer UDP protocol and allows us to encapsulate full stack from L2-7 into UDP datagrams. That is pretty similar to L2TP but more simple.
Protocol is implemented using simple 8 bytes header with 24 bits addressing string that called VNI (VXLAN Network Identifier). This lightweight way of forwarding traffic lead to the popularity of this protocol and integration it into Linux kernel.

{% include image.html url="/images/vxlan.png" description="VXLAN datagram format" %}

Regardless that the forwarding plane seem simple, overlay nature of this technology requires appropriate signalling. Different approaches were utilised to solve this problem. The first ones implied to use PIM or network controller, e.g. VMware NSX, Cisco Nexus and etc. Open source implementation which are used in [Docker](https://docs.docker.com/engine/userguide/networking/dockernetworks/) or [Flannel](https://github.com/coreos/flannel) suggest using a Key-Value backend such as [etcd](https://github.com/coreos/etcd), [Consul](consul.io) or [Zookeeper](https://zookeeper.apache.org/). Apparently a Key-Value storage would be required in such kind of solutions anyway to spread information about IP allocation but this would work inside datacenter. For inter-DC communication that does not look scalable and convenient. And here we can benefit from BGP EVPN solution.
BaGPipe BGP perfectly fits for this purpose and that is why I decided to write [CNI plugin for it](https://github.com/murat1985/bagpipe-cni).

# Topology description: containers, BaGPipe and GoBGP as RR

I was using this guide [Ethernet VPN (EVPN)](https://github.com/osrg/gobgp/blob/master/docs/sources/evpn.md) to make initial configuration of BaGPipe BGP and GoBGP and came up with following topology:

{% include image.html url="/images/bagpipe-poc.png" description="Docker, BaGPipe and GoBGP" %}

Two nodes are running docker engine and BaGPipe BGP router with IP addresses 192.168.33.10 and 33.20 accordingly.

Configuration of BaGPipe BGP is very simple. We will use only VXLAN dataplane in this example. IP VPN will require OvS installation. For node 192.168.33.10 it will be as follows, for 33.20 you need to change ```local_address``` variable.

{% highlight toml %}
[BGP]
local_address=192.168.33.10
peers=192.168.33.30
my_as=64512
enable_rtc=True
[API]
api_host=localhost
api_port=8082
[DATAPLANE_DRIVER_IPVPN]
dataplane_driver = DummyDataplaneDriver
mpls_interface=eth1
ovsbr_interfaces_mtu=4000
[DATAPLANE_DRIVER_EVPN]
dataplane_driver = linux_vxlan.LinuxVXLANDataplaneDriver
ovsbr_interfaces_mtu=4000
{% endhighlight %}
```peers``` - GoBGP RR address that is running in separate virtual machine, you can run it as docker container as well. 
```API``` defines REST api server settings
```mpls_interface``` - important parameter if you are using Vagrant with virtual box. ```eth0``` is used to connect using ssh to virtual machine and always assigned 10.0.2.15 address by default. So we should use ```eth1```.

GoBGP configuration probably even more simple but you should define capabilities and RR clients. This node has IP address 192.168.33.30/24:

{% highlight toml %}
[global.config]
  as = 64512
  router-id = "192.168.33.30"

[[neighbors]]
[neighbors.config]
  neighbor-address = "192.168.33.10"
  peer-as = 64512
[neighbors.route-reflector.config]
  route-reflector-client = true
  route-reflector-cluster-id = "192.168.33.30"
[[neighbors.afi-safis]]
  afi-safi-name = "l2vpn-evpn"

[[neighbors]]
[neighbors.config]
  neighbor-address = "192.168.33.20"
  peer-as = 64512
[neighbors.route-reflector.config]
  route-reflector-client = true
  route-reflector-cluster-id = "192.168.33.30"
[[neighbors.afi-safis]]
  afi-safi-name = "l2vpn-evpn"
{% endhighlight %}

Here ```route-reflector-cluster-id``` - is the RR cluster id that is used in loop prevention for RR iBGP environments. This value could be equal to Router ID of your RR. AFI and SAFI capabilities should be specified as well ```afi-safi-name```. Autonomous system number is the same for every peers 64512 - we are using iBGP.

From BaGPipe BGP documentation it is apparent that it supports veth (namespaces) interfaces, hence we can use it together with LXC or Docker containers. To do this we should start containers on the nodes without network (```---net none```).

{% highlight bash %}
# Node1
contid=$(docker run -dti --net none --name test1 busybox)
# Node2
contid=$(docker run -dti --net none --name test2 busybox)
{% endhighlight %}

Then we should expose network namespace of this containers as we did in [Part 1](http://murat1985.github.io/kubernetes/cni/2016/05/14/netns-and-cni.html):

{% highlight bash %}
# Get PID of the container based on container id that we get:
pid=$(docker inspect -f '{{ .State.Pid }}' $contid)
# Expose network namespace to /var/run/netns
ln -s /proc/$pid/ns/net /var/run/netns/$pid
{% endhighlight %}

Now we can see namespaces on both nodes  using ```ip netns``` output:
{% highlight bash %}
ip netns
#=> 1234
{% endhighlight %}

Next we should attach network interface to this namespace using BaGPipe BGP attach command, on both nodes:

{% highlight bash %}
# Node 1
bagpipe-rest-attach --attach --port netns --ip 192.168.10.10 --network-type evpn --vpn-instance-id $pid --rt 64512:80
# Node 2
bagpipe-rest-attach --attach --port netns --ip 192.168.10.20 --network-type evpn --vpn-instance-id $pid --rt 64512:80
{% endhighlight %}

It will create for us an interface inside each container with the name ```tovpn``` and IP address that we specified in ```--ip``` argument.

Now we can test that everything works and containers can talk over the VXLAN network 192.168.10.0/24:

{% highlight bash %}
# Node 1
docker exec -ti test1 ping 192.168.10.20
# Node 2
docker exec -ti test1 ping 192.168.10.10
{% endhighlight %}

If we use wireshark we can see the traffic:

{% include image.html url="/images/tcpdump.png" description="VXLAN container traffic" %}

To make all this easier I wrote a [patch for BaGPipe BGP](https://github.com/murat1985/bagpipe-bgp/commit/ec412d59c6988831b0aa909267d55169c49c5460). It allows us to specify container ID (```--docker-container-id test1```) to bagpipe-rest-attach and bagpipe-rest-detach commands:

{% highlight bash %}
root@dockerbgp1:~# bagpipe-rest-attach --attach --port netns --ip 192.168.10.10 --network-type evpn --docker-container-id test1 --rt 64512:80
# => using 192.168.10.254 as gateway address
#=> Will plug local namespace 2109 into network
#=> Local port: tons-2109 (96:d9:4c:f6:8d:8b)
#=> request: {"import_rt": ["64512:80"], "vpn_type": "evpn", "docker_cintainer_id": "test1", "vpn_instance_id": "2109", "ip_address": "192.168.10.10/24", "export_rt": ["64512:80"], "local_port": {"linuxif": "tons-2109"}, "advertise_subnet": false, "gateway_ip": "192.168.10.254", "mac_address": "96:d9:4c:f6:8d:8b", "readvertise": null}
#=> response: 200
{% endhighlight %}

You can find more detailed description in my [fork of BaGPipe BGP](https://github.com/murat1985/bagpipe-bgp#an-e-vpn-docker-containers-example). However it is better to use [CNI plugin](https://github.com/murat1985/bagpipe-cni) for this purpose and talk to BaGPipe BGP using [REST API](https://github.com/Orange-OpenSource/bagpipe-bgp#rest-api-tool-for-interface-attachments).

In Part 3 we will create a proof-of-concept lab with BaGPipe BGP CNI plugin and Kubernetes cluster.

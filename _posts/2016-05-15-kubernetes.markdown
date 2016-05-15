---
layout: post
title:  "Experiments with container networking: Part 3"
date:   2016-05-15 17:24:00 +1000
categories: kubernetes cni
---

# Kubernetes with BaGPipe BGP and CNI

The third part of the discussion about CNI, Kubernetes and EVPN BGP brings us to the solution that is shown on a topology below. We want our solution to stitch a Kubernetes orchestrated datacenter and provide us with seamless inter and intra DC communication between pods. All this should be implemented with regards to service-oriented model.

{% include image.html url="/images/kubernetes-dc-bgp.png" description="multi-datacenter kubernetes network routed with bgp" %}

To show that we can deliver this design we will prepare a proof-of-concept environment. To do this we require a way to integrate the BaGPipe BGP daemon into Kubernetes environment. Luckily [kubelet](http://kubernetes.io/docs/admin/kubelet/) (a process that is responsible for managing pods on the Kubernetes node) supports CNI interface. 

# How CNI plugin talks to BaGPipe BGP

First of all let's understand how CNI can talk to BaGPipe BGP daemon. Fortunately BaGPipe BGP project implemented REST API that we can use from our CNI plugin as it was discussed in the [Part 2](http://murat1985.github.io/kubernetes/cni/2016/05/15/bagpipe-gobgp.html).

There is a JSON structure that BaGPipe BGP expects to receive. Sending this kind of message allows us to avoid interaction with ```bagpipe-rest-attach``` command-line tool.

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

```Import_rt``` - import route-target community string e.g. 65000:123 (ASN:Number)
```Export_rt``` - export route-target community string e.g. 65000:123 (ASN:Number)

Route targets are used to control ```route import and export``` policy in VRF. For those who are curious BaGPipe BGP uses RouterID:Number as a Route Distinguisher attribute (e.g. 192.168.33.22:5). It is [Type 1](https://tools.ietf.org/html/rfc4364#section-4.2) RD and chosen according to [RFC7432](https://tools.ietf.org/html/rfc7432#section-7.9) recommendation. 

```Vpn_type``` - CNI plugin that I wrote supports only ```evpn``` by this time.

```Vpn_instance_id``` - unique identifier for an instance id. I put a name of veth interface.

```Ip_address``` - ip address that would be assigned to pod or container.

```Local_port``` - the same value as vpn instance id but in nested json format.

```Gateway_ip``` - ip address of the default gateway that will be assigned to pod or container

```Mac_address``` - of the interface inside container's namespace

```Advertise_subnet``` - Optional attribute if we set it to True then VRF will advertise whole subnet, by default only /32 advertised

```Readvertise``` - used to readvertise subnets received from route targets. 

I implemented a function inside our plugin to send BaGPipe BGP JSON message. It was pretty easy to do as Go has a library to communicate on HTTP protocol ```net/http```.

{% highlight go %}
func sendBagpipeReq(n *NetConf, Request string, LocalPort string, gw string, ipa string, mac string) error {
  // create an URL
  url := fmt.Sprintf("http://127.0.0.1:8082/%s_localport", Request)

  var jsonStr []byte
  // create JSON object for BagpipeBGP
  jsonStr, err := createBGPConf(n, LocalPort, gw, ipa, mac)
  // create a new HTTP request POST
  req, err := http.NewRequest("POST", url, bytes.NewBuffer(jsonStr))

  // set HTTP headers - contnet-type application/json
  req.Header.Set("Content-Type", "application/json")
  client := &http.Client{}
  // send HTTP POST request to BaGPipe BGP server
  resp, err := client.Do(req)

  if err != nil {
    panic(err)
  }

  defer resp.Body.Close()
  return err
}
{% endhighlight %}

This method probably requires certain improvement because we should handle properly response from BGP speaker. I think for this proof-of-concept lab it will be suitable though.

Our plugin has two functions that will implement adding and deleting network interface of the pod:

{% highlight golang %}
func cmdAdd(args *skel.CmdArgs) error {
  // here is some code that creates namespace
  if macAddr, vethName, err = setupVeth(args.Netns, args.IfName, n.MTU); err != nil {
    return err
  }
  // run the IPAM plugin and get back the config to apply
  result, err := ipam.ExecAdd(n.IPAM.Type, args.StdinData)
  // some code part that we will skip for simplicity ...  
  // getting ip and gateway addresses
  ip_gw = fmt.Sprintf("%v", &result.IP4.Gateway)
  ip_addr = fmt.Sprintf("%v", &result.IP4.IP)
  // sending this to BaGPipe BGP with "attach" flag
  sendBagpipeReq(n, "attach", vethName, ip_gw, ip_addr, macAddr)

}
{% endhighlight %}

Function ```cmdDel``` implemented as well:  please refer to [BaGPipe CNI plugin implementation](https://github.com/murat1985/bagpipe-cni/blob/master/bagpipe.go#L204). To delete interface we require to enter inside pod namespace and delete veth link from there. Also we need to form JSON message and send it to BaGPipe REST API with ```detach``` command.


I encountered certain difficulties in implementing cmdDel procedure because BaGPipe BGP requires a local interface name to be passed as an argument (which basicly is the interface belonging to root namespace, e.g. vethXXXXX) . Nevertheless thankfully to [ethtool go package](https://github.com/safchain/ethtool) I managed to get the *root namespace* leg index and then pass root namespace veth leg name to ```sendBagpipeReq``` function.


{% highlight golang %}
    stats, _ := ethtool.Stats(args.IfName)
    // get the index of the opposite end of the link between namespaces
    // similar to ethtool -S command
    if_index = stats["peer_ifindex"]
{% endhighlight %}

Diagram below is showing logical flow:

{% include image.html url="/images/CNI-Bagpipe.png" description="multi-datacenter kubernetes network routed with bgp" %}

CNI plugin receives configuration based on environment variables ```CNI_*``` from external application that executes it. For Kubernetes it would be kubelet. Bare in mind that configuration is taken from configuration file which stored at /etc/cni/net.d/*.conf. Example of configuration file can be found in [Part 1](http://murat1985.github.io/kubernetes/cni/2016/05/14/netns-and-cni.html). Then plugin creates veth interface pair, assigns IP address using ```ipam``` plugin and sends ```attach``` command to BaGPipe BGP daemon.

To understand better EVPN BGP NLRI format please find below a tcpdump (wireshark) snippet with explanation.

{% include image.html url="/images/BGP-EVPN-traffic.png" description="multi-datacenter kubernetes network routed with bgp" %}

# Proof of concept BaGPipe BGP, CNI plugin and Kubernetes

Now we can bring all things together and show CNI BaGPipe plugin in action. 
To implement PoC lab we will use two Kubernetes nodes running BaGPipe BGP, BaGPipe CNI plugin, kubelet, kube-proxy and docker. And one master node that is running kube-apiserver,kube-controller-manager,kube-scheduler and etcd. 

{% include image.html url="/images/POC-CNI-Kubernetes-BaGPipe.png" description="BaGPipe BGP CNI and Kubernetes Proof of Concept" %}

In kubelet configuration we should specify CNI plugin as a network plugin. It could be done in the ```/etc/kubernetes/kubelet``` file. Commands below should be executed on both Kubernetes nodes.

{% highlight bash %}
KUBELET_ARGS="--network-plugin=cni --network-plugin-dir=/etc/cni/net.d"
{% endhighlight %}

Next we should clone CNI github repository to ```/opt``` directory:
{% highlight bash %}
git clone https://github.com/containernetworking/cni /opt/cni
{% endhighlight %}

Now we should clone [BaGPipe CNI plugin](https://github.com/murat1985/bagpipe-cni) into ```/opt/cni/plugins/main/bagpipe``` directory. Also we require to download ethtool go package.

{% highlight bash %}
git clone https://github.com/murat1985/bagpipe-cni /opt/cni/plugins/main/bagpipe
# Also we need
go get github.com/safchain/ethtool
{% endhighlight %}

Then we should go to /opt/cni and build our plugins:
{% highlight bash %}
cd /opt/cni/
./build
{% endhighlight %}

Drop configuration into /etc/cni/net.d/10-bagpipe.conf and configure network. For this example we will use ```host-local``` ipam plugin with different IP ranges. Own CNI BaGPipe ipam plugin is in TODO list. Nevertheless, we can use ```range-start``` and ```range-end``` to avoid pod IP addresses overlapping.

Node 1:

{% highlight json %}
{
    "name": "bagpipe-net",
    "type": "bagpipe",
    "importrt": "64512:90",
    "exportrt": "64512:90",
    "mtu": 1500,
    "isGateway": false,
    "ipMasq": false,
    "ipam": {
        "type": "host-local",
        "range-start": "10.27.2.0",
        "range-end": "10.27.3.0",
        "subnet": "10.27.0.0/16"
    }
}
{% endhighlight %}

Node 2:

{% highlight json %}
{
    "name": "bagpipe-net",
    "type": "bagpipe",
    "importrt": "64512:90",
    "exportrt": "64512:90",
    "mtu": 1500,
    "isGateway": false,
    "ipMasq": false,
    "ipam": {
        "type": "host-local",
        "range-start": "10.27.3.1",
        "range-end": "10.27.4.0",
        "subnet": "10.27.0.0/16"
    }
}
{% endhighlight %}

Next we should setup BaGPipe BGP and start it:

Node 1:
{% highlight toml %}
[BGP]
local_address=192.168.33.21
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

Node 2:

{% highlight toml %}
[BGP]
local_address=192.168.33.21
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

We will use the GoBGP Route Reflector from [Part 2](http://murat1985.github.io/kubernetes/cni/2016/05/15/bagpipe-gobgp.html)

After configuration completed, restarting kubelet service on both nodes.

{% highlight bash %}
systemctl restart kubelet
{% endhighlight %}

Finally from master node we are running our pods. 

{% highlight bash %}
kubectl run test1 --image=nginx --replicas=2 --port=80
kubectl run test2 --image=nginx --replicas=2 --port=80
{% endhighlight %}

Now we can do experiment and ping from one of the Kubernetes nodes another one. Let's execute ping inside a docker container that is running in a pod. In this example from node 10.27.1.10 we are pinging node 10.27.0.124.

{% highlight bash %}
[root@node2 ~]# docker exec -ti bf51e589ba09 bash -c "ip addr|grep 10.27"
    inet 10.27.1.10/16 scope global eth0
[root@node2 ~]# docker exec -ti bf51e589ba09 ping 10.27.0.124

[root@node1 ~]# docker ps |grep test2 | awk ' { print $1 } '
106fe87b10cd
7fe4b4b84167
[root@node1 ~]# docker exec -ti 106fe87b10cd bash -c "ip addr |grep 10.27"
    inet 10.27.0.124/16 scope global eth0
{% endhighlight %}

And we are again using tcpdump to catch this traffic and look inside UDP datagram. As you can see 10.27.0.0/16 network ICMP traffic is encapsulated inside VXLAN UDP datagrams.

{% highlight bash %}
tcpdump -w /opt/mounts/kube.pcap -s0 -A -ni eth1
{% endhighlight %}

{% include image.html url="/images/kube-pcap.png" description="Wireshark intercept of Kubernetes VXLAN with BaGPipe BGP" %}

All in all we have shown that it is possible to use [BaGPipe BGP EVPN](https://github.com/Orange-OpenSource/bagpipe-bgp#an-e-vpn-example) implementation together with Kubernetes due to exceptional flexibility of [CNI interface](https://github.com/containernetworking/cni). We used in this PoC my own implementation of CNI plugin that I named [BaGPipe CNI](https://github.com/murat1985/bagpipe-cni). Anyone who is intersted in contribution or have questions about this PoC always welcome on email or pull request.

[Part 1](http://murat1985.github.io/kubernetes/cni/2016/05/14/netns-and-cni.html) and [Part 2](http://murat1985.github.io/kubernetes/cni/2016/05/15/bagpipe-gobgp.html).

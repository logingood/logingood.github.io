---
layout: post
title:  "Using Consul as IPAM backend for CNI plugins"
date:   2016-07-15 02:41:00 +1000
categories: kubernetes cni consul
---
# Using Consul as IPAM backend for CNI plugins

In one of the previous [posts](http://logingood.github.io/kubernetes/cni/2016/05/15/kubernetes.html) we described PoC for Kubernetes network with [BaGPipe BGP CNI plugin](https://github.com/logingood/bagpipe-cni). However we used local IP allocator and storage that comes with [CNI](https://github.com/containernetworking/cni/tree/master/plugins/ipam/host-local) basic plugins bundle. Apparently that is not useful for distributed multi-node environment. To address this issue I made an effort to create a proof of concept that uses [Consul](https://www.consul.io) backend to store IP allocations. Also here you can find examples how to use [Go Consul API](https://godoc.org/github.com/hashicorp/consul/api) - initialisation, putting values, creating session and locking. 

{% include image.html url="/images/Consul-POC-CNI-Kube.png" description="Consul with CNI PoC" %}
<br>

As we discussed previously there are two types of plugins for CNI: 

* main
* ipam

BaGPipe CNI plugin is the example of main plugins, that are used to manipulate with namespaces and create a forwarding plane for container's network. IPAM is a ip manager plugins group that defines the way how IP settings being allocated, e.g. [DHCP](https://github.com/containernetworking/cni/tree/master/plugins/ipam/dhcp), [Host-Local](https://github.com/containernetworking/cni/tree/master/plugins/ipam/host-local) and etc.

## Introduction
Looking into the structure of host-local plugin that comes with [CNI](https://github.com/containernetworking/cni) we can understand the basic IPAM plugin. There are allocator, storage and plugin itself. So far they are bonded together and there is some [work](https://github.com/containernetworking/cni/issues/116) is ongoing to decouple these layers.

Hence to implement Consul backend we should implement interface that is described [here](https://github.com/containernetworking/cni/blob/master/plugins/ipam/host-local/backend/store.go#L19-L27)
{% highlight go %}
  Lock() error
  Unlock() error
  Close() error
  Reserve(id string, ip net.IP) (bool, error)
  LastReservedIP() (net.IP, error)
  Release(ip net.IP) error
  ReleaseByID(id string) error
{% endhighlight %}

That is comparatively easy job, as Consul has pretty well documented [Go API](https://godoc.org/github.com/hashicorp/consul/api).

Also given interface greatly aligns with Consul [distributed lock conception](https://www.consul.io/docs/commands/lock.html), which is why this particular KV storage was chosen even [etcd](https://github.com/coreos/etcd) might seem more suitable as it's required for Kubernetes.

Hence to implement our backend we need the following:
* Define structure for IP allocations (we can steal it from [Flannel](https://github.com/coreos/flannel)
* Implement Lock (create a session in Consul, lock KV path of our network)
* Implement Unlock (release lock, destroy session)
* Close - we wouldn't implement and create a stub to satisfy interface
* Reserve - allocate IP address
* Find last reserved (find lastly allocated IP address)
* Release (by IP) and ReleaseByID (by Container ID)

## Structure of key paths and value

To keep track on ip allocations we should store following information:

* IP address
* MAC (TBD)
* Container ID
* Timestamp

Example of the json payload that would be stored in Consul as a value of our key:

{% highlight json %}
{
  "ip":"10.22.0.2",
  "mac":"00:00:00:00:00:00",
  "id":"59500ba30e65841846398ca11d8b6c0863a7c66a9a51b10a33a3308dce919d38",
  "timestamp":1468560614
}
{%  endhighlight %}

As I mentioned above key path we are going to steal from Flannel, e.g.:

{% highlight bash %}
curl http://127.0.0.1:8500/v1/kv/10.22.0.0-IloveConsul?pretty
[
    {
        "LockIndex": 2,
        "Key": "10.22.0.0-IloveConsul",
        "Flags": 0,
        "Value": "eyJndyI6IiIsIm5ldCI6IjEwLjIyLjAuMC8xNiIsInN0YXJ0IjoiIiwiZW5kIjoiIiwicm91dGVzIjpbeyJkc3QiOiIwLjAuMC4wLzAifV19",
        "CreateIndex": 161,
        "ModifyIndex": 171
    }
]
{%  endhighlight %}

Pro tip: Consul has very neat and nice web interface:

{% include image.html url="/images/iloveconsul.png" description="CNI network example stored in Consul" %}

## Implementation

As we mentioned above we should implement interface from store.go. To get this done we should implement certain Consul functions. I wouldn't describe in detail all the steps (you can find actual implementations in my [CNI fork](https://github.com/logingood/cni/tree/master/plugins/ipam/store/consul) and standalone plugin which doesn't implement `LastReservedIP`, consul settings are stored in the network config file and it doesn't keep track on timestamps [cni-ipam-consul](https://github.com/logingood/cni-ipam-consul)). Instead we will focus on Consul API and locking.

First of all we have a crucial condition. As we are using consul we should find a way to pass settings of our backend store inside:

* Consul address
* Consul port
* DC name

To address this issue we can use a strategy which is recommended and install Consul on agent on every node. In this case we can benefit from service discovery as well. However for Kubernetes environment it doesn't make sense as we are already using ETCD, moreover hardcoding values is not the best idea to go with. As these values must be configurable our backend should be able to get the [IPAMConfig](https://github.com/containernetworking/cni/blob/master/plugins/ipam/host-local/config.go#L26-L35) struct. 

However we don't want to keep this settings in the struct in case we will use other plugins. So that could be implemented using environment variables that [CNI now supports](https://github.com/containernetworking/cni/blob/master/pkg/types/args.go#L74-L96) along with `isIgnoreUnknown`.
I have implemented both approaches: in cni-ipam-consul these values are in configuration file, e.g.
{% highlight json %}
{
  "name": "ipv4",
    "ipam": {
        "type": "consul",
        "consul_addr": "127.0.0.1",
        "consul_port": "8500",
        "dc": "dc1",
        "...": "..."
    }
}
{%  endhighlight %}

End in the [cni repository fork](https://github.com/logingood/cni) I used environment variables in the following way:

{% highlight bash %}
CNI_ARGS="StoreAddr=127.0.0.1;StorePort=8500;StoreNs=dc1;IgnoreUnknown=1"
{% endhighlight %}

Both ways will work way. CNI_ARGS seems more flexible in this case and wouldn't interfere with multiple backend types.

## Initialization and locking

To initialize the store we are using something like below:

{% highlight go %}
func ConnectStore(Addr string, Port string, DC string) (consul *api.Client, err error) {
        config := api.DefaultConfig()
        config.Address = fmt.Sprintf("%s:%s", Addr, Port)
        config.Datacenter = fmt.Sprintf("%s", DC)
        //config.Scheme = fmt.Sprintf("%s", Scheme)
        consul, err = api.NewClient(config)
        if err != nil {
                panic(err)
        }
        return consul, err
}
{% endhighlight %}

Function `New` from host-local CNI plugin should modified respectively:

{% highlight go %}
func New(n *sequential.IPAMConfig) (*Store, error) {
        // creating new consul connection

        addr := fmt.Sprintf("%s", n.Args.StoreAddr)
        port := fmt.Sprintf("%s", n.Args.StorePort)
        dc := fmt.Sprintf("%s", n.Args.StoreNS)

        consul, err := ConnectStore(addr, port, dc)
        if err != nil {
                panic(err)
        }
        network, err := NetConfigJson(n)
        key, err := InitStore(n.Name, network, consul)
        // write values in Store object
        store := &Store{
                Consul: consul,
                Key:    key,
        }
        return store, nil
}
{% endhighlight %}

Values `addr`,`port` and `dc` we are passing as configuration using `IPAMConfig` struct. That is why we require to include this somehow. 

To implement lock we again using consul api (you can use HTTP API as well):
{% highlight go %}
func (s *Store) Lock() error {

        Session := s.Consul.Session()
        kv := s.Consul.KV()
        var entry *api.SessionEntry

        // create session
        id, _, err := Session.Create(entry, nil)
        if err != nil {
                panic(err)
        }
        // get pair object from consul
        pair, _, err := kv.Get(s.Key, nil)
        pair.Session = id
        if err != nil {
                panic(err)
        }
        // acquire is false
        acq := false
        attempts := 0
        // will try 10 times to get the lock - 10 seconds
        for acq != true {
                if attempts == 10 {
                        panic("Wasn't able to acquire the lock in 10 seconds")
                }
                acq, _, err = kv.Acquire(pair, nil)
                if err != nil {
                        panic(err)
                }
                attempts += 1
                time.Sleep(1000 * time.Millisecond)
        }
        return err
}
{% endhighlight %}

Here we are guarding ourselves and trying to lock sessions several times in case some other node locked current network. These value better to keep configurable, however we hard-coded it in this particular example.

Putting and Getting values easy job:
{% highlight go %}
func PutKV(k string, val []byte, kv *api.KV) (k_store string, err error) {
        // put key-value pair
        d := &api.KVPair{Key: k, Value: val}
        _, err = kv.Put(d, nil)
        if err != nil {
                return k, err
        } else {
                k_store = k
                return k, nil
        }
}

func GetKV(k string, kv *api.KV) (list api.KVPairs, err error) {
        // get key list
        list, _, err = kv.List(k, nil)
        if err != nil {
                panic(err)
        }
        return list, err
}
{% endhighlight %}

## CNI specific functions

To reserve the lease we are using Reserve function from interface as follows. We are creating KV path if doesn't exist and filling it with `Lease` function.

{% highlight go %}
func (s *Store) Reserve(id string, ip net.IP) (bool, error) {
        // get consul KV
        kv := s.Consul.KV()
        // create path
        path := s.Key + "/" + fmt.Sprintf("%s", ip)
        pair, _ := GetKV(path, kv)
        // if key exists return false
        if len(pair) != 0 {
                return false, nil
        }
        // otherwise create a byte object and put
        b, _ := LeaseJson(ip, id)
        PutKV(path, b, kv)
        return true, nil
}
{% endhighlight %}

Another function that should stick attention is `LastReservedIP`. I decided to use Timestamps as a method to track last reserved IP addresses. We are referring to timestamp from the object and checking it against current time:

{% highlight go %}
func (s *Store) LastReservedIP() (net.IP, error) {
        kv := s.Consul.KV()
        pairs, _ := GetKV(s.Key, kv)

        var lease Lease
        var latest_ip string
        var prev_TS int64

        prev_TS = 0

        for _, pair := range pairs {
                if err := json.Unmarshal(pair.Value, &lease); err != nil {
                        return nil, err
                }
                if lease.Timestamp > prev_TS {
                        latest_ip = pair.Key
                }
        }
        return net.ParseIP(latest_ip), nil
}
{% endhighlight %}

## Using together with BaGPipe CNI plugin and Kubernetes

To use Consul backend you should install it on one of the nodes or you can go with Consul distributed installation when every node has Consul agent installed. Consul installation is very easy process and `-dev` mode is amazing thing for testing purpose. You should just download binary from [Consul official web site](https://www.consul.io/downloads.html).

Next start you store in `-dev` mode. 
```
consul agent -dev --bind IP
```
and you are done with Consul. 

To use standalone plugin you should download it and build using `go get` command:
```
go get github.com/logingood/cni-ipam-consul
```

Don't forget to specify as `GOBIN` path `/opt/cni/bin`, e.g.
```
export GOBIN=/opt/cni/bin
```

Please refer to the article [Kubernetes with BaGPipe BGP and CNI](http://logingood.github.io/kubernetes/cni/2016/05/15/kubernetes.html) for further information.

You configuration file should be smth like below. You should specify `consul_addr`,`consul_port` and `dc`, as well as put `ipam` type as `consul`.

```
{
    "name": "bagpipe-net",
    "type": "bagpipe",
    "importrt": "64512:90",
    "exportrt": "64512:90",
    "mtu": 1500,
    "isGateway": false,
    "ipMasq": false,
    "consul_addr": "192.168.33.30",
    "consul_port": "8500",
    "dc": "dc1",
    "ipam": {
        "type": "consul",
        "range-start": "10.27.3.1",
        "range-end": "10.27.4.0",
        "subnet": "10.27.0.0/16"
    }
}
```

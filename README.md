# Summary

A multi-vm `Vagrantfile` for setting up a k3s cluster using metallb in BGP mode, running a service preserving source IP.
Addapted from this post https://medium.com/@toja/running-k3s-with-metallb-on-vagrant-bd9603a5113b
Changes:
1) Updated to the latest kube-flannel.yml and made the modification to that as described below.
2) Updated to the latest version of metallb.yaml https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml


# Usage

```sh
vagrant up
```
After this completes you can ssh into the client and test the setup

```sh
vagrant ssh client
% curl http://192.168.90.192

should return a valid html page with a 200
vagrant@client:~$ curl -v http://192.168.90.192
* Rebuilt URL to: http://192.168.90.192/
*   Trying 192.168.90.192...
* TCP_NODELAY set
* Connected to 192.168.90.192 (192.168.90.192) port 80 (#0)
> GET / HTTP/1.1
> Host: 192.168.90.192
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.10.0
< Date: Tue, 26 Nov 2019 22:12:02 GMT
< Content-Type: text/plain
< Transfer-Encoding: chunked
< Connection: keep-alive
<
CLIENT VALUES:
client_address=10.42.0.1
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://192.168.90.192:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=192.168.90.192
user-agent=curl/7.58.0
BODY:
* Connection #0 to host 192.168.90.192 left intact
```


# Description

The setup is composed of 5 vms and two private networks.

## client

The client vm is used only to query the service from another network. To test the routing setup by BGP.

* IPs: 
  - 192.168.90.10

# router

The `router` vm connects to 3 networks, the `public_network` which might be used to access the vms from the host.

> The `bridge` name should match the interface being used to access internet on the host.

This vm provides connection between the two private networks: 

* k3s-metallb-net 192.168.33.0/24 Used by the nodes as ClusterIPs and by the router on the interface listening where's listening the bgp service.
* client-metallb-net 192.168.90.0/24  Used the `client` vm.

* IPs: 
  - 192.168.90.1
  - 192.168.33.1
  - 192.168.1.200

# flannel setup

Out of the box flannel will not work with vagrant, neither with k3s.

The available `flannel.yml` file requires two modifications to work on this setup:

1. To specify the interface to be used by flannel: `--iface=enp0s8`
2. To replace the default k8s network block (`10.244.0.0/16`) by the one used by k3s: `10.42.0.0/16`

The `kube-flannel.yml` file is provided with the changes required to make k3s work with Vagrant.

The first interface on a vm is on the network which allows vagrant to communicate with and to provide NAT traffic to the guest.


# Troubleshooting

When looking at the logs from k3s, `kubetail` proved to be an interesting tool.

```
vagrant ssh master
```

```sh
curl -sO https://raw.githubusercontent.com/johanhaleby/kubetail/master/kubetail ; chmod 755 kubetail ; sudo mv kubetail /usr/local/bin/
kubetail -l component=speaker -n metallb-system
```

Check the pods from metallb

```
vagrant@node0:~$ kubectl get po -o wide -n metallb-system
```

The speakers pods should be using an IP from the  `192.168.33.x/24` network.


Checking BGP routes on the bird daemon.

```
vagrant ssh router
```

```
sudo birdc
show protocols all
```

Check for `connections refused`, that points to networking issues, double-check all ips are assigned as expected.

Inspecting http traffic

```
tcpdump -i any -s 0 'tcp port http' 
```


# cleanup

```sh
vagrant destroy -f
```


## References:
* https://medium.com/@toja/running-k3s-with-metallb-on-vagrant-bd9603a5113b
* https://qiita.com/omatsu32/items/20b337b96595b6320be3
* https://metallb.universe.tf/usage/#local-traffic-policy-1
* https://kubernetes.io/docs/tutorials/services/source-ip/

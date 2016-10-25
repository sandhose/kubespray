HA endpoints for K8s
====================

The following components require a highly available endpoints:
* etcd cluster,
* kube-apiserver service instances.

The former provides the
[etcd-proxy](https://coreos.com/etcd/docs/latest/proxy.html) service to access
the cluster members in HA fashion.

The latter relies on a 3rd side reverse proxies, like Nginx or HAProxy, to
achieve the same goal.

Etcd
----

Etcd proxies are deployed on each node in the `k8s-cluster` group. A proxy is
a separate etcd process. It has a `localhost:2379` frontend and all of the etcd
cluster members as backends. Note that the `access_ip` is used as the backend
IP, if specified. Frontend endpoints cannot be accessed externally as they are
bound to a localhost only.

The `etcd_access_endpoint` fact provides an access pattern for clients. And the
`etcd_multiaccess` (defaults to `false`) group var controlls that behavior.
When enabled, it makes deployed components to access the etcd cluster members
directly: `http://ip1:2379, http://ip2:2379,...`. This mode assumes the clients
do a loadbalancing and handle HA for connections. Note, a pod definition of a
flannel networking plugin always uses a single `--etcd-server` endpoint!


Kube-apiserver
--------------

K8s components require a loadbalancer to access the apiservers via a reverse
proxy. Kargo includes support for an nginx-based proxy that resides on each
non-master Kubernetes node. This is referred to as localhost loadbalancing. It
is less efficient than a dedicated load balancer because it creates extra
health checks on the Kubernetes apiserver, but is more practical for scenarios
where an external LB or virtual IP management is inconvenient.

This option is configured by the variable `loadbalancer_apiserver_localhost`.
you will need to configure your own loadbalancer to achieve HA. Note that
deploying a loadbalancer is up to a user and is not covered by ansible roles
in Kargo. By default, it only configures a non-HA endpoint, which points to
the `access_ip` or IP address of the first server node in the `kube-master`
group. It can also configure clients to use endpoints for a given loadbalancer
type. The following diagram shows how traffic to the apiserver is directed.

![Image](figures/loadbalancer_localhost.png?raw=true)

  Note: Kubernetes master nodes still use insecure localhost access because
  there are bugs in Kubernetes <1.5.0 in using TLS auth on master role
  services. This makes backends receiving unencrypted traffic and may be a
  security issue when interconnecting different nodes, or maybe not, if those
  belong to the isolated management network without external access.

A user may opt to use an external loadbalancer (LB) instead. An external LB
provides access for external clients, while the internal LB accepts client
connections only to the localhost, similarly to the etcd-proxy HA endpoints.
Given a frontend `VIP` address and `IP1, IP2` addresses of backends, here is
an example configuration for a HAProxy service acting as an external LB:
```
listen kubernetes-apiserver-https
  bind <VIP>:8383
  option ssl-hello-chk
  mode tcp
  timeout client 3h
  timeout server 3h
  server master1 <IP1>:443
  server master2 <IP2>:443
  balance roundrobin
```

And the corresponding example global vars config:
```
apiserver_loadbalancer_domain_name: "lb-apiserver.kubernetes.local"
loadbalancer_apiserver:
  address: <VIP>
  port: 8383
```

This domain name, or default "lb-apiserver.kubernetes.local", will be inserted
into the `/etc/hosts` file of all servers in the `k8s-cluster` group. Note that
the HAProxy service should as well be HA and requires a VIP management, which
is out of scope of this doc. Specifying an external LB overrides any internal
localhost LB configuration.

  Note: In order to achieve HA for HAProxy instances, those must be running on
  the each node in the `k8s-cluster` group as well, but require no VIP, thus
  no VIP management.

Access endpoints are evaluated automagically, as the following:

| Endpoint type                | kube-master   | non-master          |
|------------------------------|---------------|---------------------|
| Local LB                     | http://lc:p   | https://lc:sp       |
| External LB, no internal     | https://lb:lp | https://lb:lp       |
| No ext/int LB (default)      | http://lc:p   | https://m[0].aip:sp |

Where:
* `m[0]` - the first node in the `kube-master` group;
* `lb` - LB FQDN, `apiserver_loadbalancer_domain_name`;
* `lc` - localhost;
* `p` - insecure port, `kube_apiserver_insecure_port`
* `sp` - secure port, `kube_apiserver_port`;
* `lp` - LB port, `loadbalancer_apiserver.port`, defers to the secure port;
* `ip` - the node IP, defers to the ansible IP;
* `aip` - `access_ip`, defers to the ip.
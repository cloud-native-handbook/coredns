# 《云原生指南之 CoreDNS》

CoreDNS 是一个 DNS Server，

## 概念

* Server Block
* Plugin Block

## 存根域和上游 DNS

> https://coredns.io/plugins/kubernetes/#stubdomains-and-upstreamnameservers

```
.:53 {
    kubernetes cluster.local {
        cidrs 172.254.0.0/16
        upstream
    }
    proxy cloud.local 192.168.10.102:53
    proxy . 8.8.8.8:53
}
```

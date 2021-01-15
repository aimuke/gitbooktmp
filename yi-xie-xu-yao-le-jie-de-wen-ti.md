# 一些需要了解的问题

## k8s

* k8s  各组件的功能
* 各种资源的含义，比如 deployment 和 ReplicaSet，有了 deployment 为什么还要ReplicaSet
* k8s 中一个服务访问另外一个服务的过程

## 证书

* 同一个主机上有多个域名，对应多个证书，应该如何区分应该使用哪个证书。比如一个节点上同时注册了多个子域名，比如 a.example.com, b.example.com, c.example.com 分别对应不同的证书，认证的时候应该如何确认应该使用哪个证书。
* https 证书认证的过程

## istio 

* 服务端口为什么需要命名
* listener 中 filter 是使用 port 来进行分组的，为什么不使用ip 来进行分组。（tcp和https 是使用ip分组的）








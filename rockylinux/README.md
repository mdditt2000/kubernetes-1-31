# Multi-Cluster Deployment using Rocky Linux

Demo on YouTube [video]()

This repo is created to document installing multiple kubernetes 1.31 clusters using Rocky Linux with Flannel and F5 BIG-IP. F5 BIG-IP uses F5 CIS to automate the networking with **StaticRouteSupport** Diagram below represents the deployment

![diagram]()

** the following arguments allows CIS to configure static routes on BIG-IP with node/pod subnets assigned to the nodes in kubernetes clusters. This enables direct routing from BIG-IP to kubernetes Pods using cluster mode without encapsulating into tunnel.

```
args:
  --static-routing-mode=true
  --orchestration-cni=<flannel>
```

CIS [repo]()

Diagram shows configured routes on BIG-IP

![Routes](g)

CIS logs showing API call to BIG-IP to configure routes

```
Wrote static route config section: {[{k8s-master-cluster-1-10.192.125.116 10.244.0.0/24 10.192.125.116} {k8s-worker-cluster-1-10.192.125.117 10.244.1.0/24 10.192.125.117} {k8s-master-cluster-2-10.192.125.118 10.245.0.0/24 10.192.125.118} {k8s-worker-cluster-2-10.192.125.119 10.245.1.0/24 10.192.125.119}]}
```
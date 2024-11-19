# Multi-Cluster Flannel Deployment using Static Route from F5 CIS

Demo on YouTube [video]()

Multiple kubernetes 1.31 cluster using default Flannel CNI with unique POD CIDRs. Creating multiple **VXLAN** tunnels will **not work** for multi-cluster deployments so routing needs to be implemented. F5 CIS can achieve this with **StaticRouteSupport** This repo demonstrates the new feature and provides a recorded demo on YouTube. Unique POD CIDR are a requirement for flannel. Diagram below represents the deployment

![diagram](https://github.com/mdditt2000/kubernetes-1-31/blob/main/multi-cluster-flannel/diagram/2024-11-19_10-47-59.png)

** the following arguments allows CIS to configure static routes on BIG-IP with node/pod subnets assigned to the nodes in kubernetes clusters. This enables direct routing from BIG-IP to kubernetes Pods using cluster mode without encapsulating into tunnel.

```
args:
  --static-routing-mode=true
  --orchestration-cni=<flannel>
```

CIS [repo](https://github.com/mdditt2000/kubernetes-1-31/tree/main/multi-cluster-flannel/cluster-1/cis-deployment)

Diagram shows configured routes on BIG-IP

![Routes](https://github.com/mdditt2000/kubernetes-1-31/blob/main/multi-cluster-flannel/diagram/2024-11-19_10-54-23.png)

CIS logs showing API call to BIG-IP to configure routes

```
Wrote static route config section: {[{k8s-master-cluster-1-10.192.125.116 10.244.0.0/24 10.192.125.116} {k8s-worker-cluster-1-10.192.125.117 10.244.1.0/24 10.192.125.117} {k8s-master-cluster-2-10.192.125.118 10.245.0.0/24 10.192.125.118} {k8s-worker-cluster-2-10.192.125.119 10.245.1.0/24 10.192.125.119}]}
```
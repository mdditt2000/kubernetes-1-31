# Setup Static Route Support with CIS and Flannel

Demo on YouTube [video]()

New kubernetes cluster using default Flannel CNI. Traditionally requirements was to use **VXLAN** however with **StaticRouteSupport** you can move away from Tunnels!! This repo demonstrates the new feature and provides a recorded demo on YouTube

** Support for CIS to configure static routes in the BIG-IP with node subnets assigned to the nodes in the OpenShift/k8s cluster. This enables direct routing from BIG-IP to k8s Pods in cluster mode without VXLAN tunnel configuration on BIG-IP

```
args:
  --static-routing-mode=true
  --orchestration-cni=<ovn-k8s/flannel>
```

Diagram shows Configured Routes on BIG-IP

![Routes](https://github.com/mdditt2000/kubernetes-1-31/blob/main/staticroutesupport/flannel/picture/2024-11-11_15-24-09.png)

CIS logs showing API call to BIG-IP to configure routes

``
NET Config: {"routes": [{"name": "k8s-master-cluster-1-10.192.125.177", "network": "10.244.0.0/24", "gw": "10.192.125.177"}, {"name": "k8s-work-cluster-1-10.192.125.178", "network": "10.244.1.0/24", "gw": "10.192.125.178"}]}
``
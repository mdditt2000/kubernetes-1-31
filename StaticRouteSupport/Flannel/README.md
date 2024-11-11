# Setup Static Route Support with CIS and Flannel

New kubernetes cluster using default Flannel CNI. Traditionally requirements was to use VXLAN however with **StaticRouteSupport** you can move away from Tunnels!! This repo demonstrates the new feature and provides a demo

** Support for CIS to configure static routes in the BIG-IP with node subnets assigned to the nodes in the OpenShift/k8s cluster. This enables direct routing from BIG-IP to k8s Pods in cluster mode without vxaln tunnel configuration on BIG-IP

``
args:
  --static-routing-mode=true
  --orchestration-cni=<ovn-k8s/flannel>
``


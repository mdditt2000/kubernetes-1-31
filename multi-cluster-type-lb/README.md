# F5 BIG-IP Multi-Cluster for Kubernetes using Service Type LoadBalancer

Demo on YouTube [video](https://youtu.be/-Ojnb8PCWDg)

This YouTube demo and and GitHub repo demonstrates how to use F5 BIG-IP for your multi-cluster Kubernetes environments using **Service Type Load Balancer**. This solution has made tremendous progress over the last year and I want thank everybody who tested and provided feedback. We believe now that **F5 BIG-IP Multi-Cluster for Kubernetes** is ready for primetime! 

In this environment I am using the following

* 3 kubernetes 1.31 cluster
* Flannel installed with unique POD networks
* Ratio traffic distribution between Kubernetes clusters using CRDs
* Keeping Ingress simple and requiring no hacking away at the CNI stack for external networking
* No additional licensing required other than BIG-IP LTM
* No CRDs, using the service to define the BIG-IP ingress via Service-Type LoadBalancing

**Note** F5 CIS can achieve this with **StaticRouteSupport** This repo demonstrates the new feature and provides a recorded demo on YouTube. Unique POD CIDR are a requirement for flannel. Diagram below represents the deployment. This is currently getting validated for Windows Nodes! 

![diagram](https://github.com/mdditt2000/kubernetes-1-31/blob/main/multi-cluster-type-lb/diagram/2024-11-19_10-47-59.png)

** the following arguments allows CIS to configure static routes on BIG-IP with node/pod subnets assigned to the nodes in kubernetes clusters. This enables direct routing from BIG-IP to kubernetes Pods using cluster mode without encapsulating into tunnel.

```
args:
  --static-routing-mode=true
  --orchestration-cni=<flannel>
```

CIS [repo](https://github.com/mdditt2000/kubernetes-1-31/tree/main/multi-cluster-type-lb/cluster-1/cis-deployment)

Diagram shows configured routes on BIG-IP

![Routes](https://github.com/mdditt2000/kubernetes-1-31/blob/main/multi-cluster-type-lb/diagram/2024-11-19_10-54-23.png)

CIS logs showing API call to BIG-IP to configure routes

```
Wrote static route config section: {[{k8s-master-cluster-1-10.192.125.116 10.244.0.0/24 10.192.125.116} {k8s-worker-cluster-1-10.192.125.117 10.244.1.0/24 10.192.125.117} {k8s-master-cluster-2-10.192.125.118 10.245.0.0/24 10.192.125.118} {k8s-worker-cluster-2-10.192.125.119 10.245.1.0/24 10.192.125.119}]}
```

# CIS Primary and Secondary

As you can see from the diagram their are **two instances** of CIS. One instance in **cluster-1 is primary** and second instance is **secondary in cluster-2**. CIS uses a heartbeat to determine the HA state. 

* This **primary/secondary heartbeat** will be re-worked in CIS 2.20 coming next. We understand the challenges and going to make some changes.

However today the configuration is created using the ConfigMap. Also **note** **serviceTypeLBDiscover** needs to be set for none local services as CIS does not discover these services by default. 

```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    f5nr: "true"
  name: extended-spec-config
  namespace: kube-system
data:
  extendedSpec: |
    mode: default
    highAvailabilityCIS:
      primaryEndPoint: http://10.192.125.122/
      probeInterval: 30
      retryInterval: 3
      primaryCluster:
        clusterName: cluster-1
        secret: default/kubeconfig-1
      secondaryCluster:
        clusterName: cluster-2
        secret: default/kubeconfig-2
        serviceTypeLBDiscovery: true
    externalClustersConfig:
    - clusterName: cluster-3
      secret: default/kubeconfig-3
      serviceTypeLBDiscovery: true
```

Extended ConfigMap [repo](https://github.com/mdditt2000/kubernetes-1-31/blob/main/multi-cluster-type-lb/cluster-1/cis-deployment/extended-spec-config.yaml)

# Ratio Traffic Configuration between Kubernetes Clusters using Service Type Load Balancing

Using a very simple TransportServer CRD to manipulate traffic between the cluster. Simple change the weight in the CRD to present the desired load balancing distribution by BIG-IP across the clusters. Only need one CRD where CIS is running. In this case cluster-1 primary and cluster-2 secondary

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    cis.f5.com/ip: 10.192.125.123
    cis.f5.com/multiClusterServices: |
      [
        {"clusterName": "cluster-1", "weight": 50},
        {"clusterName": "cluster-2", "weight": 50},
        {"clusterName": "cluster-3", "weight": 50}
      ]
  name: hello-world-app
  labels:
    app: hello-world-app
spec:
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    app: hello-world-app
  type: LoadBalancer
```

CRD [repo](https://github.com/mdditt2000/kubernetes-1-31/blob/main/multi-cluster-type-lb/cluster-1/demo-app/pod/hello-world-80/f5-hello-world-app-service.yaml)
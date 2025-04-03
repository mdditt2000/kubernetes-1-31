BigIP referenced Health monitor is not supported for the Service Type LB. Please find the sample CRs with the existing Health monitors.

```
=== Transport server
apiVersion: cis.f5.com/v1
kind: TransportServer
metadata:
  name: test-ts
spec:
  partition: big-ip-k8s  --- you can reference a global tenent in the CIS specs
  mode: standard
  pool:
    monitors:
    - name: /Common/tcp-half
      reference: bigip
    multiClusterServices:
      - clusterName: linux-01
        namespace: default
        service: f5-hello-world-3024
        servicePort: 8080
        weight: 100
      - clusterName: linux-02
        namespace: default
        service: f5-hello-world-3024
        servicePort: 8080
        weight: 0
  type: tcp
  virtualServerName: f5-hello-world-ts
  virtualServerPort: 8081
  virtualServerAddress: 10.8.4.1
```

```
=== virtual sever
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: vs-1
spec:
  host: *.example.com
  partition: big-ip-k8s  --- you can reference a global tenent in the CIS specs
  pools:
  - monitors:
    - name: /Common/tcp-half
      reference: bigip
    multiClusterServices:
    - clusterName: linux-01
      namespace: default
      service: f5-hello-world
      servicePort: 8080
      weight: 100
    - clusterName: linux-02
      namespace: default
      service: f5-hello-world
      servicePort: 8080
      weight: 0
  virtualServerAddress: 10.15.22.250
  ```
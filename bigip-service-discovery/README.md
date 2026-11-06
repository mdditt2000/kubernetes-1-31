*Note: The links and images in this document may point to external sources. Please review them before opening.*

# Kubernetes Pod IP Watcher for F5 BIG-IP AS3 Service Discovery

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.24+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![F5 AS3](https://img.shields.io/badge/F5%20AS3-3.40+-E4002B?logo=f5&logoColor=white)](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

A lightweight, event-driven Kubernetes controller and HTTP service discovery endpoint tailored specifically for **F5 BIG-IP Application Services 3 (AS3)** dynamic service discovery (`consul` or `http` providers).

It continuously monitors Kubernetes Pod lifecycle events (`ADDED`, `MODIFIED`, `DELETED`) using the Kubernetes Watch API and serves an HTTP endpoint formatted specifically to satisfy F5's `f5-service-discovery: getNodes.js` parser without needing an external HashiCorp Consul cluster.

---

## Table of Contents

* [Architecture Diagram](#architecture-diagram)
* [Pod Addition Flow Diagram](#pod-addition-flow-diagram)
* [Full Event Sequence Flow](#full-event-sequence-flow)
* [Key Features](#key-features)
* [Repository Structure](#repository-structure)
* [Prerequisites](#prerequisites)
* [Quickstart & Deployment](#quickstart--deployment)
  * [1. All-in-One Kubernetes Manifest](#1-all-in-one-kubernetes-manifest)
  * [2. Deploy to Cluster](#2-deploy-to-cluster)
  * [3. Verify Service Discovery Output](#3-verify-service-discovery-output)
* [F5 BIG-IP AS3 Declaration](#f5-big-ip-as3-declaration)
* [Configuration Options](#configuration-options)
* [Troubleshooting & Diagnostics](#troubleshooting--diagnostics)
* [License](#license)

---

## Architecture Diagram

```mermaid
flowchart TB
    subgraph K8S["Kubernetes Cluster"]
        subgraph NS_TARGET["Namespace: default"]
            POD_A["Target Pod: web-app-1<br/>IP: 10.244.1.45:80<br/>Status: Running"]
            POD_B["Target Pod: web-app-2<br/>IP: 10.244.2.88:80<br/>Status: Running"]
        end

        subgraph K8S_CP["Kubernetes Control Plane"]
            K8S_API["kube-apiserver<br/>(CoreV1 Pod Watch API)"]
        end

        subgraph NS_WATCHER["Namespace: ip-watcher-system"]
            WATCHER["pod-ip-watcher Controller<br/>(Python 3.11 In-Cluster Client)"]
            SCRIPT_CM["ConfigMap<br/>pod-ip-watcher-script"]
            STORAGE[("Atomic Cache<br/>/data/pod_ips.json")]
            SVC["Service: pod-ip-watcher-sd<br/>Type: ClusterIP | Port: 8080"]
        end

        K8S_API == "Real-Time Watch Events (ADDED / MODIFIED / DELETED)" ==> WATCHER
        SCRIPT_CM -. "Mounted at /scripts" .-> WATCHER
        WATCHER == "Atomic Write (os.replace)" ==> STORAGE
        STORAGE -. "Read into memory" .-> WATCHER
        WATCHER -- "Exposes HTTP Discovery API" --> SVC
        POD_A -. "Discovered via namespace: default" .-> K8S_API
        POD_B -. "Discovered via namespace: default" .-> K8S_API
    end

    subgraph F5_SYS["F5 BIG-IP System (TMOS)"]
        AS3["AS3 Engine (restnoded)<br/>f5-service-discovery / getNodes.js"]
        POOL["LTM Pool: web_pool<br/>Members:<br/>• 10.244.1.45:80<br/>• 10.244.2.88:80"]
        VS["Virtual Server<br/>192.168.10.150:80"]
    end

    SVC == "HTTP GET /v1/health/service/my-service?passing=true<br/>Poll Interval: 15s (Returns Root JSON Array)" ==> AS3
    AS3 -- "Dynamically updates Pool Members" --> POOL
    VS -- "Dispatches client traffic" --> POOL
    POOL == "Direct Pod Routing (Calico / Flannel / BGP CNI route)" ==> POD_A
    POOL == "Direct Pod Routing (Calico / Flannel / BGP CNI route)" ==> POD_B

    classDef k8s fill:#326ce5,stroke:#fff,stroke-width:1px,color:#fff;
    classDef f5 fill:#e4002b,stroke:#fff,stroke-width:1px,color:#fff;
    classDef comp fill:#24292e,stroke:#58a6ff,stroke-width:1px,color:#fff;
    class K8S_API,SVC,POD_A,POD_B k8s;
    class AS3,POOL,VS f5;
    class WATCHER,SCRIPT_CM,STORAGE comp;
```

---

## Pod Addition Flow Diagram

This flowchart outlines the end-to-end lifecycle and decision logic executed across Kubernetes, the controller cache, F5 AS3 service discovery, and TMOS load balancing when a new Pod is deployed:

```mermaid
flowchart TD
    subgraph STAGE1["Stage 1: Pod Scheduling & CNI Assignment"]
        A([Deploy / Scale Pod<br/>kubectl apply / CI/CD]) --> B[kube-scheduler assigns Pod to Worker Node]
        B --> C[Node CNI Plugin allocates IP address<br/>e.g., 10.244.2.88]
        C --> D[Kubelet updates Pod Status<br/>phase: Running | podIP: 10.244.2.88]
        D --> E[kube-apiserver commits status to etcd]
    end

    subgraph STAGE2["Stage 2: Real-Time Watch Stream & Ingestion"]
        E == "HTTP Streaming Watch Event: ADDED" ==> F[pod-ip-watcher Controller]
        F --> G{extract_pod_info<br/>Valid Pod IP assigned?}
        G -- No --> H[Discard / Wait for subsequent MODIFIED event]
        G -- Yes --> I[Update in-memory dictionary<br/>pods_state key: default/pod-name]
    end

    subgraph STAGE3["Stage 3: Atomic Cache Serialization"]
        I --> J[Serialize all active Pods to root JSON Array]
        J --> K[Write to temporary buffer<br/>/data/pod_ips.json.tmp]
        K --> L["Atomic swap: os.replace()<br/>/data/pod_ips.json"]
        L --> M[("Local File Cache Updated<br/>(Zero read contention / lock-free)")]
    end

    subgraph STAGE4["Stage 4: F5 AS3 Dynamic Discovery"]
        N([AS3 Polling Timer Elapsed<br/>updateInterval: 15s]) --> O["F5 restnoded (getNodes.js)<br/>HTTP GET /v1/health/service/my-service?passing=true"]
        O --> P[pod-ip-watcher HTTP Server on Port 8080]
        M -. "Read current state" .-> P
        P -- "200 OK: Root JSON Array [ {...} ]" --> Q{getNodes.js Validation<br/>Array.isArray === true?}
        Q -- No --> R[Error: Data must be a JSON array of objects]
        Q -- Yes --> S["Execute JMESPath query<br/>[*].{id: id, ip: ip}"]
    end

    subgraph STAGE5["Stage 5: TMOS Sync & Health Probing"]
        S --> T[mcpd / tmsh transaction:<br/>Add 10.244.2.88:80 to LTM web_pool]
        T --> U[BIG-IP initiates Active Health Check<br/>HTTP GET / to 10.244.2.88:80]
        U --> V{Monitor Response OK?}
        V -- No --> W[Member marked UNCHECKED / DOWN]
        V -- Yes --> X[Pool member marked UP in LTM]
    end

    subgraph STAGE6["Stage 6: Client Data Plane Forwarding"]
        Y([External Client Request]) --> Z["Virtual Server (192.168.10.150:80)"]
        Z --> AA[Load balances across active pool members]
        AA == "Routed via CNI overlay / BGP" ==> AB["Target Pod (10.244.2.88:80)"]
        X -. "Enabled for traffic distribution" .-> AA
    end

    classDef step fill:#1f2937,stroke:#58a6ff,stroke-width:1px,color:#fff;
    classDef decision fill:#0e4429,stroke:#3fb950,stroke-width:1px,color:#fff;
    classDef error fill:#490202,stroke:#f85149,stroke-width:1px,color:#fff;
    classDef startEnd fill:#326ce5,stroke:#fff,stroke-width:1px,color:#fff;
    classDef f5step fill:#791a1a,stroke:#e4002b,stroke-width:1px,color:#fff;

    class A,N,Y startEnd;
    class B,C,D,E,F,I,J,K,L,M,P,AB step;
    class G,Q,V decision;
    class H,R,W error;
    class O,S,T,U,X,Z,AA f5step;
```

---

## Full Event Sequence Flow

This diagram illustrates the complete, full lifecycle of events across all actors—from controller bootstrap and initial synchronization, to steady-state polling, dynamic scale-up/scale-down handling, connection drop recovery (`410 Gone`), and client data plane forwarding:

```mermaid
sequenceDiagram
    autonumber
    actor Client as External Client
    participant K8s as kube-apiserver
    participant Watcher as pod-ip-watcher (Daemon)
    participant Disk as Local File Cache (/data)
    participant HTTP as Discovery HTTP Server (8080)
    participant AS3 as F5 restnoded (getNodes.js)
    participant LTM as BIG-IP LTM (mcpd / Pool)
    participant Pod as Kubernetes Target Pods

    %% Phase 1: Controller Bootstrap
    rect rgb(30, 41, 59)
    Note over Watcher,Disk: Phase 1: Controller Initialization & Initial Reconciliation
    Watcher->>K8s: GET /api/v1/namespaces/default/pods (Initial List)
    K8s-->>Watcher: 200 OK (Pod-A: 10.244.1.45, resourceVersion="10050")
    Watcher->>Watcher: Populate in-memory pods_state = {"pod-a": {IP: 10.244.1.45}}
    Watcher->>Disk: Atomic write via os.replace() -> /data/pod_ips.json
    Watcher->>K8s: GET /api/v1/namespaces/default/pods?watch=true&resourceVersion=10050
    Note over K8s,Watcher: Persistent HTTP Watch Stream Open
    end

    %% Phase 2: AS3 Initial Polling
    rect rgb(55, 19, 24)
    Note over AS3,LTM: Phase 2: Steady-State AS3 Dynamic Discovery
    AS3->>HTTP: GET /v1/health/service/my-service?passing=true
    HTTP->>Disk: Read /data/pod_ips.json
    Disk-->>HTTP: Return current JSON array
    HTTP-->>AS3: 200 OK: [{"id": "pod-a", "ip": {"private": "10.244.1.45", "public": "10.244.1.45"}}]
    AS3->>AS3: Verify Array.isArray(data) === true
    AS3->>AS3: Apply JMESPath: [*].{id: id, ip: ip}
    AS3->>LTM: Sync Pool Members -> Add [10.244.1.45:80]
    LTM->>Pod: Send Active Health Monitor (HTTP GET /)
    Pod-->>LTM: 200 OK -> Mark 10.244.1.45:80 UP
    end

    %% Phase 3: Pod Addition / Scaling
    rect rgb(22, 51, 35)
    Note over K8s,LTM: Phase 3: Real-Time Pod Added (Scale-Out)
    K8s-->>Watcher: Watch Stream Event: ADDED (Pod-B: 10.244.2.88)
    Watcher->>Watcher: Extract IP & update pods_state += {"pod-b": 10.244.2.88}
    Watcher->>Disk: Atomic write -> /data/pod_ips.json (Pods: A, B)
    Note over AS3: Polling interval triggers (15s)
    AS3->>HTTP: GET /v1/health/service/my-service?passing=true
    HTTP-->>AS3: 200 OK: [Pod-A, Pod-B]
    AS3->>LTM: Update Pool Members -> Add [10.244.2.88:80]
    LTM->>Pod: Active Health Monitor probe to 10.244.2.88:80
    Pod-->>LTM: 200 OK -> Mark 10.244.2.88:80 UP
    end

    %% Phase 4: Data Plane Forwarding
    rect rgb(35, 45, 65)
    Note over Client,Pod: Phase 4: Client Traffic Load Balancing
    Client->>LTM: HTTP Request to Virtual Server (192.168.10.150:80)
    LTM->>Pod: Route to 10.244.1.45:80 (CNI Overlay Route)
    Client->>LTM: HTTP Request to Virtual Server (192.168.10.150:80)
    LTM->>Pod: Route to 10.244.2.88:80 (CNI Overlay Route)
    end

    %% Phase 5: Pod Deletion
    rect rgb(55, 19, 24)
    Note over K8s,LTM: Phase 5: Real-Time Pod Deleted (Scale-In / Failure)
    K8s-->>Watcher: Watch Stream Event: DELETED (Pod-A)
    Watcher->>Watcher: Delete "pod-a" from pods_state
    Watcher->>Disk: Atomic write -> /data/pod_ips.json (Pods: B)
    Note over AS3: Polling interval triggers (15s)
    AS3->>HTTP: GET /v1/health/service/my-service?passing=true
    HTTP-->>AS3: 200 OK: [Pod-B]
    AS3->>LTM: Update Pool Members -> Remove [10.244.1.45:80]
    end

    %% Phase 6: Connection Recovery / 410 Gone
    rect rgb(45, 30, 20)
    Note over K8s,Watcher: Phase 6: Resilient Watch Stream Recovery
    K8s-->>Watcher: HTTP 410 Gone (resourceVersion expired / etcd compacted)
    Watcher->>Watcher: Catch ApiException(status=410)
    Watcher->>K8s: GET /api/v1/namespaces/default/pods (Re-sync List)
    K8s-->>Watcher: 200 OK (Pod-B, new resourceVersion="10580")
    Watcher->>Disk: Re-verify & update /data/pod_ips.json
    Watcher->>K8s: GET /api/v1/namespaces/default/pods?watch=true&resourceVersion=10580
    Note over K8s,Watcher: Watch Stream Resumed Without Dropping Discovery API
    end
```

---

## Key Features

* **Strict AS3 Schema Compatibility**: Returns a root-level JSON Array (`[ { ... } ]`) on `/v1/health/service/<sd_service_name>?passing=true` to satisfy `Array.isArray()` checks in `f5-service-discovery: getNodes.js`.
* **Zero External Dependencies**: Simulates HashiCorp Consul's health-check service discovery endpoint directly from Kubernetes pod status without deploying or maintaining a Consul cluster.
* **Event-Driven & Low Resource Footprint**: Uses Kubernetes streaming `watch` connections instead of periodic polling loops, minimizing API server and cluster load.
* **Atomic Zero-Downtime Reads**: Updates the cached state file using `os.replace` on temporary files to ensure BIG-IP never encounters locked or partially written JSON.
* **Least-Privilege Security**: Bound via a namespace-scoped `Role` and `RoleBinding` restricted strictly to `get`, `list`, and `watch` permissions within the monitored namespace.

---

## Repository Structure

```text
.
├── README.md
├── manifests/
│   └── all-in-one.yaml        # Complete namespace, RBAC, ConfigMap, Deployment, and Service
├── as3/
│   └── as3-declaration.json  # Reference F5 AS3 declaration using Consul Service Discovery
└── src/
    ├── Dockerfile            # Container buildfile for air-gapped environments
    ├── requirements.txt      # Python dependencies
    └── pod_ip_watcher.py     # Standalone Python controller and HTTP server
```

---

## Prerequisites

* **Kubernetes Cluster**: v1.24+ (Worker nodes must route to BIG-IP or share CNI overlay routes).
* **F5 BIG-IP**: TMOS v14.1+ with **AS3 v3.40+** installed.
* **Network Connectivity**: BIG-IP must be able to reach the Service `ClusterIP` or `NodePort` on port `8080`.

---

## Quickstart & Deployment

### 1. All-in-One Kubernetes Manifest

Save the following YAML as `manifests/all-in-one.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ip-watcher-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-ip-watcher-sa
  namespace: ip-watcher-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-ip-watcher-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-ip-watcher-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: pod-ip-watcher-sa
    namespace: ip-watcher-system
roleRef:
  kind: Role
  name: pod-ip-watcher-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-ip-watcher-script
  namespace: ip-watcher-system
data:
  pod_ip_watcher.py: |
    import json
    import os
    import time
    import signal
    import logging
    from datetime import datetime, timezone
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import threading
    from kubernetes import client, config, watch
    from kubernetes.client.rest import ApiException

    logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

    stop_requested = False
    output_path = os.getenv("OUTPUT_FILE_PATH", "/data/pod_ips.json")
    http_port = int(os.getenv("HTTP_PORT", "8080"))
    sd_service_name = os.getenv("SD_SERVICE_NAME", "my-service")

    def handle_signal(signum, frame):
        global stop_requested
        logging.info(f"Signal {signum} received, stopping...")
        stop_requested = True

    signal.signal(signal.SIGTERM, handle_signal)
    signal.signal(signal.SIGINT, handle_signal)

    def load_k8s_config():
        try:
            config.load_incluster_config()
            logging.info("Loaded in-cluster Kubernetes configuration.")
        except config.ConfigException:
            config.load_kube_config()
            logging.info("Loaded local kubeconfig configuration.")

    def extract_pod_info(pod):
        if not pod.status or not pod.status.pod_ip:
            return None
        pod_ip = pod.status.pod_ip
        return {
            "id": pod.metadata.name,
            "name": pod.metadata.name,
            "namespace": pod.metadata.namespace,
            "pod_ip": pod_ip,
            "ip": {
                "private": pod_ip,
                "public": pod_ip
            },
            "host_ip": pod.status.host_ip,
            "phase": pod.status.phase,
            "node_name": pod.spec.node_name,
            "labels": pod.metadata.labels or {}
        }

    def write_json_atomically(pods_state, path):
        os.makedirs(os.path.dirname(path), exist_ok=True)
        temp_path = f"{path}.tmp"
        payload = list(pods_state.values())
        with open(temp_path, "w", encoding="utf-8") as f:
            json.dump(payload, f, indent=2)
        os.replace(temp_path, path)
        logging.info(f"Wrote {len(payload)} pods to {path}")

    class DiscoveryHandler(BaseHTTPRequestHandler):
        def do_GET(self):
            expected_prefix = f"/v1/health/service/{sd_service_name}"
            if self.path == "/" or self.path.startswith(expected_prefix) or self.path == "/pods":
                self.send_response(200)
                self.send_header("Content-Type", "application/json")
                self.end_headers()
                if os.path.exists(output_path):
                    with open(output_path, "rb") as f:
                        self.wfile.write(f.read())
                else:
                    self.wfile.write(b"[]\n")
            elif self.path == "/healthz":
                self.send_response(200)
                self.send_header("Content-Type", "text/plain")
                self.end_headers()
                self.wfile.write(b"OK\n")
            else:
                self.send_response(404)
                self.end_headers()

        def log_message(self, format, *args):
            return

    def run_http_server():
        server = HTTPServer(("0.0.0.0", http_port), DiscoveryHandler)
        logging.info(f"HTTP Discovery server listening on 0.0.0.0:{http_port}")
        while not stop_requested:
            server.handle_request()
        server.server_close()

    def watch_pods():
        load_k8s_config()
        v1 = client.CoreV1Api()
        pods_state = {}

        while not stop_requested:
            try:
                logging.info("Listing pods in namespace 'default'...")
                pod_list = v1.list_namespaced_pod(namespace="default")
                pods_state.clear()
                for pod in pod_list.items:
                    info = extract_pod_info(pod)
                    if info:
                        pods_state[pod.metadata.name] = info

                write_json_atomically(pods_state, output_path)
                resource_version = pod_list.metadata.resource_version

                w = watch.Watch()
                logging.info(f"Watching pod stream from resourceVersion={resource_version}...")
                for event in w.stream(v1.list_namespaced_pod, namespace="default", resource_version=resource_version, timeout_seconds=300):
                    if stop_requested:
                        w.stop()
                        break
                    pod = event["object"]
                    key = pod.metadata.name
                    ev_type = event["type"]
                    changed = False

                    if ev_type in ("ADDED", "MODIFIED"):
                        info = extract_pod_info(pod)
                        if info:
                            if pods_state.get(key) != info:
                                pods_state[key] = info
                                changed = True
                        elif key in pods_state:
                            del pods_state[key]
                            changed = True
                    elif ev_type == "DELETED" and key in pods_state:
                        del pods_state[key]
                        changed = True

                    if changed:
                        write_json_atomically(pods_state, output_path)

            except ApiException as e:
                if e.status == 410:
                    logging.warning("ResourceVersion expired (410 Gone). Re-syncing...")
                else:
                    logging.error(f"K8s API error: {e}. Retrying in 5s...")
                    time.sleep(5)
            except Exception as e:
                logging.error(f"Unexpected watcher error: {e}. Retrying in 5s...")
                time.sleep(5)

    if __name__ == "__main__":
        http_thread = threading.Thread(target=run_http_server, daemon=True)
        http_thread.start()
        watch_pods()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-ip-watcher
  namespace: ip-watcher-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-ip-watcher
  template:
    metadata:
      labels:
        app: pod-ip-watcher
    spec:
      serviceAccountName: pod-ip-watcher-sa
      initContainers:
        - name: install-deps
          image: python:3.11-slim
          command: ["sh", "-c", "echo 'Installing kubernetes client...'; pip install --no-cache-dir --target=/pip-packages kubernetes>=29.0.0 && echo 'Dependencies installed.'"]
          volumeMounts:
            - name: pip-lib
              mountPath: /pip-packages
      containers:
        - name: watcher
          image: python:3.11-slim
          command: ["python", "-u", "/scripts/pod_ip_watcher.py"]
          env:
            - name: PYTHONUNBUFFERED
              value: "1"
            - name: PYTHONPATH
              value: "/pip-packages"
            - name: TARGET_NAMESPACE
              value: "default"
            - name: SD_SERVICE_NAME
              value: "my-service"
            - name: HTTP_PORT
              value: "8080"
            - name: OUTPUT_FILE_PATH
              value: "/data/pod_ips.json"
          volumeMounts:
            - name: script-vol
              mountPath: /scripts
            - name: pip-lib
              mountPath: /pip-packages
            - name: shared-data
              mountPath: /data
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
      volumes:
        - name: script-vol
          configMap:
            name: pod-ip-watcher-script
        - name: pip-lib
          emptyDir: {}
        - name: shared-data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: pod-ip-watcher-sd
  namespace: ip-watcher-system
spec:
  type: ClusterIP
  selector:
    app: pod-ip-watcher
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

### 2. Deploy to Cluster

```bash
kubectl apply -f manifests/all-in-one.yaml
```

Monitor rollout until ready:

```bash
kubectl rollout status deployment/pod-ip-watcher -n ip-watcher-system
```

### 3. Verify Service Discovery Output

Run a non-interactive curl container to verify the JSON output:

```bash
kubectl run curl-test --rm -i --restart=Never --image=curlimages/curl -n default -- \
  curl -s http://pod-ip-watcher-sd.ip-watcher-system.svc.cluster.local:8080/v1/health/service/my-service?passing=true
```

Expected JSON Array output:

```json
[
  {
    "id": "nginx-demo-7bf66bd59-a1b2c",
    "name": "nginx-demo-7bf66bd59-a1b2c",
    "namespace": "default",
    "pod_ip": "10.244.1.45",
    "ip": {
      "private": "10.244.1.45",
      "public": "10.244.1.45"
    },
    "host_ip": "192.168.1.101",
    "phase": "Running",
    "node_name": "worker-node-1",
    "labels": {
      "app": "nginx-demo"
    }
  }
]
```

---

## F5 BIG-IP AS3 Declaration

Post the following declaration to your BIG-IP AS3 endpoint (`POST https://<BIG-IP-IP>/mgmt/shared/appsvcs/declare`):

```json
{
  "class": "AS3",
  "action": "deploy",
  "persist": true,
  "declaration": {
    "class": "ADC",
    "schemaVersion": "3.45.0",
    "id": "k8s-pod-discovery",
    "HTTP_SD_Tenant": {
      "class": "Tenant",
      "Sample_App": {
        "class": "Application",
        "template": "generic",
        "serviceMain": {
          "class": "Service_HTTP",
          "virtualAddresses": [
            "192.168.10.150"
          ],
          "pool": "web_pool"
        },
        "web_pool": {
          "class": "Pool",
          "monitors": [
            "http"
          ],
          "members": [
            {
              "servicePort": 80,
              "addressDiscovery": "consul",
              "updateInterval": 15,
              "uri": "http://pod-ip-watcher-sd.ip-watcher-system.svc.cluster.local:8080/v1/health/service/my-service?passing=true",
              "jmesPathQuery": "[*].{id: id, ip: ip}"
            }
          ]
        }
      }
    }
  }
}
```

---

## Configuration Options

The following environment variables can be adjusted on the `watcher` container:

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `TARGET_NAMESPACE` | `default` | Kubernetes namespace to monitor for Pods. |
| `SD_SERVICE_NAME` | `my-service` | Service name in the discovery URI path (`/v1/health/service/<SD_SERVICE_NAME>`). |
| `HTTP_PORT` | `8080` | Port for the embedded discovery HTTP server. |
| `OUTPUT_FILE_PATH` | `/data/pod_ips.json` | Path on disk where the serialized Pod state is written. |
| `PYTHONUNBUFFERED` | `1` | Forces unbuffered stdout/stderr output for instant `kubectl logs`. |

---

## Troubleshooting & Diagnostics

### 1. `severe: [f5-service-discovery: getNodes.js] error: Data must be a JSON array of objects`

* **Root Cause**: AS3's discovery parser requires the query result to evaluate to a JavaScript array (`Array.isArray() === true`).
* **Verification**: Run `curl -s <uri>` from BIG-IP. If the output starts with `{` rather than `[`, update the Python controller to the current version in this repository.
* **JMESPath**: Ensure your AS3 declaration uses `"jmesPathQuery": "[*].{id: id, ip: ip}"`.

### 2. Discovered Nodes Not Showing in BIG-IP LTM Pool

* **Check AS3 Logs on BIG-IP**:
  ```bash
  tail -f /var/log/restnoded/restnoded.log | grep -E "getNodes|Task|HTTP_SD_Tenant"
  ```
* **Verify HTTP Connectivity from BIG-IP**:
  ```bash
  curl -I http://pod-ip-watcher-sd.ip-watcher-system.svc.cluster.local:8080/v1/health/service/my-service?passing=true
  ```
* **DNS Resolution Check**: If BIG-IP cannot resolve `svc.cluster.local`, either:
  * Add Kubernetes CoreDNS to BIG-IP's DNS resolvers (`tmsh modify sys dns name-servers add { <CoreDNS-ClusterIP> }`), or
  * Replace the hostname in the AS3 `uri` with the Service's ClusterIP directly.

### 3. Check Script Ingestion inside ConfigMap

```bash
# Verify the script in ConfigMap is complete and un-truncated
kubectl get configmap pod-ip-watcher-script -n ip-watcher-system -o jsonpath='{.data.pod_ip_watcher\.py}' | wc -l

# Check compilation inside the running container
kubectl exec -it -n ip-watcher-system deploy/pod-ip-watcher -c watcher -- python3 -m py_compile /scripts/pod_ip_watcher.py
```

---

## License

This project is licensed under the Apache 2.0 License. See the [LICENSE](LICENSE) file for details.
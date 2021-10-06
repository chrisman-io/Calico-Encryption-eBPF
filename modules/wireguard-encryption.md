# WireGuard Encryption

Calico leverages WireGuard to secure on the wire in-cluster pod traffic. Wireguard was incorporated into linux kernel 5.6+ but earlier versions may be backported in certain distributions. Refer to Calico's [Encrypt data in transit](https://docs.tigera.io/compliance/encrypt-cluster-pod-traffic) documentations for more detail.

Best practice is to install Wireguard on the Base machine image used in the Node pool(s). Ubuntu 20.04LTS has `wireguard` module incorporated. However, in this demo we install `wireguard-tools` package manually.

1. Install WireGuard Tools on Kubernetes Nodes

SSH into all worker nodes and install `wireguard-tools`. Note installation method depends on package manager being used

```bash
sudo apt-get update
sudo apt install wireguard-tools -y
```

2. Enable WireGuard for a cluster

```bash
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```

3. Deploy an application across nodes

Deploy the Google Microservices demo as this creates more inter-pod traffic

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/kubernetes-manifests.yaml
```


4. Verify encryption is enabled

Wireguard Public keys are annotated on each node

```bash
kubectl get node -o yaml
```

Each node will display the Wireguard key for example:

```
...
  kind: Node
  metadata:
    annotations:
      node.alpha.kubernetes.io/ttl: "0"
      projectcalico.org/WireguardPublicKey: uwrhHmFmN0OnkGs6ZpLVeBVPLvehmwhthxiZnffGiwc=
      volumes.kubernetes.io/controller-managed-attach-detach: "true"
...
```

We can view the Wireguard tunnel interfaces on each node

```bash
#install net-tools
sudo apt install net-tools

# View Wireguard tunnel interfaces:
ifconfig
```
Output will show the `wireguard.cali` interface:
```
wireguard.cali: flags=209<UP,POINTOPOINT,RUNNING,NOARP>  mtu 8941
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)
        RX packets 32854  bytes 13593172 (13.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 39332  bytes 12721136 (12.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
Running the `wg` command will show more detail

```bash
sudo wg show
```
Output example:
```
interface: wireguard.cali
  public key: HnEEj1xdsSxLh0qulqz6c0yHZQ8Lh8nlb0x2on0nERo=
  private key: (hidden)
  listening port: 51820
  fwmark: 0x40000000

peer: uwrhHmFmN0OnkGs6ZpLVeBVPLvehmwhthxiZnffGiwc=
  endpoint: 192.168.43.178:51820
  allowed ips: 192.168.35.43/32, 192.168.61.219/32, 192.168.38.247/32, 192.168.51.115/32, 192.168.43.178/32, 192.168.59.200/32, 35.81.80.144/32, 192.168.62.39/32, 192.168.39.6/32, 192.168.59.179/32, 192.168.37.102/32, 192.168.59.215/32, 192.168.49.116/32, 192.168.63.76/32, 192.168.57.215/32, 192.168.43.174/32, 192.168.43.223/32, 192.168.44.158/32, 192.168.55.184/32, 192.168.36.62/32
  latest handshake: 35 seconds ago
  transfer: 24.49 MiB received, 23.21 MiB sent
```
<br>

*Optional Task - Viewing Wireguard stats in the Calico Enterprise UI*

Prometheus stats must be enabled and a service created to access

```bash
# Enable nodeMetricsPort
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"nodeMetricsPort":9091}}'

# Manifest to create Service, ServiceMonitor and NetworkPolicies
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: calico-prometheus-metrics
  namespace: calico-system
  labels:
    k8s-app: calico-node
spec:
  ports:
  - name: calico-prometheus-metrics-port
    port: 9091
    protocol: TCP
  selector:
    k8s-app: calico-node
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  generation: 1
  labels:
    team: network-operators
  name: calico-node-monitor-additional
  namespace: tigera-prometheus
spec:
  endpoints:
  - bearerTokenSecret:
      key: ""
    honorLabels: true
    interval: 5s
    port: calico-prometheus-metrics-port
    scrapeTimeout: 5s
  namespaceSelector:
    matchNames:
    - calico-system
  selector:
    matchLabels:
      k8s-app: calico-node
---
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  labels:
    projectcalico.org/tier: allow-tigera
  name: allow-tigera.prometheus-calico-node-prometheus-metrics-egress
  namespace: tigera-prometheus
spec:
  egress:
  - action: Allow
    destination:
      ports:
      - 9091
    protocol: TCP
    source: {}
  selector: app == 'prometheus' && prometheus == 'calico-node-prometheus'
  tier: allow-tigera
  types:
  - Egress
---
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  labels:
    projectcalico.org/tier: allow-tigera
  name: allow-tigera.calico-node-prometheus-metrics-ingress
  namespace: calico-system
spec:
  tier: allow-tigera
  selector: k8s-app == 'calico-node'
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == 'prometheus' && prometheus == 'calico-node-prometheus'
    destination:
      ports:
      - 9091
EOF
```
<br>

## Cleanup Tasks

```bash
# Delete Google Microservice Demo app
kubectl delete -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/kubernetes-manifests.yaml

# Delete Yaobank app
kubectl delete -f https://raw.githubusercontent.com/tigera/ccol2aws/main/yaobank.yaml
```


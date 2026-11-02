# Lab 18: Kubernetes Monitoring with Helm

## Objective

Deploy Prometheus, Alertmanager, Grafana, and the Prometheus node-exporter DaemonSet using the `kube-prometheus-stack` Helm chart. Use Grafana to view Kubernetes metrics.

| Component | Purpose |
|---|---|
| Prometheus | Collects and stores metrics |
| Alertmanager | Handles alerts sent by Prometheus |
| Grafana | Displays metrics in dashboards |
| Node exporter | Exposes node metrics, such as CPU and memory, on port `9100` |
| Prometheus Operator | Manages Prometheus monitoring resources |

## 1. Prepare the Monitoring Node

A new node was added to the cluster and labeled `workload=monitoring`.

Replace `YOUR_NODE_NAME` with its actual Kubernetes node name:

```bash
kubectl get nodes
kubectl label node YOUR_NODE_NAME workload=monitoring --overwrite
kubectl get nodes -L workload
```

- `get nodes`: Lists nodes and their readiness.
- `label node`: Adds the monitoring label; `--overwrite` updates it if already present.
- `-L workload`: Displays the label as a column.

**Scheduling note:** A label identifies the node but does not automatically place monitoring Pods there. The installation below uses the chart's default scheduling. Explicit node selectors in Helm values are needed to pin selected components to the new node. Node exporter should run across the required nodes, rather than only on the monitoring node.

## 2. Verify Helm and Add the Repository

Helm was installed before deploying the chart.

```bash
helm version
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

- `helm version`: Verifies Helm is available.
- `helm repo add`: Registers the repository containing the monitoring chart.
- `helm repo update`: Refreshes the local chart index.

## 3. Install the Monitoring Stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

- `monitoring`: Names this Helm release.
- `prometheus-community/kube-prometheus-stack`: Selects the chart.
- `--namespace monitoring`: Places namespaced resources in `monitoring`.
- `--create-namespace`: Creates the namespace if it does not exist. The original command without this flag requires the namespace to exist already.

The chart includes Prometheus, Alertmanager, Grafana, and node exporter by default. Prometheus Operator resources configure monitoring discovery, and the chart supplies Grafana dashboards.

If the release is already installed, inspect it instead of repeating `helm install`:

```bash
helm list -n monitoring
helm status monitoring -n monitoring
kubectl get pods -n monitoring -o wide
kubectl get svc -n monitoring
```

These commands show the release status, Pod readiness and placement, and available Services. Wait for the required Pods to become ready.

## 4. Retrieve Grafana Login Details

```bash
kubectl --namespace monitoring get secrets monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
 echo
```

- `get secrets monitoring-grafana`: Reads the Grafana Secret.
- `-o jsonpath`: Extracts the encoded administrator password.
- `base64 -d`: Decodes the value.
- `echo`: Prints a newline after the password.

For the default setup, log in with username `admin` and the retrieved password. Do not commit the password or include it in screenshots.

## 5. Access the Grafana Dashboard

Run the command used for this setup:

```bash
kubectl port-forward -n monitoring --address 0.0.0.0 \
  svc/monitoring-grafana 3000:80
```

- `port-forward`: Forwards connections to Grafana through Kubernetes.
- `svc/monitoring-grafana`: Selects the Grafana Service.
- `3000:80`: Maps port `3000` on the machine running kubectl to Service port `80`.
- `--address 0.0.0.0`: Listens on all IPv4 interfaces on that machine.

Keep this terminal open while using Grafana. From a browser on the same machine, open `http://localhost:3000`. If kubectl runs on an EC2 instance, open `http://EC2_PUBLIC_IP:3000` from your computer, provided the instance is reachable and its firewall and security group permit access. Restrict inbound TCP `3000` to your own public IP.

For access only from the machine running kubectl, omit `--address 0.0.0.0` to use the default localhost binding.

After login, open **Dashboards** and choose a provided Kubernetes dashboard, such as **Kubernetes / Compute Resources / Cluster**, if available. Select the appropriate namespace or node filters.

## 6. Validate the Lab Requirements

The following are verification steps; their results have not been supplied yet.

### Check Node Exporter Placement

```bash
kubectl get daemonsets -n monitoring
kubectl get pods -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter -o wide
kubectl get nodes -o wide
```

- `get daemonsets`: Shows desired, current, and ready Pod counts.
- `-l`: Filters Pods by the node-exporter label.
- `-o wide`: Shows the node hosting each Pod.

Confirm one ready node-exporter Pod on each required node, including the control-plane node if required by the lab. If a node is missing, inspect scheduling constraints and taints/tolerations.

### Check Metrics Exposure

From a machine that can reach a node's internal IP, replace `NODE_INTERNAL_IP` and run:

```bash
curl --max-time 10 http://NODE_INTERNAL_IP:9100/metrics
```

`curl` retrieves the metrics endpoint, and `--max-time 10` limits the request to ten seconds. Repeat for each required node. Expected output includes metrics such as `node_cpu_seconds_total` and `node_memory_MemAvailable_bytes`.

### Check Prometheus Collection and Grafana Data

Find the Prometheus Service:

```bash
kubectl get svc -n monitoring
```

In another terminal, replace `PROMETHEUS_SERVICE_NAME` with the Service exposing Prometheus on port `9090`:

```bash
kubectl port-forward -n monitoring svc/PROMETHEUS_SERVICE_NAME 9090:9090
```

Open `http://localhost:9090` on the machine running kubectl. If kubectl runs on EC2, use an SSH tunnel to reach this localhost-bound port from your computer.

Check **Status → Targets** for healthy node-exporter targets and query:

```promql
node_cpu_seconds_total
```

The chart normally configures node-exporter scraping through ServiceMonitor discovery. Confirm targets are healthy and Grafana dashboards display data before marking the lab complete.



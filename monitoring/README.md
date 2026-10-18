# Monitoring with Prometheus and Grafana

Prometheus and Grafana run on a separate VM, outside the cluster, so the monitoring keeps
working when the cluster has a problem. The cluster only runs small exporters.

| Where | What | Port |
|---|---|---|
| Every node | node-exporter: CPU, memory, disk and network of the machine | 9100 |
| Every node | kubelet (built-in cAdvisor): CPU and memory of every container | 10250 |
| One worker | kube-state-metrics: state of the pods, Deployments, volumes | NodePort 30900 |
| Monitoring VM | Prometheus, which collects the metrics every 30 seconds | 9090 |
| Monitoring VM | Grafana, with four dashboards in the folder "Fleetman" | 3000 |

Prometheus finds the three nodes through the Kubernetes API of the master
(`192.168.38.136:6443`), with a read-only token. The pods themselves are not reachable from
outside the cluster, so the container metrics are read from the kubelet of each node.

## Files

- `kubernetes.yaml`: the exporters and the read-only account, applied once on the cluster.
- `server/docker-compose.yml`: Prometheus and Grafana on the monitoring VM.
- `server/prometheus.yml`: what Prometheus collects.
- `server/grafana/`: the Prometheus data source and the four dashboards (Node Exporter Full,
  and Kubernetes Global, Nodes and Pods from grafana.com).
- `server/kubernetes/`: token and certificate of the cluster, copied by hand, never committed.

## Setup

1. On the master, deploy the exporters:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/codeTun/demo-automation/main/monitoring/kubernetes.yaml
   kubectl -n monitoring get pods -o wide
   ```

   Expected: one `node-exporter` pod per node and one `kube-state-metrics` pod, all Running.

2. On the monitoring VM, install Docker and get the files:

   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose-v2 git
   # On a VM cloned from a node where `dpkg -l containerd.io` shows the package installed,
   # use Docker's repository instead (already configured there):
   #   sudo apt install -y docker-ce docker-ce-cli docker-compose-plugin git
   sudo usermod -aG docker $USER      # then log out and log in again
   git clone https://github.com/codeTun/demo-automation.git
   mkdir -p demo-automation/monitoring/server/kubernetes
   ```

3. On the master, copy the token and the certificate of the cluster to the VM:

   ```bash
   kubectl -n monitoring get secret prometheus-external-token -o jsonpath='{.data.token}' | base64 -d > token
   kubectl -n monitoring get secret prometheus-external-token -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
   scp token ca.crt <user>@<monitoring-vm-ip>:demo-automation/monitoring/server/kubernetes/
   rm token ca.crt
   ```

4. On the monitoring VM, start Prometheus and Grafana:

   ```bash
   cd demo-automation/monitoring/server
   chmod 644 kubernetes/token kubernetes/ca.crt   # Prometheus runs as an unprivileged user
   docker compose up -d
   ```

5. Check:
   - `http://<monitoring-vm-ip>:9090/targets`: 8 targets UP (prometheus 1, node-exporter 3,
     kubelet-cadvisor 3, kube-state-metrics 1). A target that is DOWN shows its error there.
   - `http://<monitoring-vm-ip>:3000`: log in with `admin` / `admin`, choose a new password,
     then Dashboards, folder "Fleetman".

To update after a change in this folder: `git pull` then `docker compose up -d` on the VM,
and `docker compose restart prometheus` if only `prometheus.yml` changed.

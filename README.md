
# Karpenter Demo Scenario

This scenario demonstrates the capabilities of Karpenter for dynamically managing Kubernetes node pools, specifically focusing on spot and on-demand instances, as well as handling different consolidation policies. The steps below outline the process, including scaling deployments and observing how Karpenter responds to changes in demand.

## Steps

### 0. Demo Setup

- Load kube-ops-view locally and expose k8s cluser locally and grafana

```bash
kubectl proxy --accept-hosts '.*' &
docker run -it -p 8080:8080 -e CLUSTERS=http://docker.for.mac.localhost:8001 hjacobs/kube-ops-view &
kubectl port-forward --namespace monitoring svc/grafana 3000:80
```

## Install Prometheus and Grafana and add Persistent Volume for Grafana

```bash
helm repo add grafana-charts https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/prometheus-values.yaml | envsubst | tee prometheus-values.yaml
helm install --namespace monitoring prometheus prometheus-community/prometheus --values prometheus-values.yaml

curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/grafana-values.yaml | tee grafana-values.yaml
helm install --namespace monitoring grafana grafana-charts/grafana --values grafana-values.yaml
```

- Add the following PersistentVolume (PV) for Grafana.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: monitoring
  name: pv-storage-prometheus-alertmanager-0
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: "/mnt/data/prometheus-alertmanager"
```


### 1. Create a New Node Pool with `consolidationPolicy: WhenEmpty`

- Begin by creating a new node pool that has the `consolidationPolicy` set to `WhenEmpty`. The YAML for this node pool will be provided by you.

```bash
kubectl apply -f node-pool-spot.yml
```


### 2. Apply the Inflate Demo Deployment

- Deploy the Inflate demo application to create a baseline workload on your cluster.

```bash
kubectl apply -f inflate-demo.yml
```

### 3. Scale the Deployment to 1 and then to 0

- Scale the Inflate deployment to 1 to observe Karpenter allocating a spot instance.
  
```bash
kubectl scale deployment inflate --replicas=1
```

- Next, scale the deployment back to 0 and observe that the spot instance is terminated.

```bash
kubectl scale deployment inflate --replicas=0
```

- Verify the termination in AWS Spot Instance claims.

### 4. Override the Node Pool with a Different Disruption Policy

- Override the existing node pool with a new `consolidationPolicy: WhenUnderutilized` and switch to an on-demand capacity type.

```bash
kubectl apply -f node-pool-on-demand.yml
```

### 5. Scale the Inflate Deployment to 3

- Scale the deployment to 3 replicas and observe that Karpenter provisions a new on-demand instance.

```bash
kubectl scale deployment inflate --replicas=3
```

### 6. Scale the Inflate Deployment to 10

- Scale the deployment to 10 replicas. Karpenter will provision an XL instance because the previous node cannot accommodate the additional replicas.

```bash
kubectl scale deployment inflate --replicas=10
```

### 7. Scale the Inflate Deployment to 6

- Scale the deployment down to 6 replicas. Karpenter will terminate one of the nodes (3 pods) as the XL instance can now accommodate the remaining replicas.

```bash
kubectl scale deployment inflate --replicas=6
```

### 8. Override the Node Pool with Spot and On-Demand Capacity Types

- Update the node pool to support both spot and on-demand capacity types.

```bash
kubectl apply -f node-pool-both.yml
```

- Scale the deployment up to 12 replicas and evaluate Karpenter behavior (let's take alook on Karpenter logs)

```bash

# Replace kube-system namespace if you installed Karpenter within another namespace 
kubectl logs -n kube-system -c controller $(kubectl get pods -n kube-system -o name | grep karpenter | head -n 1) | less

```

### 9. Tip: Voluntary Disruptions Can Be Prevented by Disruption Budgets


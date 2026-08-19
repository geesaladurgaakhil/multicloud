# GKE Multi-Cluster Ingress

## Objective

Deploy the same application on two GKE clusters in different regions and expose both applications through a **single global external IP address** using **GKE Multi-Cluster Ingress (MCI)**.

### Architecture

```text
                    Internet
                       |
                       |
              Global External LB
                       |
                Single Public IP
                       |
             +---------+---------+
             |                   |
        GKE US Cluster      GKE EU Cluster
        (gke-us)             (gke-eu)
             |                   |
        whereami Pod        whereami Pod
             |                   |
             +---------+---------+
                       |
                Fleet / MCI
                       |
              Config Cluster
                 gke-us
```

> In this lab, `gke-us` is the **config cluster**. It controls the Multi-Cluster Ingress configuration.

---

# 1. Prerequisites

Make sure you have:

* Google Cloud project
* Billing enabled
* Google Cloud CLI (`gcloud`)
* `kubectl`
* Two GKE clusters
* Required IAM permissions

Login:

```bash
gcloud auth login
```

Set the project:

```bash
gcloud config set project PROJECT_ID
```

Enable GKE API:

```bash
gcloud services enable container.googleapis.com
```

Verify:

```bash
gcloud container clusters list
```

---

# 2. Create Two GKE Clusters

For this lab we use:

| Cluster  | Region         | Purpose              |
| -------- | -------------- | -------------------- |
| `gke-us` | `us-central1`  | Config + Application |
| `gke-eu` | `europe-west1` | Application          |

Create the US cluster:

```bash
gcloud container clusters create gke-us \
  --location=us-central1 \
  --enable-ip-alias
```

Create the EU cluster:

```bash
gcloud container clusters create gke-eu \
  --location=europe-west1 \
  --enable-ip-alias
```

Verify:

```bash
gcloud container clusters list
```

Expected:

```text
NAME     LOCATION        STATUS
gke-us   us-central1     RUNNING
gke-eu   europe-west1    RUNNING
```

---

# 3. Get Kubernetes Credentials

Get credentials for the US cluster:

```bash
gcloud container clusters get-credentials gke-us \
  --location=us-central1
```

Get credentials for the EU cluster:

```bash
gcloud container clusters get-credentials gke-eu \
  --location=europe-west1
```

Check contexts:

```bash
kubectl config get-contexts
```

You should see both clusters.

---

# 4. Register Both Clusters in a Fleet

Register `gke-us`:

```bash
gcloud container fleet memberships register gke-us \
  --gke-cluster=us-central1/gke-us \
  --enable-workload-identity
```

Register `gke-eu`:

```bash
gcloud container fleet memberships register gke-eu \
  --gke-cluster=europe-west1/gke-eu \
  --enable-workload-identity
```

Verify:

```bash
gcloud container fleet memberships list
```

You should see:

```text
gke-us
gke-eu
```

---

# 5. Enable Multi-Cluster Ingress

Make `gke-us` the config cluster.

```bash
gcloud container fleet ingress enable \
  --config-membership=gke-us \
  --location=us-central1 \
  --project=PROJECT_ID
```

This can take several minutes.

Check the status:

```bash
gcloud container fleet ingress describe \
  --project=PROJECT_ID
```

Verify that the MCI CRDs exist:

```bash
kubectl config use-context gke-us
```

Then:

```bash
kubectl get crds | grep multicluster
```

Expected:

```text
multiclusteringresses.networking.gke.io
multiclusterservices.networking.gke.io
```

The Google documentation notes that the config cluster can be any fleet member, but only the active config cluster is processed by the Multi-Cluster Ingress controller.

---

# 6. Create Namespace on Both Clusters

Create `namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: whereami
```

Switch to US:

```bash
kubectl config use-context gke-us
```

Create namespace:

```bash
kubectl apply -f namespace.yaml
```

Switch to EU:

```bash
kubectl config use-context gke-eu
```

Create namespace:

```bash
kubectl apply -f namespace.yaml
```

Verify:

```bash
kubectl get namespace whereami
```

The namespace should exist on both clusters.

---

# 7. Deploy Application on Both Clusters

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whereami
  namespace: whereami
  labels:
    app: whereami
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whereami
  template:
    metadata:
      labels:
        app: whereami
    spec:
      containers:
        - name: whereami
          image: us-docker.pkg.dev/google-samples/containers/gke/whereami:v1
          ports:
            - containerPort: 8080
```

Deploy to US:

```bash
kubectl config use-context gke-us

kubectl apply -f deployment.yaml
```

Deploy to EU:

```bash
kubectl config use-context gke-eu

kubectl apply -f deployment.yaml
```

Verify US:

```bash
kubectl get pods -n whereami
```

Verify EU:

```bash
kubectl get pods -n whereami
```

Both clusters should have running Pods.

---

# 8. Create MultiClusterService

Switch to the config cluster:

```bash
kubectl config use-context gke-us
```

Create `mcs.yaml`:

```yaml
apiVersion: networking.gke.io/v1
kind: MultiClusterService
metadata:
  name: whereami-mcs
  namespace: whereami
spec:
  template:
    spec:
      selector:
        app: whereami
      ports:
        - name: web
          protocol: TCP
          port: 8080
          targetPort: 8080
```

Apply:

```bash
kubectl apply -f mcs.yaml
```

Verify:

```bash
kubectl get mcs -n whereami
```

Expected:

```text
NAME           AGE
whereami-mcs   ...
```

The `MultiClusterService` selects Pods with:

```text
app=whereami
```

across the fleet member clusters.

---

# 9. Verify MultiClusterService

Check the generated service:

```bash
kubectl get svc -n whereami
```

You should see a generated headless Service.

Also check endpoints:

```bash
kubectl get endpoints -n whereami
```

You can inspect the MCS:

```bash
kubectl describe mcs whereami-mcs -n whereami
```

---

# 10. Create MultiClusterIngress

Create `mci.yaml`:

```yaml
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: whereami-ingress
  namespace: whereami
spec:
  template:
    spec:
      backend:
        serviceName: whereami-mcs
        servicePort: 8080
```

Apply it **only to the config cluster**:

```bash
kubectl config use-context gke-us

kubectl apply -f mci.yaml
```

Expected:

```text
multiclusteringress.networking.gke.io/whereami-ingress created
```

---

# 11. Check MultiClusterIngress

Run:

```bash
kubectl get mci -n whereami
```

Or:

```bash
kubectl get multiclusteringress -n whereami
```

Check details:

```bash
kubectl describe mci whereami-ingress -n whereami
```

Wait for Google Cloud Load Balancer resources to be created.

This can take several minutes.

---

# 12. Get the Global IP Address

Run:

```bash
kubectl get mci whereami-ingress -n whereami
```

Look for:

```text
STATUS
IP
```

You can also use:

```bash
kubectl describe mci whereami-ingress -n whereami
```

You should eventually see a public IP address.

Example:

```text
IP: 34.xxx.xxx.xxx
```

---

# 13. Test the Application

Use the IP obtained above:

```bash
curl http://GLOBAL_IP
```

Or open in your browser:

```text
http://GLOBAL_IP
```

You should receive a response from the `whereami` application.

---

# 14. Test Multi-Cluster Routing

Run:

```bash
curl http://GLOBAL_IP
```

multiple times.

The global load balancer can route requests to healthy backends across the member clusters.

Conceptually:

```text
                Global Load Balancer
                        |
              +---------+---------+
              |                   |
        gke-us Pods          gke-eu Pods
              |                   |
              +---------+---------+
                        |
                  whereami App
```

Multi-Cluster Ingress uses Google Cloud's global external Application Load Balancer and NEGs to manage Pod backends across clusters.

---

# 15. Validate Both Clusters

### Check US

```bash
kubectl config use-context gke-us

kubectl get pods -n whereami
kubectl get deployment -n whereami
```

### Check EU

```bash
kubectl config use-context gke-eu

kubectl get pods -n whereami
kubectl get deployment -n whereami
```

### Check Config Cluster

```bash
kubectl config use-context gke-us

kubectl get mcs -n whereami
kubectl get mci -n whereami
```

---

# 16. Test Failover

This is an important part of the lab.

First verify that the application is running in both clusters:

```bash
kubectl get pods -n whereami
```

Now scale down the application in `gke-eu`:

```bash
kubectl config use-context gke-eu

kubectl scale deployment whereami \
  --replicas=0 \
  -n whereami
```

Verify:

```bash
kubectl get pods -n whereami
```

Now test the global IP:

```bash
curl http://GLOBAL_IP
```

Traffic should continue to be served by the healthy cluster.

Restore the EU application:

```bash
kubectl scale deployment whereami \
  --replicas=2 \
  -n whereami
```

Verify:

```bash
kubectl get pods -n whereami
```

---

# 17. Useful Troubleshooting Commands

Check fleet:

```bash
gcloud container fleet memberships list
```

Check clusters:

```bash
gcloud container clusters list
```

Check MCI:

```bash
kubectl get mci -n whereami
```

Detailed MCI status:

```bash
kubectl describe mci whereami-ingress -n whereami
```

Check MCS:

```bash
kubectl get mcs -n whereami
```

Check Pods:

```bash
kubectl get pods -n whereami -o wide
```

Check generated Services:

```bash
kubectl get svc -n whereami
```

Check events:

```bash
kubectl get events -n whereami --sort-by=.lastTimestamp
```

Check MCI CRDs:

```bash
kubectl get crds | grep multicluster
```

---

# 18. Important Concepts

### Config Cluster

The central GKE cluster where you create:

```text
MultiClusterService
MultiClusterIngress
```

In this lab:

```text
gke-us = Config Cluster
```

### Member Cluster

A GKE cluster registered in the fleet.

```text
gke-us
gke-eu
```

### MultiClusterService

Acts as the multi-cluster equivalent of a Kubernetes Service.

It selects application Pods across member clusters.

### MultiClusterIngress

Acts as the multi-cluster equivalent of Kubernetes Ingress.

It creates/configures the global external Application Load Balancer.

### Fleet

A logical group of GKE clusters used for multi-cluster management.

### NEG

Network Endpoint Groups are used by Multi-Cluster Ingress to track Pod endpoints and provide them as load-balancer backends.

---

# 19. Final Architecture

```text
                           Internet
                              |
                              |
                       Global Anycast IP
                              |
                              |
                 +-------------------------+
                 | Global External        |
                 | Application LB         |
                 +------------+------------+
                              |
                 +------------+------------+
                 |                         |
             NEG / Pods                NEG / Pods
                 |                         |
        +--------+--------+       +--------+--------+
        |     gke-us      |       |     gke-eu      |
        |                 |       |                 |
        | whereami Pods   |       | whereami Pods   |
        +-----------------+       +-----------------+
                 ^                         ^
                 |                         |
                 +------------+------------+
                              |
                         GKE Fleet
                              |
                     Config Cluster
                         gke-us
                              |
                 +------------+------------+
                 |                         |
          MultiClusterService      MultiClusterIngress
                 |                         |
                 +------------+------------+
                              |
                     Global Load Balancer
```

# 20. Cleanup

Delete the MultiClusterIngress:

```bash
kubectl config use-context gke-us

kubectl delete -f mci.yaml
```

Delete the MultiClusterService:

```bash
kubectl delete -f mcs.yaml
```

Delete application from US:

```bash
kubectl config use-context gke-us

kubectl delete -f deployment.yaml
kubectl delete -f namespace.yaml
```

Delete application from EU:

```bash
kubectl config use-context gke-eu

kubectl delete -f deployment.yaml
kubectl delete -f namespace.yaml
```

Delete the GKE clusters:

```bash
gcloud container clusters delete gke-us \
  --location=us-central1

gcloud container clusters delete gke-eu \
  --location=europe-west1
```

---

## Official Documentation

Google Cloud's current documentation for the setup is available here:

[GKE Multi-Cluster Ingress Setup](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress-setup?authuser=19&utm_source=chatgpt.com)

[Deploying Ingress across clusters](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress?authuser=7&utm_source=chatgpt.com)

> **Note:** Multi-Cluster Ingress is a Google-hosted controller for GKE that provides a shared global external HTTP(S) load balancer across multiple clusters/regions.

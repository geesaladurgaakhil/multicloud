# End-to-End Azure DR Architecture — Portal Implementation Guide

## What we're building (mapping your diagram to real Azure resources)

| Diagram element | Azure resource | Role |
|---|---|---|
| Actor | End user | Hits your public endpoint |
| AFD | Azure Front Door (Standard/Premium) | Global entry point, WAF, edge routing |
| AD DS | Azure AD Domain Services | Identity for AKS/VM workloads (optional if using Entra ID only) |
| TF Manager | Azure Traffic Manager | DNS-level priority/failover routing between regions |
| Public Subnet (x2) | Subnets in a Hub/Spoke VNET (or per-region VNETs peered) | Hosts the regional LB + AKS nodes |
| LB (x2) | Azure Load Balancer (auto-created by AKS Service type=LoadBalancer) | Regional entry to AKS |
| AKS Zone1/Zone2 | AKS cluster with Availability Zones enabled | Compute for workloads, zone-redundant |
| ACR | Azure Container Registry (Premium, geo-replicated) | Shared image store for both regions |
| Azure Monitor | Log Analytics + Container Insights + Alerts | Observability & health probes that drive failover |

**Region plan for this guide:** Region-1 = `East US`, Region-2 = `West US 2`. Swap for whichever pair fits you (must be a paired-region combo ideally, e.g. East US / West US, or Central India / South India for your Hyderabad location if you want lower latency).

**Routing design:** Front Door (or Traffic Manager) does active/passive failover. AKS Cluster Region-1 is Priority 1 (active), Region-2 is Priority 2 (passive/standby). DR test = fail Region-1 → traffic should shift to Region-2 automatically.

---

## Prerequisites

- Azure subscription with Owner/Contributor rights
- Azure CLI installed locally *or* just Cloud Shell in the Portal (both work; this guide uses Portal UI)
- `kubectl` installed locally (for deploying the test app and simulating failure)
- A domain name is optional — Front Door gives you a default `*.azurefd.net` endpoint to test with

---

## Step 1 — Resource Group

1. Portal → **Resource groups** → **Create**
2. Name: `rg-dr-demo`
3. Region: `East US` (this is just the RG metadata region, resources inside can be anywhere)
4. **Review + create**

---

## Step 2 — Networking (VNETs + Public Subnets)

Since AKS clusters in two regions can't share one VNET (VNETs are regional), create one VNET per region and peer them only if you need cross-region pod-to-pod traffic (not required for this DR pattern — Traffic Manager/Front Door handles routing, not VNET peering).

### VNET — Region 1
1. Portal → **Virtual networks** → **Create**
2. RG: `rg-dr-demo`, Name: `vnet-region1`, Region: `East US`
3. Address space: `10.1.0.0/16`
4. Add subnet: `snet-public-region1` → `10.1.1.0/24`
5. Create

### VNET — Region 2
1. Repeat: Name `vnet-region2`, Region `West US 2`
2. Address space: `10.2.0.0/16`
3. Subnet: `snet-public-region2` → `10.2.1.0/24`
4. Create

---

## Step 3 — Azure Container Registry (shared, geo-replicated)

1. Portal → **Container registries** → **Create**
2. RG: `rg-dr-demo`, Registry name: `acrdrdemo<uniquesuffix>`, Location: `East US`
3. **SKU: Premium** (required for geo-replication)
4. Create
5. After deployment → open the registry → **Replications** → **Add**
6. Add a replication in `West US 2`
   - This means both AKS clusters pull from the *same logical registry name*, but Azure serves the image from the closest replica — no single point of failure on the registry itself.
7. Push a test image (from Cloud Shell or local Docker):
   ```
   az acr login --name acrdrdemo<uniquesuffix>
   docker pull mcr.microsoft.com/azuredocs/aks-helloworld:v1
   docker tag mcr.microsoft.com/azuredocs/aks-helloworld:v1 acrdrdemo<uniquesuffix>.azurecr.io/aks-helloworld:v1
   docker push acrdrdemo<uniquesuffix>.azurecr.io/aks-helloworld:v1
   ```

---

## Step 4 — AKS Cluster in Region-1 (Active)

1. Portal → **Kubernetes services** → **Create** → **Create a Kubernetes cluster**
2. **Basics tab**
   - RG: `rg-dr-demo`
   - Cluster name: `aks-region1`
   - Region: `East US`
   - Availability zones: **1, 2, 3** (this gives you the "Zone1 / Zone2" nodes shown in your diagram)
   - Node size: `Standard_DS2_v2` (fine for testing)
   - Node count: 2
3. **Networking tab**
   - Network configuration: **Azure CNI** (or Kubenet for simplicity)
   - Virtual network: `vnet-region1`
   - Subnet: `snet-public-region1`
   - Load balancer: **Standard** (required — this is the "LB" in your diagram, Azure creates it automatically when you expose a Service of type LoadBalancer)
   - Enable "Private cluster": **No** (we want a public LB endpoint for Traffic Manager to hit)
4. **Integrations tab**
   - Container registry: select `acrdrdemo<uniquesuffix>` (this auto-grants AcrPull role to the cluster's managed identity)
   - Azure Monitor: **Enable**, create/select a Log Analytics workspace `law-dr-demo`
5. **Review + create**

---

## Step 5 — AKS Cluster in Region-2 (Passive/Standby)

Repeat Step 4 exactly, but:
- Cluster name: `aks-region2`
- Region: `West US 2`
- VNET/Subnet: `vnet-region2` / `snet-public-region2`
- Same ACR (`acrdrdemo<uniquesuffix>`)
- Same or a regional Log Analytics workspace (`law-dr-demo` can be shared cross-region — Log Analytics workspaces can ingest from any region)

---

## Step 6 — Deploy the test workload to both clusters

Connect to each cluster and deploy the same app so you have something to actually fail over.

**Region-1:**
```
az aks get-credentials --resource-group rg-dr-demo --name aks-region1
kubectl create deployment aks-helloworld --image=acrdrdemo<uniquesuffix>.azurecr.io/aks-helloworld:v1
kubectl expose deployment aks-helloworld --type=LoadBalancer --port=80
kubectl get service aks-helloworld --watch
```
Wait until `EXTERNAL-IP` populates — that IP is your Region-1 public entry point (the "LB" box under Region-1 in your diagram).

**Region-2:** repeat against `aks-region2` context:
```
az aks get-credentials --resource-group rg-dr-demo --name aks-region2
kubectl create deployment aks-helloworld --image=acrdrdemo<uniquesuffix>.azurecr.io/aks-helloworld:v1
kubectl expose deployment aks-helloworld --type=LoadBalancer --port=80
kubectl get service aks-helloworld --watch
```
Note both external IPs — you'll need them in Step 7.

---

## Step 7 — Traffic Manager (the "TF Manager" diamond)

1. Portal → **Traffic Manager profiles** → **Create**
2. Name: `tm-dr-demo` (this becomes `tm-dr-demo.trafficmanager.net`)
3. **Routing method: Priority** (this gives active/passive DR behavior — exactly what your diagram implies with one primary flow and a fallback)
4. RG: `rg-dr-demo`
5. Create → open the profile → **Endpoints** → **Add**
   - Endpoint 1: Type = **External endpoint**, Name = `region1-endpoint`, FQDN or IP = Region-1 LB external IP, **Priority = 1**
   - Endpoint 2: Type = **External endpoint**, Name = `region2-endpoint`, FQDN or IP = Region-2 LB external IP, **Priority = 2**
6. Go to **Configuration** on the profile:
   - DNS TTL: 30 seconds (lower = faster failover for testing; default 300s is too slow for a demo)
   - Protocol: HTTP, Port 80, Path: `/`
   - Probing interval: **10 seconds**, Tolerated failures: **1** (fast detection for your test)
7. Test it: browse to `http://tm-dr-demo.trafficmanager.net` — it should resolve to Region-1 and show the hello-world page.

---

## Step 8 — Azure Front Door (the "AFD" box, sits in front of Traffic Manager)

Front Door gives you the global anycast entry point + WAF + caching that your diagram shows above Traffic Manager.

1. Portal → **Front Door and CDN profiles** → **Create** → **Front Door (Standard/Premium)**
2. RG: `rg-dr-demo`, Name: `afd-dr-demo`
3. Endpoint name: `dr-demo-app` → gives you `dr-demo-app.azurefd.net`
4. **Origin group**: Add origin group `og-regions`
   - Add origin → Origin type: **Custom**, Host name: `tm-dr-demo.trafficmanager.net`
   - (Alternative design: skip Traffic Manager entirely and add both LB IPs directly as two Front Door origins with Priority 1/2 — Front Door alone can do priority-based failover. Using Traffic Manager underneath, as your diagram shows, adds a second independent health-check layer, which is good practice for true DR.)
5. Health probes: Path `/`, Protocol HTTPS/HTTP, Interval 30s
6. Route: `/*` → origin group `og-regions`
7. Create
8. Test: browse to `https://dr-demo-app.azurefd.net` — should serve the hello-world app via Traffic Manager → Region-1.

---

## Step 9 — Azure AD Domain Services (the "AD DS" box)

Only needed if your workloads require classic AD (LDAP/Kerberos/NTLM) — e.g., legacy apps on AKS-hosted Windows containers, or domain-joined jump boxes. If your apps are fine with Entra ID (Azure AD) directly, you can skip this and just use AKS's Entra ID integration instead (simpler, no extra cost).

If you do need it:
1. Portal → **Azure AD Domain Services** → **Create**
2. RG: `rg-dr-demo`, DNS domain name: `drdemo.local` (must not clash with public domains)
3. SKU: **Standard** (Enterprise if you need it region-redundant)
4. Networking: deploy into a **dedicated subnet** in `vnet-region1` (Microsoft requires a dedicated subnet, e.g. `10.1.2.0/24`), not the AKS subnet
5. Notifications: add your admin group
6. Create — this takes ~30-45 minutes to provision
7. Once live, update the VNET's DNS servers to point to the AAD DS IPs, and peer `vnet-region2` to `vnet-region1` if Region-2 workloads also need to resolve the domain.

---

## Step 10 — Azure Monitor (top-right circle)

Most of this got wired in during Step 4/5 (Container Insights). Round it out:

1. Portal → your Log Analytics workspace `law-dr-demo` → confirm both AKS clusters appear under **Insights → Containers**
2. **Alerts** → **Create alert rule**
   - Scope: both AKS clusters
   - Condition: "Node CPU utilization" or better — "Unhealthy nodes" / API server availability
   - Action group: create one with your email/SMS
3. Also add an alert on the **Traffic Manager profile** itself: Metric = "Endpoint status by endpoint", alert when Region-1 endpoint status = Degraded — this is what tells you *why* failover triggered, mirroring the monitoring role shown in your diagram.

---

## Step 11 — Test the DR failover (the actual point of this exercise)

You have three layers that can fail over — test each so you understand the full chain:

### A. Simulate Region-1 failure at the AKS level
```
az aks get-credentials --resource-group rg-dr-demo --name aks-region1
kubectl scale deployment aks-helloworld --replicas=0
```
This makes the Region-1 Service stop responding on port 80 (no backend pods).

### B. Watch Traffic Manager detect it
- Portal → `tm-dr-demo` → **Endpoints** — within ~10-20 seconds (per your probe settings), `region1-endpoint` should flip from **Online** to **Degraded**.
- Run `nslookup tm-dr-demo.trafficmanager.net` repeatedly — once degraded, it should start resolving to the Region-2 IP.

### C. Confirm Front Door has already rerouted
- Keep refreshing `https://dr-demo-app.azurefd.net` in a browser (hard refresh / incognito to avoid DNS caching) — it should keep serving the app with zero visible downtime beyond the probe-detection window, now being served from Region-2.

### D. Measure your RTO
- Note the timestamp you scaled to 0, and the timestamp the browser test in step C started succeeding again via Region-2. That gap is your effective RTO for this config — tune Traffic Manager's probe interval/tolerated failures and Front Door's health probe interval to tighten it.

### E. Fail back
```
kubectl scale deployment aks-helloworld --replicas=2
```
Confirm the endpoint returns to **Online** and (since Region-1 is Priority 1) traffic shifts back automatically.

### Alternative failure simulations (more realistic than kubectl scale)
- **Stop the whole cluster:** `az aks stop --name aks-region1 --resource-group rg-dr-demo` — a harder failure, takes longer to detect/restart, good for testing worst-case RTO.
- **Delete the LB Service temporarily:** `kubectl delete service aks-helloworld` in Region-1 — simulates a networking-layer failure rather than an app-layer one.
- **Block the health probe path with an NSG rule** on the Region-1 subnet — tests whether your probes are actually checking the right thing (a good "did I configure this correctly" sanity check).

---

## Cleanup (avoid ongoing cost)

Delete the whole resource group once you're done testing — this removes everything except Azure AD Domain Services if you created it in a separate step (it has its own deletion protections):
```
az group delete --name rg-dr-demo --yes --no-wait
```

---

## Notes / things worth deciding before you productionize this

- **Traffic Manager vs Front Door doing the failover**: Running both (as your diagram shows) is defense-in-depth but adds a second DNS hop and TTL to reason about. Many teams pick just one. If you keep both, make Front Door's probe interval *faster* than Traffic Manager's so Front Door doesn't wait on a stale Traffic Manager view.
- **Data tier**: this guide only covers stateless compute failover. If your app has a database, that needs its own DR story (e.g., geo-replicated Azure SQL, Cosmos DB multi-region, or a stateful set with cross-region storage replication) — this diagram doesn't show a data layer, so add one if your real app has state.
- **AAD DS cost**: it's billed hourly regardless of use (~$100+/month for Standard SKU) — don't leave it running just for a DR test.

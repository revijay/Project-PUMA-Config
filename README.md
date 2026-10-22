# Project PUMA Environment Setup

## Step 1 - Create a Development Cluster in FYRE

### 1. Log in to FYRE

Open:

https://fyre.ibm.com

Sign in using your IBM credentials.

Navigate to:

**Development**

---

### 2. Create an OpenShift Cluster

Select:

**VMs -> Build VM / OCP**

Choose:

**OpenShift Container Platform (OCP)**

---

### 3. Configure the Cluster

Use the following settings:

- Site: SVL
- Group: WAS
- Select OCP Build Type: Standard
- Platform: X
- Cluster Name: project-puma
- OpenShift Version: 4.21.24
- Standard Build: Medium
- SSH Key: Leave unmarked
- Click Build

#### Cluster Topology

- 1 API Node
- 3 Master Nodes
- 3 Worker Nodes

Total Nodes: **7**

---

### 4. Wait for Provisioning

Verify:

- Hosts Down = 0
- All nodes show **VM Running**
- All nodes show **Host Running**

Expected nodes:

- api.project-puma
- project-puma-master0
- project-puma-master1
- project-puma-master2
- project-puma-worker0
- project-puma-worker1
- project-puma-worker2

---

### 5. Obtain OpenShift Access

Open the cluster details page.

Record:

- Access URL
- kubeadmin password

---

## Step 2 - Access OpenShift

### 1. Open the OpenShift Console

Open the Access URL obtained from FYRE.

### 2. Log In

Username:

```text
kubeadmin
```

Password:

```text
<password from FYRE>
```

---

## Step 3 - Create the PUMA Namespace

Navigate to:

```text
Administration → Namespaces
```

Select:

```text
Create Namespace
```

Create:

```text
Name: puma
Display Name: PUMA
Description: Usage Metering Service testing
```

Select:

```text
Create
```

---

## Step 4 - Create the UMS Catalog Source

Navigate to:

```text
Administration → Cluster Settings
```

Open:

```text
Configuration → OperatorHub → Sources
```

Select:

```text
Create CatalogSource
```

Name:

```text
usage-metering-service
```

Image:

```text
icr.io/cpopen/ibm-usage-metering-operator-catalog:1.0.4
```

Select:

```text
Create
```

---

## Step 5 - Install IBM Usage Metering Operator

Navigate to:

```text
Ecosystem → Software Catalog
```

Search:

```text
IBM Usage Metering
```

Select:

```text
Install
```

Install into namespace:

```text
puma
```

Wait for status:

```text
Succeeded
```

---

## Step 6 - Deploy IBM Usage Metering

Navigate to:

```text
Ecosystem → Installed Operators
```

Open:

```text
IBM Usage Metering
```

Select:

```text
IBM Usage Metering
```

Click:

```text
Create IBMUsageMetering
```

Paste below in the yaml view:

```yaml
apiVersion: operator.ibm.com/v1
kind: IBMUsageMetering
metadata:
  name: ibm-usage-metering-instance
  namespace: puma
spec:
  license:
    accept: true
```

Select:

```text
Create
```

Verify the IBMUsageMetering resource reaches:

```text
Ready
```

status.

---

## Step 7 - Verify UMS Deployment

Navigate to:

```text
Workloads → Pods
```

Verify all UMS pods are:

```text
Running
Ready
```

Verify there are:

```text
0 CrashLoopBackOff pods
0 Error pods
```

Verify the operator logs contain:

```text
INFO Starting manager
INFO Reconcile loop starting
INFO Reconcile loop finished
```

Verify the operand logs contain:

```text
INFO Starting IBM Usage Metering Collector...
INFO Starting ibm-usage-metering API server on port: 8080
Everything is ready. Begin running and processing data.
```

---

## Step 8 - Verify UMS Routes

Navigate to:

```text
Networking → Routes
```

Verify the following routes exist:

```text
ibm-usage-metering-fetch
ibm-usage-metering-upload
```

Record the route URLs.

These routes are required for:

- UMS API access
- Audit snapshots
- Payload export
- Troubleshooting and validation

---

## Step 9 - Create an IBM Entitlement Key Secret

Obtain an IBM Entitlement Key from:

```text
https://myibm.ibm.com/products-services/containerlibrary
```

Encode the key:

```bash
export TOKEN="<ENTITLEMENT_KEY>"
printf %s "$TOKEN" | base64 | tr -d '\n'
```

Navigate to:

```text
Workloads → Secrets
```

Select:

```text
Create → From YAML
```

Create the secret:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: swc-entitlement-key
  namespace: puma
data:
  token: <BASE64_ENCODED_ENTITLEMENT_KEY>
type: Opaque
```

---

## Step 10 - Configure Software Central Integration (Sandbox)

Navigate to:

```text
Operators → Installed Operators → IBM Usage Metering (under Provided APIs section) -> click on ibm-usage-metering-instance
```

Open:

```text
ibm-usage-metering-instance
```

Edit the resource and add:

```yaml
spec:
  sender:
    softwareCentral:
      enable: true
      entitlementKeySecret: swc-entitlement-key
      sandbox: true
      anonymize: false
```

Save the resource.

---

## Step 11 - Verify Software Central Configuration

Verify the IBMUsageMetering resource contains:

```yaml
spec:
  sender:
    softwareCentral:
      enable: true
      sandbox: true
```

Verify UMS pods remain:

```text
Running
Ready
```

> Note: After Software Central Sandbox is enabled, UMS automatically generates and uploads payloads on its configured schedule. Data may not appear immediately in downstream systems such as Segment and Amplitude.

---

## Step 12 - Create an Amplitude Chart to View Liberty Usage Metrics (Sandbox)

### 1. Log in and Select the Sandbox Project

1. Log in to Amplitude using your IBM SSO credentials.
2. In the top-left corner dropdown, search for and select:

```text
Red Hat Marketplace Data Pipeline Test (Project ID: 323463)
```

> Note: This is the designated sandbox workspace for Software Central test telemetry.

### 2. Create a New Segmentation Chart

1. Click the blue **+ Create** button in the left sidebar (or the **+** icon at the top left).
2. Select **Chart** → choose **Segmentation**.

### 3. Configure Events, Filters, and Grouping

In the left configuration panel:

**A. Select the Event**

1. Under **Events or Metrics**, click **Select event...**.
2. Search for and select: `Account Contractual Usage`.

**B. Apply Filters**

1. Click **+ Filter by**:
   - Property: `productTitle`
   - Operator: `=` (is)
   - Value: `Liberty`
2. Click **+ Filter by** again (optional, to isolate specific categories):
   - Property: `productId`
   - Operator: `=` (is)
   - Values: select `feature`, `runtime-event`, and `version-event` within the same filter line.

**C. Group by Metric / Feature**

1. Click **+ Group-by** (next to **+ Filter by**).
2. Select property: `productMetric` (or `unitDescription` / `metricId`).

### 4. Configure Measurement and Visualization Settings

1. **Change Measurement to Event Totals:** Under **Measured as** (in the middle-left section), click **Event Totals** (so it turns blue).
2. **Set Date Range:** In the top-right corner of the chart area, click the date dropdown. Change from **Last 30 days** to **Last 7 days** or **Today** (Interval: Daily).
3. **Switch Visualization Style:** Click the chart style dropdown next to the date picker and select **Bar chart** (or **Line chart**).

### 5. Save the Chart to Your Team or Personal Space

1. Click the **Save** button in the top-right corner.
2. In the Save modal:
   - Chart Name: `PUMA | Liberty Usage Metrics | Sandbox`
   - Save Location: select your Team Space or Personal Space (e.g., Sweetty P Devassy's Space).
3. Click **Save**.

---

# Validation & Troubleshooting

## Verify UMS API Connectivity

Obtain:

- UMS URL
- UMS Token

From:

```text
Workloads → ConfigMaps → ibm-usage-metering-upload-config
```

and

```text
Workloads → Secrets → ibm-usage-metering-upload-token
```

Run:

```bash
curl -k \
--request GET \
--url "<UMS_URL>/api/v1/raw" \
--header "Authorization: Bearer <TOKEN>"
```

Verify a successful response is returned.

---

## TLS Handshake Errors

If UMS API requests fail with TLS or certificate validation errors:

Export the cluster CA certificate:

```bash
kubectl get cm kube-root-ca.crt -n puma -o jsonpath="{.data.ca.crt}" > ca.crt
```

Retry the request using:

```bash
curl \
--cacert ca.crt \
<UMS_ENDPOINT>
```

For quick testing only:

```bash
curl -k \
<UMS_ENDPOINT>
```

---

## Generate an Audit Snapshot

Export the cluster CA certificate:

```bash
kubectl get cm kube-root-ca.crt -n puma -o jsonpath="{.data.ca.crt}" > ca.crt
```

Download a snapshot:

```bash
curl \
--cacert ca.crt \
--output snapshot.zip \
--request GET \
--url "<UMS_URL>/api/v1/snapshot" \
--header "Authorization: Bearer <TOKEN>"
```

Verify:

```text
snapshot.zip
```

is created.

---

## Export a Software Central Payload

Generate a Software Central payload from UMS:

```bash
curl \
--header "Authorization: Bearer <UMS_TOKEN>" \
--cacert ca.crt \
--output swc_payload.tar.gz \
"<UMS_URL>/api/v1/swc"
```

Verify:

```text
swc_payload.tar.gz
```

is created.

---

## Manually Upload a Payload to Software Central Sandbox

Use only for troubleshooting or validation purposes.

```bash
curl -X POST \
"https://sandbox.swc.saas.ibm.com/metering/api/v2/metrics" \
-H "Authorization: Bearer <IBM_ENTITLEMENT_KEY>" \
-F "file=@swc_payload.tar.gz;type=application/gzip"
```

---

## Create route or endpoint in UMS
By default one fetch endpoint and upload endpoint will be generated in UMS. But the upload endpoint can only be fetched from the cluster only. If you want to add an upload endpoint which can be called from outside then, you can do so by referring [this](https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.19.x?topic=c-creating-routes) guide.

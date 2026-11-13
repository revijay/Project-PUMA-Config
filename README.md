# Project PUMA Environment Setup

# UMS Setup

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

## Step 9 - Create Route or Endpoint in UMS

If you want to host Usage Metering Service at a public URL and the route was not created automatically during installation, create the route manually.

### Procedure

1. Log in to your OpenShift cluster console.
2. Navigate to:

```text
Networking → Routes
```

3. From the **Project** drop-down list, choose `puma` (or `ibm-usage-metering`).
4. To create the route, click **Create Route**, and choose **YAML view**.
5. Update the `spec` section as follows:

```yaml
spec:
  # Example: ibm-usage-metering-fetch-ibm-usage-metering.apps.cluster.domain.com
  host: ${ROUTE_HOSTNAME}
  
  # Target service
  to:
    kind: Service
    name: ibm-usage-metering-instance
    weight: 100
  
  # Port configuration - using named port (e.g. 'ibm-usage-metering-fetch' or 'ibm-usage-metering-upload')
  port:
    targetPort: ibm-usage-metering-fetch
```

> **Note:** `${ROUTE_HOSTNAME}` is an alias or DNS that points to the service.

6. Click **Create**.

### Example: Creating an Upload Route and Verifying Connectivity

#### 1. Create Route YAML

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ibm-usage-metering-upload
  namespace: project-puma
spec:
  host: ibm-usage-metering-upload-project-puma.apps.resh-project-puma.cp.fyre.ibm.com
  to:
    kind: Service
    name: ibm-usage-metering-instance
    weight: 100
  port:
    targetPort: ibm-usage-metering-upload
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

#### 2. Check Whether the Upload Endpoint Is Working

Post a sample event to the upload endpoint:

```bash
curl -k -X POST https://ibm-usage-metering-upload-project-puma.apps.resh-project-puma.cp.fyre.ibm.com/api/v1/event \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "productId": "version-event",
    "productName": "Liberty",
    "productMetric": "26.0.0.7",
    "metricValue": "1",
    "productCloudpakRatio": "",
    "cloudpakId": "",
    "cloudpakName": "",
    "cloudpakMetric": "",
    "logDate": "2026-10-06"
  }'
```

#### 3. Verify in UMS Logs

Metrics are successfully received in the UMS instance. Verify by checking the `ibm-usage-metering-instance` logs under **Workloads → Pods**.

---

For additional details, refer to the official IBM documentation on [Creating routes](https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.19.x?topic=c-creating-routes).

---

## Step 10 - Create an IBM Entitlement Key Secret

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

## Step 11 - Configure Software Central Integration (Sandbox)

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

## Step 12 - Verify Software Central Configuration

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

## Export a Software Central Payload Manually from UMS

Set up the required environment variables:

```bash
# 1. Get UMS Upload Route URL (for posting events; project-puma is the namespace)
export UMS_URL=$(oc get route ibm-usage-metering-upload -n project-puma -o jsonpath='{.spec.host}')

# 2. Get UMS Fetch Route URL (for exporting SWC payloads)
export FETCH_URL=$(oc get route ibm-usage-metering-fetch -n project-puma -o jsonpath='{.spec.host}')

# 3. Get UMS API Token
export UMS_TOKEN=$(oc get secret ibm-usage-metering-upload-token -n project-puma -o jsonpath='{.data.token}' | base64 -d)

# 4. Get Software Central Entitlement Key
export SWC_TOKEN=$(oc get secret swc-entitlement-key -n project-puma -o jsonpath='{.data.token}' | base64 -d)

# 5. Verify all variables are populated
echo "UMS_URL   : https://${UMS_URL}"
echo "FETCH_URL : https://${FETCH_URL}"
echo "UMS_TOKEN : ${UMS_TOKEN:0:10}..."
echo "SWC_TOKEN : ${SWC_TOKEN:0:10}..."
```

Export and send the payload:

```bash
# 1. Export the Software Central payload
curl -k -L -X GET "https://${FETCH_URL}/api/v1/swc" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  --output swc_payload.tar.gz

# 2. Verify file was created (should be > 1 KB)
ls -lh swc_payload.tar.gz

# 3. Push payload directly to Software Central Sandbox
curl -X POST "https://sandbox.swc.saas.ibm.com/metering/api/v2/metrics" \
  -H "Authorization: Bearer ${SWC_TOKEN}" \
  -F "file=@swc_payload.tar.gz;type=application/gzip"
```

View the metrics in the Software Central Sandbox dashboard:

https://sandbox.swc.saas.ibm.com/en-in/software-central/workspace/dashboard/usage?fromDate=1762214400000&toDate=1793750399999

---

# UMS Adoption Metrics Testing (CR + mappingId)

This section records how product-specific **adoption** data (as opposed to contractual licence data) was sent through UMS to Software Central and Amplitude, what was verified, and what is still open. It was tested on the `project-puma` namespace on 2026-11-12.

## Summary of Results

| Question | Answer |
|---|---|
| Can UMS send an event typed `ADOPTION`? | **[Tested]** Yes, when a registration CR (`IBMServiceMeterDefinition`) with `metricType: ADOPTION` exists and the payload carries its `mappingId`. |
| Does the flat `/api/v1/event` payload give `ADOPTION`? | **[Tested]** No. UMS logged the converted record with `"metricType":"CONTRACTUAL"` and Amplitude showed `Account Contractual Usage`. |
| Does an ADOPTION event reach Software Central? | **[Tested]** Yes. Upload status `success`, 1 of 1 events processed. |
| Does it reach Amplitude? | **[Tested]** Yes, as the event `Account Adoption Usage`, about one minute after the upload. |
| Are unregistered metric names rejected? | **[Tested]** Not in the sandbox. A made-up name (`libertyAdoptionTest`) in the metricName field passed UMS, Software Central and Amplitude. **[Said]** names are a free choice, but check with core products. |
| Is a CR required? | **[Inference]** No. A payload-only ADOPTION event with a custom `instanceId` reached UMS, Software Central and Amplitude (Test 1), but the CR `open-liberty-adoption-test` still existed in the namespace. The payload had no `mappingId` and a different metric name, so the CR was almost certainly not used. Delete the CR and resend to confirm. A CR is still needed for breakdowns **[Said]**. |
| What if `metricType` is omitted in the new format? | **[Tested, UMS log]** UMS logs `MetricType unspecified in payload, will set MetricType to ADOPTION`. The flat format is different: UMS converts it to `CONTRACTUAL`. |
| Can version and edition be sent? | **[Said]** Yes, with the DB2 pattern (`breakdownDefs`). **[Open]**, see Test 2. |
| What `productId` is accepted? | **[Tested]** The ReadyTool Cloud Annotations `productID`. The 64-character SHA-256 hash failed in Software Central. |

## Background: Why the Original Payload Did Not Work

### The hash as `productId`

Liberty originally sent a 64-character SHA-256 hash as `productId` in the flat payload. UMS accepted and exported these events, but they did not reach Amplitude, and Software Central showed `fixed_by_amendment`.

Only `productId` differed between working and failing payloads. Results by length of the hash prefix **[Tested]**:

| `productId` | Result |
|---|---|
| `runtime-event`, `version-event`, `test-product-123` | Reached Amplitude |
| 16, 32 and 40 hex characters | Reached Amplitude |
| 48, 56, 63 and 64 hex characters | Exported by UMS, not processed by Software Central |
| 41 to 47 characters | Not tested |

The Software Central contact said `productId` must be a valid ReadyTool product ID or part number. A repeated `eventId` is treated as an amendment.

### The ReadyTool product ID

| Product | Cloud Annotations `productID` | PID |
|---|---|---|
| Open Liberty | `faee6552d7104cc49c4cc1222ff6ea32` | `5724J08` (added by Software Central) |
| WebSphere Liberty 26.0 | `63a2a4dfc91f4485994c7aab94c55ab1` | `5724-J08` |

The Open Liberty ReadyTool entry lists only `PROCESSOR_VALUE_UNIT`, `SOCKET` and `VIRTUAL_PROCESSOR_CORE`, and shows "End-to-end validation with LS or UMS: Not done".

With the Open Liberty ID in the flat payload and `productMetric=VIRTUAL_PROCESSOR_CORE`, the event reached Amplitude as `Account Contractual Usage`. Software Central added `productCode`, `UT30`, `UT35` and `offerAccountingSystemCode`. That event is **contractual**, so this payload should not ship.

### `instanceId` in the flat payload

Adding `instanceId` to the flat payload made UMS reject it **[Tested]**:

```text
could not parse payload: no valid data field present in payload
```

This happened with the ReadyTool ID and with `productId=runtime-event`. The same payloads without `instanceId` were accepted. **[Said]** `instanceId` is valid in the new format at instance level, and the flat format is the legacy API. Why the flat format rejects it is **[Open]**. The UMS team asked for the failing payload. See Test 1.

## Concepts

### The two UMS resources

| | `IBMUsageMetering` | `IBMServiceMeterDefinition` |
|---|---|---|
| Purpose | Installs and configures the UMS service | Describes one metric a product sends |
| Created in | Step 6 of this guide (`ibm-usage-metering-instance`) | Not part of the base setup |
| Contains | `license.accept`, `sender.softwareCentral` | `productId`, `productName`, `metricName`, `metricType`, `processing`, `mappingId`, optional `instanceId` |
| Owner | Whoever deploys UMS | The product team |

### CR, payload and `mappingId` **[Doc]**

- The **CR is static**. It defines the metric once.
- The **JSON payload is dynamic**. It carries the measured value, a timestamp and optional attributes.
- **`mappingId`** appears in both. UMS finds the CR with the same `mappingId` and combines the two. Any unique string works. A product GUID is recommended.
- The combined size of CR and payload must not exceed 100 KB (HTTP 400 above it).

### `metricType` **[Doc]**

| Value | Meaning |
|---|---|
| `CONTRACTUAL` | Defined in the product's licence information. Appears in the audit snapshot. |
| `ADOPTION` | Not shown in compliance reports or the audit snapshot. Used only to track how much a product is used. |

### `processing` **[Doc]**

Defines how several payloads in one day combine: `MAX` (high-water mark), `ADD` (sum), `LAST` (last value), `COUNT` (number of payloads). A flat payload gets `MAX` by default **[Tested]**.

### Defaults UMS applies when the CR does not set them **[Tested]**

- `InstanceID` is the namespace (shown in Amplitude as the cluster plus namespace)
- `Method` is `RATIO_SUM`
- `MetricDescription` is the metric name

## Procedure

All commands assume namespace `project-puma`. Use the environment variables from **Export a Software Central Payload Manually from UMS**, and keep tokens out of commands you paste into chats or commits.

### Step 1 - Confirm the CRD exists

```bash
oc get crd | grep -i servicemeterdefinition
oc get ibmservicemeterdefinitions -A
```

The CRD `ibmservicemeterdefinitions.operator.ibm.com` is installed with the UMS operator. In this cluster no definitions existed before this test.

### Step 2 - Create the registration CR

`mappingId` must be a **string**. An all-digit value must be quoted, or `oc apply` fails with `spec.mappingId ... must be of type string: "number"`.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operator.ibm.com/v1
kind: IBMServiceMeterDefinition
metadata:
  name: open-liberty-adoption-test
  namespace: project-puma
spec:
  mappingId: "11111111111111111111111111111111"
  processing: LAST
  data:
    productId: faee6552d7104cc49c4cc1222ff6ea32
    productName: Open Liberty
    metricName: libertyAdoptionTest
    metricType: ADOPTION
EOF
oc get ibmservicemeterdefinitions -n project-puma
```

`libertyAdoptionTest` is a test-only metric name. Do not use it as a product metric.

### Step 3 - Wait for the CR to load, then send the payload

The payload carries only the `mappingId`, a timestamp and the value. `productId`, name, metric and type come from the CR.

```bash
curl -k -L -w "\n%{http_code}\n" -X POST "https://${UMS_URL}/api/v1/event" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  -d '[{"mappingId":"11111111111111111111111111111111","instances":[{"metrics":[{"measurementTimestamp":"2026-11-11T00:00:00Z","metricQuantity":1}]}]}]'
```

**Timing [Tested]:** the same payload sent about 2 seconds after creating the CR returned `500` with `could not parse payload: productId is required`. Sent again after a few minutes it returned `200`. **[Inference]** the CR needs a short time to be picked up. The exact delay was not measured.

Keep the response body when testing (do not use `-o /dev/null`). Errors are returned synchronously in the body, while some rejections only appear in the pod log.

### Step 4 - Verify in UMS

Pod log (`ibm-usage-metering-instance`) shows the CR and payload joined, with `"metricType":"ADOPTION"`:

```text
Checking payload {"data":{"measurementTimestamp":"2026-11-11T00:00:00Z","metricName":"libertyAdoptionTest","metricQuantity":1,"metricType":"ADOPTION","productId":"faee6552d7104cc49c4cc1222ff6ea32","productName":"Open Liberty"},"mappingId":"1111...","processing":"LAST"} against stored events
...
Validating payload
Merging data from payloads with stored events
Events updated successfully
```

The `warn ... does not support string indexing` line that precedes it comes from the old flat-format check and can be ignored for this format.

Stored data (use the **fetch** route for `/api/v1/raw`):

```bash
export FETCH_URL=$(oc get route ibm-usage-metering-fetch -n project-puma -o jsonpath='{.spec.host}')
curl -k -s -H "Authorization: Bearer ${UMS_TOKEN}" "https://${FETCH_URL}/api/v1/raw" | jq . | grep -n -B3 -A12 "libertyAdoptionTest"
```

Expected entry: `metricType: ADOPTION`, `processing: LAST`, and `dailyAggregates` for `2026-11-11` with `metricQuantity: 1`.

### Step 5 - Export and find the event

```bash
curl -k -L -X GET "https://${FETCH_URL}/api/v1/swc" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  --output swc_payload.tar.gz
mkdir -p swc_check && tar -xzf swc_payload.tar.gz -C swc_check
grep -n "libertyAdoptionTest" swc_check/usage.json
```

The exported event has `"metricType":"adoption"`. Earlier events have `"metricType":"contract"`.

### Step 6 - Upload only the new event

The export holds every stored event. Uploading all of it re-sends old events, and Software Central reports `multiStatus` with repeated `eventId`s treated as amendments. Filter to one event first:

```bash
mkdir -p swc_adoption
cp swc_check/manifest.json swc_adoption/
jq '.data |= map(select(.measuredUsage[0].metricId=="libertyAdoptionTest"))' swc_check/usage.json > swc_adoption/usage.json
jq '.data | length' swc_adoption/usage.json     # expect 1
tar -czf swc_adoption.tar.gz -C swc_adoption manifest.json usage.json

curl -X POST "https://sandbox.swc.saas.ibm.com/metering/api/v2/metrics" \
  -H "Authorization: Bearer ${SWC_TOKEN}" \
  -F "file=@swc_adoption.tar.gz;type=application/gzip"
```

### Step 7 - Verify in Software Central and Amplitude

1. Software Central Sandbox, **Incoming report status**: the row for `swc_adoption.tar.gz` shows `success`. The file status was `totalEvents: 1, routedEvents: 1, processedEvents: 1`.
2. Amplitude (Red Hat Marketplace Data Pipeline Test, project 323463): search the event **`Account Adoption Usage`**. Do not filter on `Account Contractual Usage`.
3. Useful filters: `productId = faee6552d7104cc49c4cc1222ff6ea32`, `unit = libertyAdoptionTest`, or `device_id =` the upload's correlation ID. The Amplitude `device_id` equals the Software Central correlation ID of the upload **[Tested]**, so the Amplitude "user" follows the upload and not the account.

## Observed Result in Amplitude

| Property | Value |
|---|---|
| `event_type` | `Account Adoption Usage` |
| `productId` | `faee6552d7104cc49c4cc1222ff6ea32` |
| `productTitle` | `Open Liberty` (taken from the CR) |
| `productCode` / `productCodeType` | `5724J08` / `PID` (added by Software Central) |
| `unit` | `libertyAdoptionTest` |
| `quantity` / `quantityAggregator` | `1` / `last_value` |
| `instanceId` | cluster plus namespace (set by UMS) |
| `type` | `Daily Raw usage` |
| `accountPlan` | `PAID` (the contractual events showed `CONTRACT`; cause unknown) |
| `unitMetadata.usageDetails.startDate` | `Wed, 11 Nov 2026 00:00:00 GMT` |
| `client_event_time` | one day after the usage date (`2026-11-12`) |

Timing: upload at 07:08, event in Amplitude at 07:09.

## Meeting Results with the UMS Team (2026-11-12)

Source: meeting with Tomasz Zeller, Dariusz Permus and Michał Paluch. These are **[Said]** statements from the transcript and are not yet verified against the adopter's guide. Items marked *test* are covered by Test 1 or Test 2 below.

| Topic | What was said | Status |
|---|---|---|
| `productId` | Use the ReadyTool ID. It is the reconciliation key (entitlement, CCID, IBM taxonomy). | Matches our test |
| Flat payload (`productId`, `productMetric`, `metricValue`, `logDate`) | Legacy ILS-compatible API, CONTRACTUAL by design, only for products migrating from ILS. Do not use for new work. | Matches our test |
| CR | Not required. Needed only for hierarchical metrics. A payload-only request works in the new format. A CR is recommended because static data lives there and payloads stay small. No CR-free example exists in the docs yet. | Confirmed by Test 1 |
| `instanceId` | A free string with no validation, default is the namespace. You define it with your own logic. It goes **in the payload at instance level**. | Confirmed by Test 1 |
| Our `instanceId` rejection | Unexplained. The UMS team asked for the failing payload and cluster access. | **[Open]** |
| Aggregation | Depends on `processing`: `MAX` keeps the largest value, `ADD` sums the day's payloads. | Documented |
| Who deploys UMS | The adopting product owns, installs, upgrades and configures it. Typically one per namespace. One can cover several namespaces. | Liberty-side owner **[Open]** |
| Metric names | Free choice. No unified names for adoption metrics. Check with core products to avoid inventing names. | Choice **[Open]** |
| Amendments | UMS accepts data up to 7 days back and re-uploads a changed day with the same event ID. The UMS team has not tested this. | Not tested |
| API specification | No formal OpenAPI. The adopter's guide is the reference. | - |
| Opt-out | UMS does not control the upload. The Software Central flag in the `IBMUsageMetering` CR or Helm values does. The IBM entitlement key must be configured. The product sets the flag per the customer's choice. | **[Open]** for Liberty |
| Version and edition | Use the DB2 pattern: a CR with `breakdownDefs` (`groupBy` attributes), `processing: LAST`, `metricName: INSTANCE_COUNT`, and a payload with `mappingId`, `measurementTimestamp`, `metricQuantity: 1` and the attributes as plain fields. The meaning of `repeat: true` in the DB2 CR is unknown. | *Test 2* |
| Our earlier test payload | A version-like value was used as `metricName`. That is the wrong place for it. | Understood |
| Follow-up | Send the UMS team a spreadsheet of the desired metrics (version, deployment type, licence and so on). They will help shape the payload. | **[Open]** |

## Test 1 - Payload Only, No CR, ADOPTION, `instanceId` at Instance Level

**Purpose:** check the statements that a CR is optional and that `instanceId` is accepted inside the payload.

**Result (run 2026-11-12) [Tested]:** passed end to end for a usage date of the **previous day**.

| Stage | Observed |
|---|---|
| UMS | `200`. Pod log: `Events updated successfully`, with `instanceId`, `metricType: ADOPTION` and the ReadyTool `productId` in the data. `/api/v1/raw` showed `INSTANCE_COUNT`, `ADOPTION`, `processing: MAX` (UMS default), and a daily aggregate. |
| Export (`/api/v1/swc`) | `clusterId` = `<cluster host>, test-instance-1`, `metricType: adoption`, `metricAggregationType: point-in-time`, `productType: cloudPak`. |
| Software Central | `success` for both uploads. |
| Amplitude | Event `Account Adoption Usage` for the **2026-11-11** event only. `instanceId` and `environment` = `<cluster host>, test-instance-1`, `unit` = `INSTANCE_COUNT`, `productCode` 5724J08, `quantity` 1, `quantityAggregator` `last_value`, `productTitle` Open Liberty (from the payload). `client_event_time` = 2026-11-12 00:00 (the end of the usage day). |

Conclusions:
- A CR is probably **not** needed for a plain ADOPTION metric. **[Inference]** The CR `open-liberty-adoption-test` still existed during the test. Evidence it was not used: no `mappingId` in the payload, metric name `INSTANCE_COUNT` instead of the CR's `libertyAdoptionTest`, and `processing: MAX` (the UMS default) instead of the CR's `LAST`. Delete the CR (`oc delete ibmservicemeterdefinition open-liberty-adoption-test -n project-puma`), wait a few minutes, resend with a new `instanceId` (for example `test-instance-nocr`) and a past date, and confirm `ADOPTION` in `/api/v1/raw` and the export.
- **`metricType` default [Tested, UMS log]:** a second payload without `metricType` (`instanceId` `test-instance-notype`, date 2026-11-10) was accepted, and the pod log shows `MetricType unspecified in payload, will set MetricType to ADOPTION`. So the new format defaults to ADOPTION, while the flat format is converted to CONTRACTUAL. The stored value in `/api/v1/raw` and the export was not checked. The Liberty client should still send `"metricType":"ADOPTION"` explicitly, because the default may change in another UMS version.
- A custom `instanceId` reaches Amplitude. UMS shows it as `<cluster host>, <instanceId>`. **[Tested]**
- `processing` defaulted to `MAX`. Whether it can be set in a payload-only request is **[Open]**.
- The pod log shows two `warn ... failed to eval condition` lines (the flat-format check and the `mappingId` check). They did not stop processing. **[Inference]** they are harmless.
- The event dated **2026-11-12** (the day of sending) had been uploaded with Software Central status `success`, but it was **not found in Amplitude** at the time of checking (about 17:30 UTC on 11-12). The event for 11-11, uploaded shortly after, appeared. **[Inference]** an unfinished day is held back or dated into the future, which would be a one-day lag for a daily sender. Not confirmed. Re-check the 11-12 event after 2026-11-13 00:00 UTC.
- Practical rule for tests: send a **past** date and filter Amplitude on `unit` or `productId`, not on `device_id`, which changes with each upload.

For Test 1 with a different filter: in Steps 5 and 6, replace `libertyAdoptionTest` with `INSTANCE_COUNT`. If the export holds several dates, also filter on `.start==<epoch ms of the day>`, for example `jq '.data |= map(select(.measuredUsage[0].metricId=="INSTANCE_COUNT" and .start==1794355200000))'` for 2026-11-11.

All product data is in the payload. `<INSTANCE_ID>` is any string. Use a test value such as `test-instance-1` first, because the real source (hash, UUID, or `getServerId()`) is undecided.

```bash
export INSTANCE_ID=test-instance-1

curl -k -L -w "\n%{http_code}\n" -X POST "https://${UMS_URL}/api/v1/event" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  -d '[{"instances":[{"instanceId":"'"${INSTANCE_ID}"'","metrics":[{"productId":"faee6552d7104cc49c4cc1222ff6ea32","productName":"Open Liberty","metricName":"INSTANCE_COUNT","metricQuantity":1,"metricType":"ADOPTION","measurementTimestamp":"2026-11-12T00:00:00Z"}]}]}]'
```

If the response is a parse error that names `measurementTimestamp`, remove that field and resend. Keep the response body.

**Check, in order:**

1. HTTP status and body of the curl call.
2. Pod log of `ibm-usage-metering-instance`: look for `Validating payload`, `Events updated successfully`, or `Rejecting data`. Confirm `"metricType":"ADOPTION"` and the `instanceId` value.
3. Stored data:
   ```bash
   curl -k -s -H "Authorization: Bearer ${UMS_TOKEN}" "https://${FETCH_URL}/api/v1/raw" | jq . | grep -n -B3 -A12 "INSTANCE_COUNT"
   ```
4. Export and filter as in Steps 5 and 6, replacing `libertyAdoptionTest` with `INSTANCE_COUNT`. Expect exactly 1 event before uploading.
5. Software Central status `success`, then Amplitude event `Account Adoption Usage`. Check whether `instanceId` is `test-instance-1` or still the cluster plus namespace.

**Outcomes and what each means:**

| Result | Meaning |
|---|---|
| Accepted, ADOPTION, custom `instanceId` reaches Amplitude | The CR is optional and `instanceId` in the payload works. Per-instance counts are possible. |
| Accepted, but Amplitude shows cluster plus namespace | UMS or Software Central drops the custom `instanceId`. Ask the UMS team. |
| `productId is required` or a similar rejection | A CR is needed in practice for this setup. Contradicts the meeting. Send the response to the UMS team. |
| `no valid data field present in payload` | Same message as the flat-format failure. Send both payloads to the UMS team. |

## Test 2 - Version and Edition Breakdown (DB2 Pattern)

**Purpose:** check whether Liberty version and edition can be sent as breakdown attributes and whether they reach Amplitude. Not run yet.

### 2.0 Confirm the CR schema first

The exact field names of `breakdownDefs` are **not verified** here. The CRD in the cluster is the source of truth, and the DB2 example from the meeting chat is the model. Run:

```bash
oc get crd ibmservicemeterdefinitions.operator.ibm.com -o yaml | grep -n -i -B2 -A25 "breakdown"
oc explain ibmservicemeterdefinition.spec --api-version=operator.ibm.com/v1
```

Compare the output with the CR below and correct field names before applying it. If the CRD has no `breakdownDefs`, the UMS version is too old and the test cannot run as written.

### 2.1 Create the CR

Unverified skeleton, based on the meeting description. Adjust to the schema from 2.0 and to the DB2 example.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operator.ibm.com/v1
kind: IBMServiceMeterDefinition
metadata:
  name: open-liberty-breakdown-test
  namespace: project-puma
spec:
  mappingId: "22222222222222222222222222222222"
  processing: LAST
  data:
    productId: faee6552d7104cc49c4cc1222ff6ea32
    productName: Open Liberty
    metricName: INSTANCE_COUNT
    metricType: ADOPTION
  breakdownDefs:
    - groupBy:
        - libertyVersion
        - edition
EOF
oc get ibmservicemeterdefinitions -n project-puma
```

Do not add `repeat: true` in the first run, because its meaning is unknown. If the DB2 CR has it, run a second pass with it and compare.

### 2.2 Send a payload

Wait a few minutes after creating the CR (see the timing note in Step 3).

```bash
curl -k -L -w "\n%{http_code}\n" -X POST "https://${UMS_URL}/api/v1/event" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  -d '[{"mappingId":"22222222222222222222222222222222","instances":[{"metrics":[{"measurementTimestamp":"2026-11-12T00:00:00Z","metricQuantity":1,"libertyVersion":"26.0.0.12","edition":"Open"}]}]}]'
```

Send a second payload with different values on the same day, to see how breakdowns behave:

```bash
curl -k -L -w "\n%{http_code}\n" -X POST "https://${UMS_URL}/api/v1/event" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  -d '[{"mappingId":"22222222222222222222222222222222","instances":[{"metrics":[{"measurementTimestamp":"2026-11-12T00:00:00Z","metricQuantity":1,"libertyVersion":"25.0.0.9","edition":"WebSphere"}]}]}]'
```

### 2.3 Check

1. Pod log: the CR and payload joined, no `Rejecting data`.
2. `/api/v1/raw`: search for `INSTANCE_COUNT`. Look for the breakdown values `26.0.0.12`, `Open`, `25.0.0.9`, `WebSphere` and how they are grouped. Note whether the second payload replaced the first (`LAST` on the whole metric) or was stored as a separate group.
3. Export, filter to this metric, expect the breakdown in `usage.json`.
4. Upload to Software Central, then check Amplitude for `Account Adoption Usage` and whether the breakdown values appear as properties. Record the property names.

**Outcomes and what each means:**

| Result | Meaning |
|---|---|
| Both groups stored, and the values appear in Amplitude | Version and edition can be reported this way. Use this design. |
| Stored in UMS, absent in Amplitude | Breakdowns stay in UMS and do not reach Amplitude. Ask the UMS team what Software Central forwards. |
| Second payload overwrote the first | `LAST` applies across groups in this setup. Ask how DB2 gets per-group values, and whether `repeat` is involved. |
| Rejected | Compare the CR with the DB2 example, then send both to the UMS team. |

### 2.4 After both tests

Send the UMS team: the failing flat payload with `instanceId` and its error, the working CR plus payload from the first test, the results of Tests 1 and 2, and the metrics spreadsheet.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `oc apply`: `spec.mappingId ... must be of type string: "number"` | All-digit `mappingId` parsed as a YAML number | Quote it: `mappingId: "1111..."` |
| `Invalid URL ... [No Host]` HTML page from `errors.edgesuite.net` | `UMS_URL` empty in a new shell, so the request went to `https:///api/v1/event` | Re-export the variables, check `echo $UMS_URL` |
| `500 could not parse payload: productId is required` | The CR was not loaded yet, or no CR matches the `mappingId` | Wait a few minutes and resend, check `oc get ibmservicemeterdefinitions -n project-puma` |
| `200` but pod log shows `Rejecting data ... no valid data field present in payload` | `instanceId` (or unknown keys) in a flat payload | Remove it from the flat payload. **[Said]** `instanceId` is valid at instance level in the new format (Test 1). Why the flat format rejects it is **[Open]**, asked of the UMS team |
| Pod log: `parsing time "\u003c2026-11-05\u003eT00:00:00Z"` | Literal `<` `>` left around a date placeholder | Use a plain date: `2026-11-05` |
| `jq: parse error` on `/api/v1/raw` | Wrong route or empty variable returning HTML or text | Use `${FETCH_URL}`, check with `curl -i` |
| Software Central `multiStatus` | Unfiltered export re-sent old events | Filter to the new event (Step 6) |
| Software Central `fixed_by_amendment` | Repeated `eventId`, or an invalid `productId` such as a hash | Use the ReadyTool ID, use a new date for the same product and metric |
| Curly quotes (`”`) in a copied command | Pasted through a document | Use straight quotes, or use a heredoc |

A `200` from `/api/v1/event` means the request was received. It does not prove the event was stored. Check the pod log or `/api/v1/raw`.

## Notes on the Liberty Client

- The Liberty code at the time of this test (branch `liberty-ums-integration`) still sends the **flat** payload with the SHA-256 hash as `productId` and the Liberty version in `productMetric`. It does not send the ReadyTool ID or use a CR. **[Said]** the flat format is the legacy ILS-compatible API, CONTRACTUAL by design, and should not be used for new adoption data.
- A shippable client would send the new-format payload (`instances` / `metrics`, ADOPTION), a UTC timestamp and any attributes, using a URL, token and certificate read from configuration. It should send nothing when UMS is not configured and treat non-2xx responses as warnings.
- If a CR is used, it must exist before the client sends. Who creates it for Liberty is **[Open]**.
- Security: do not commit tokens, the UMS URL or trust-all TLS code. A real bearer token was committed earlier and pushed. It must stay rotated.

## UMS Deployment Model (from the IBM guides) **[Doc]**

- UMS is installed in the **same namespace** as the product. One instance per namespace is the model, and several can be installed. **[Said]** one UMS can also cover several namespaces, and two products in one namespace share one UMS.
- Supported on OpenShift and CNCF Kubernetes. Install methods are OLM catalog source, Helm, and CASE for air-gapped environments.
- Standard mode covers usage-based and adoption metrics. PVU, VPC and GPGPU need the Helm install with the License Service sidecar.
- Endpoint URL and TLS certificate are in the ConfigMap `ibm-usage-metering-upload-config` (the guide also names `ibm-usage-metering-service-upload-config`, which is inconsistent). The token is in the Secret `ibm-usage-metering-upload-token`, key `token`.
- Containerized products use UMS. VMs and bare metal use ILMT (see the next section).
- Recommended sending frequency is at least once a day.

---


# IBM License Metric Tool (ILMT) Setup

Official Documentation Link: https://pages.github.ibm.com/cloud-license-reporter/UsageMeteringGuides/License_Metric_Tool/lmt_adoption_metrics.html

This guide describes the complete end-to-end steps to provision a VM, configure Open Liberty Adoption Metrics using IBM License Metric Tool (ILMT), run the Disconnected Scanner, upload scan results, verify data ingestion in ILMT, and validate metric delivery in IBM Software Central Sandbox and Amplitude.

### Target Data Flow

```text
Open Liberty Server (VM / Bare Metal)
  --> Local SWID Tag & UMS Files (/opt/ibm/wlp/iso-swid/)
  --> Disconnected Scanner (/opt/ibm/scanner)
  --> ILMT Test Server (https://adoption-metrics-test.pl.eurolabs.ibm.com)
  --> IBM Software Central Sandbox (https://sandbox.swc.saas.ibm.com)
  --> Amplitude Dashboard (Project Puma - "Account Adoption Usage" Event Stream)
```

## Table of Contents
1. [Prerequisites & Access Requirements](#1-prerequisites--access-requirements)
2. [Step 1: Provisioning & Preparing the Linux VM](#2-step-1-provisioning--preparing-the-linux-vm)
3. [Step 2: ReadyTool Process & Downloading Official Liberty SWID Tag](#3-step-2-readytool-process--downloading-official-liberty-swid-tag)
4. [Step 3: Setting Up Open Liberty UMS Files on the VM](#4-step-3-setting-up-open-liberty-ums-files-on-the-vm)
5. [Step 4: Installing & Running the ILMT Disconnected Scanner](#5-step-4-installing--running-the-ilmt-disconnected-scanner)
6. [Step 5: Uploading Scan Packages to ILMT Server (Automated vs. Manual)](#6-step-5-uploading-scan-packages-to-ilmt-server-automated-vs-manual)
7. [Step 6: ILMT Server Processing & Verification](#7-step-6-ilmt-server-processing--verification)
8. [Step 7: Flow 2 Data Transfer to Software Central Sandbox & Amplitude](#8-step-7-flow-2-data-transfer-to-software-central-sandbox--amplitude)
9. [Complete Troubleshooting & FAQs](#9-complete-troubleshooting--faqs)
10. [Reference Links](#10-reference-links)

---

## 1. Prerequisites & Access Requirements

Before starting, ensure you have:
1. **Linux VM:** Ubuntu 20.04/22.04 LTS or RHEL 8/9 (x86_64 architecture).
2. **IBM ReadyTool Access:** https://readytool.pl.eurolabs.ibm.com:9443/ReadyTool/login.xhtml
3. **ILMT Test Server Access:** https://adoption-metrics-test.pl.eurolabs.ibm.com
   - Request access by posting a message in the `#ilmt-faq` Slack channel:
     - Slack Thread Link: https://ibm-cloud.slack.com/archives/C02J1UUUWF8/p1793611701088659
   - Request assignment to Computer Group: "WebSphere Liberty - project PUMA" (Group ID: 141).
4. **Disconnected Scanner Package:** Download `LMT-DisconnectedScanner-linux-9.2.45.0-*.tar.gz` from Box:
   - Box Download Link: https://ibm.ent.box.com/s/5blpm6xou86tndfsrtpmi63jvl70yeya
5. **IBM Entitlement Key (IEK):** From https://myibm.ibm.com/products-services/containerlibrary
6. **Software Central Sandbox Access:** https://sandbox.swc.saas.ibm.com
7. **Manual Offline Testing Scripts (`ums_export_data.sh` & `ums_upload_to_swc.sh`):**
   - Box Download Link: https://ibm.ent.box.com/s/6gmvcbu7byruw7oatknd6lpytpljy28h

---

## 2. Step 1: Provisioning & Preparing the Linux VM

### 2.1 Provisioning the VM (FYRE or Cloud VM)
1. Go to IBM FYRE (https://fyre.ibm.com) or your virtualization portal.
2. Deploy a standard Linux VM:
   - **OS:** Ubuntu 22.04 LTS or RHEL 8/9
   - **CPU:** 4 Cores
   - **Memory:** 8 GB RAM
   - **Add Public IP**
3. Note the Public IP address (e.g. `9.60.228.101`) and root credentials.

### 2.2 SSH Access & Installing Required OS Packages
Open your terminal and SSH into the VM:
```bash
ssh root@<VM_IP>
```

Update package repositories and install required utilities (Python 3, jq, XML linter, hexdump, curl):

**For Ubuntu/Debian:**
```bash
apt update && apt install -y python3 jq libxml2-utils bsdextrautils curl
```

**For RHEL/CentOS:**
```bash
dnf install -y python3 jq libxml2 util-linux curl
```

---

## 3. Step 2: ReadyTool Process & Downloading Official Liberty SWID Tag

### 3.1 Locating the Confirmed SWID Tag in ReadyTool
To get the official SWID tag for WebSphere Application Server Liberty:

1. **Log in to IBM ReadyTool:**
   - URL: https://readytool.pl.eurolabs.ibm.com:9443/ReadyTool/login.xhtml
2. Click **Browse Products** in the top navigation bar.
3. Search for **WebSphere Application Server Liberty** and click on it.
4. Under the **Software Offering Structure**, click on **WebSphere Application Server Liberty 26.0**.
5. Navigate to the **Software Component Details** section to see confirmed registration details:
   - **Component Name:** WebSphere Application Server Liberty
   - **Persistent ID:** `fbf6a96d49214c0abc6a3bc5da6e48cd`
   - **Product ID (PID):** `5724J08`
   - **Product Persistent ID:** `63a2a4dfc91f4485994c7aab94c55ab1`
   - **Tagging Status:** `TAG_CONFIRMED`
   - **Software tag:** `ibm.com_WebSphere_Application_Server_Liberty-26.0.0.swidtag`
6. Under **Software tags**, download the XML tag file(	ibm.com_WebSphere_Application_Server_Liberty-26.0.0.swidtag) or copy its content.

### 3.2 Product Manager LICR Review Workflow (For New Releases)
When new software versions are registered:
1. Release Manager creates the ReadyTool package.
2. Product Manager / Offering Manager reviews the package against the License Information Creation & Repository (LICR) database.
3. If matches are correct, PM clicks "Accept" (or "Reject" if corrections to bundled components or pricing metrics are required).
4. Central Readiness Team approves and locks the Component GUID in the IBM Software Catalog.
5. Release Manager downloads the generated `.swidtag` and hands it to Development.

### 3.3 Official Liberty SWID Tag XML Template
- **File Name:** `ibm.com_WebSphere_Application_Server_Liberty-26.0.0.swidtag`
- **Location on VM:** `/opt/ibm/wlp/iso-swid/`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<SoftwareIdentity name="WebSphere Application Server Liberty" tagId="ibm.com-fbf6a96d49214c0abc6a3bc5da6e48cd-26.0.0" version="26.0.0" versionScheme="multipartnumeric" xml:lang="en" xmlns="http://standards.iso.org/iso/19770/-2/2015/schema.xsd">
<Meta persistentId="fbf6a96d49214c0abc6a3bc5da6e48cd"/>
<Meta generator="4-1-20210113"/>
<Entity name="IBM" regid="ibm.com" role="licensor tagCreator softwareCreator"/>
</SoftwareIdentity>
```

---

## 4. Step 3: Setting Up Open Liberty UMS Files on the VM

Reference Documentation:
https://pages.github.ibm.com/cloud-license-reporter/UsageMeteringGuides/License_Metric_Tool/contractual_metrics/lmt_enablement_procedure.html

### 4.1 Directory Structure Creation
Create the standard directory layout on the VM:
```bash
mkdir -p /opt/ibm/wlp/iso-swid
cd /opt/ibm/wlp/iso-swid
```

Save the SWID tag file:
```bash
cat << 'EOF' > /opt/ibm/wlp/iso-swid/ibm.com_WebSphere_Application_Server_Liberty-26.0.0.swidtag
<?xml version="1.0" encoding="UTF-8"?>
<SoftwareIdentity name="WebSphere Application Server Liberty" tagId="ibm.com-fbf6a96d49214c0abc6a3bc5da6e48cd-26.0.0" version="26.0.0" versionScheme="multipartnumeric" xml:lang="en" xmlns="http://standards.iso.org/iso/19770/-2/2015/schema.xsd">
<Meta persistentId="fbf6a96d49214c0abc6a3bc5da6e48cd"/>
<Meta generator="4-1-20210113"/>
<Entity name="IBM" regid="ibm.com" role="licensor tagCreator softwareCreator"/>
</SoftwareIdentity>
EOF
```
IMP:Manual Mode (Immediate Verification & PoC Testing)
This document is for  static metrics
For dynamic metrics, refer https://pages.github.ibm.com/cloud-license-reporter/UsageMeteringGuides/License_Metric_Tool/adoption_metrics/lmt_create_metadata_file.html#template-for-dynamic-metrics

1. Manual mode uses local CLI scripts (./ums_export_data.sh and ./ums_upload_to_swc.sh) from your workstation to extract data from ILMT immediately and push it to Software Central/Amplitude within seconds.The 3 Critical Rules for Manual Mode:
   
  Rule 1: Completed Past Day Timestamp (yesterday):
  
  The ILMT Export API holds current-day active data until midnight UTC.
  
  To export immediately via ./ums_export_data.sh, the measurementTimestamp in liberty.ums.data.json MUST be set to a completed past date (e.g., yesterday = now - 1 day).

  If set to today (now), the script returns "No new data available".
  
  Rule 2: Fresh Instance/Mapping ID for Repeated Same-Day Tests:
  
  Once ILMT exports a specific (mappingId + instanceId + timestamp) combination, it marks that record as already exported to prevent duplicate billing/reporting.
  
  If you need to run manual tests multiple times on the same day, change the instanceId (e.g. "/opt/ibm/wlp/manual_poc3", "/opt/ibm/wlp/manual_poc4") or mappingId to force ILMT to treat it as a new un-exported dataset.
  
  Rule 3: Wait for Hourly ETL Import:
  
  After running ./upload_results.sh on the VM, you must wait for the ILMT server to finish its scheduled hourly ETL import (runs at :30 past each hour).
Check Management > Data Imports in the ILMT console for ETL from UMSImport: Success before running ./ums_export_data.sh 

2. Automatic Mode (Production & Nightly Sync)
"LAST" Processing Overwrite Rule:
If multiple scans or version updates occur on the same calendar day, ILMT evaluates the LAST state at end-of-day. Only the final active state (e.g. 26.0.0.104) is packaged into that night's batch.


### 4.2 Creating the Metadata File (liberty.ums.meta.json)
File Path: `/opt/ibm/wlp/iso-swid/liberty.ums.meta.json`

```bash
cat << 'EOF' > /opt/ibm/wlp/iso-swid/liberty.ums.meta.json
{
  "spec": {
    "mappingId": "puma_liberty_adoption",
    "processing": "LAST",
    "repeat": true,
    "breakdownDefs": [
      {
        "breakdownBy": {
          "name": "INSTANCE_COUNT"
        },
        "groupBy": [
          "version",
          "java_version",
          "os_version",
          "is_containerized",
          "feature_cdi",
          "feature_jsonb",
          "feature_mphealth"
        ],
        "source": "metricQuantity",
        "processing": "LAST"
      }
    ],
    "data": {
      "componentId": "fbf6a96d49214c0abc6a3bc5da6e48cd",
      "componentName": "WebSphere Application Server Liberty",
      "metricName": "INSTANCE_COUNT",
      "metricDescription": "Liberty Runtime Instance Count and Adoption Telemetry",
      "metricType": "ADOPTION",
      "instanceId": "/opt/ibm/wlp/3"
    }
  }
}
EOF
```

#### Field Reference:
* `mappingId` (String, Required): Stable identifier linking metadata to payload files.
* `processing` (String, Required): Must be set to `LAST` when `repeat: true`.
* `repeat` (Boolean, Required for static): Set to `true` for continuous daily reporting of static metrics.
* `breakdownBy.name` (String, Required if `repeat: true`): Must match the `metricName` (`"INSTANCE_COUNT"`).
* `groupBy` (Array of Strings, Required if `repeat: true`): Whitelist of metric property names forwarded to Segment/Amplitude.
* `componentId` (String, Required): 32-character hex GUID from ReadyTool/SWID tag.
* `componentName` (String, Optional): Display name from SWID tag.
* `metricName` (String, Required): Primary metric name (normalized to uppercase).
* `metricType` (String, Required): Must be set to `"ADOPTION"`.
* `instanceId` (String, Optional): Installation path hint (`/opt/ibm/wlp`) for SHA-256 instance hash.
* `productId` (String, Optional): Optional IBM Product ID (e.g., `"5724J08"`).

### 4.3 Creating the Payload File with 0x1E Delimiter (liberty.ums.data.json)
File Path: `/opt/ibm/wlp/iso-swid/liberty.ums.data.json`

> **Note:** Placing `liberty.ums.data.json` directly inside `/opt/ibm/wlp/iso-swid/` alongside the metadata and SWID tag ensures the scanner creates a clean package with zero folder manifest warnings.

Run this Python script on the VM to generate the payload:
1. Manual Mode Testing
```bash
python3 -c '
import json, datetime

yesterday = (datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=1)).strftime("%Y-%m-%dT12:00:00.000Z")
RS = "\x1e"  # ASCII 0x1E Record Separator

payload = [
  {
    "mappingId": "puma_liberty_adoption",
    "instances": [
      {
        "instanceId": "/opt/ibm/wlp/3",
        "metrics": [
          {
            "measurementTimestamp": yesterday,
            "metricQuantity": 1,
            "version": "26.0.0.103",
            "java_version": "17.0.10",
            "os_version": "Linux 5.14.0",
            "is_containerized": 0,
            "feature_cdi": 1,
            "feature_jsonb": 1,
            "feature_mphealth": 1
          }
        ]
      }
    ]
  }
]

with open("/opt/ibm/wlp/iso-swid/liberty.ums.data.json", "w", encoding="utf-8") as f:
    f.write(RS + json.dumps(payload, indent=2) + "\n")
print(f"Generated liberty.ums.data.json for completed day: {yesterday}")
'
```
2.Automatic Mode
```bash
python3 -c '
import json, datetime

now = datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.000Z")
RS = "\x1e"  # ASCII 0x1E Record Separator

record = [
    {
        "mappingId": "puma_liberty_adoption",
        "instances": [
            {
                "instanceId": "/opt/ibm/wlp/3",
                "metrics": [
                    {
                        "measurementTimestamp": now,
                        "metricQuantity": 1,
                        "version": "26.0.0.70",
                        "java_version": "17.0.10",
                        "os_version": "Linux 5.14.0",
                        "is_containerized": 0,
                        "feature_cdi": 1,
                        "feature_jsonb": 1,
                        "feature_mphealth": 1
                    }
                ]
            }
        ]
    }
]

with open("/opt/ibm/wlp/iso-swid/liberty.ums.data.json", "w", encoding="utf-8") as f:
    f.write(RS + json.dumps(record, indent=2) + "\n")
print(f"SUCCESS: Generated with mappingId: puma_liberty_adoption (timestamp: {now})")
'
```
### 4.4 Pre-Flight Verification & Format Checks
Run these checks on the VM:

1. **Verify all three files exist in `/opt/ibm/wlp/iso-swid/`:**
   ```bash
   ls -la /opt/ibm/wlp/iso-swid/
   ```

2. **Check record separator (first byte MUST be `1e`):**
   ```bash
   hexdump -C /opt/ibm/wlp/iso-swid/liberty.ums.data.json | head -2
   ```
   *Expected:* `00000000  1e 5b 0a 20 ...`

3. **Validate JSON syntax:**
   ```bash
   jq . /opt/ibm/wlp/iso-swid/liberty.ums.meta.json
   tr -d '\036' < /opt/ibm/wlp/iso-swid/liberty.ums.data.json | jq .
   ```

4. **Verify `mappingId` matches:**
   ```bash
   grep mappingId /opt/ibm/wlp/iso-swid/liberty.ums.meta.json
   grep mappingId /opt/ibm/wlp/iso-swid/liberty.ums.data.json
   ```

5. **Validate SWID XML:**
   ```bash
   xmllint --noout /opt/ibm/wlp/iso-swid/ibm.com_WebSphere_Application_Server_Liberty-26.0.0.swidtag
   ```

### 4.5 File Rotation Best Practices
* Write active telemetry to `liberty.ums.data.json`.
* Maximum file size: 100 KB.
* When the file reaches 100 KB, roll it to `liberty_1.ums.data.json`, keeping a maximum of 10 history files.

---

## 5. Step 4: Installing & Running the ILMT Disconnected Scanner

### 5.1 Downloading Scanner from Box & Extracting to /opt/ibm/scanner
1. On your Mac/workstation, download `LMT-DisconnectedScanner-linux-9.2.45.0-*.tar.gz` from Box:
   https://ibm.ent.box.com/s/5blpm6xou86tndfsrtpmi63jvl70yeya
2. Copy it to the VM:
   ```bash
   scp LMT-DisconnectedScanner-linux-*.tar.gz root@<VM_IP>:/root/
   ```
3. On the VM, extract to `/opt/ibm/scanner`:
   ```bash
   mkdir -p /opt/ibm/scanner
   cd /opt/ibm/scanner
   tar -xzf /root/LMT-DisconnectedScanner-linux-*.tar.gz
   ```

### 5.2 Initial Setup
Run the setup script once:
```bash
cd /opt/ibm/scanner
./setup.sh
```

### 5.3 Running the Scan
Execute the catalog, ISO tag, and usage metering scan:
```bash
./run_sw_and_pack.sh
```

Verify scanner output package:
```bash
ls -la /opt/ibm/scanner/output/
```
*Expected archive created:* `/opt/ibm/scanner/output/<timestamp>-<hostname>-<id>.tar.gz`

---

## 6. Step 5: Uploading Scan Packages to ILMT Server (Automated vs. Manual)

### 6.1 How to Obtain the ILMT API Token (DATA_UPLOAD_TOKEN)
1. Log in to the ILMT Web Console:
   - URL: https://adoption-metrics-test.pl.eurolabs.ibm.com
2. In the top-right navigation bar, click your User Profile icon -> select **User Profile**.
3. Scroll to the **API Token** / **REST API Token** section.
4. Click **Generate Token** (or "Show Token" / "Copy").
5. Copy the 40-character hexadecimal token (e.g. `5929654d9786c9becf52cf283ffd8313ce53066b`).

### 6.2 Option A: Automated CLI Upload (./upload_results.sh)
```bash
export DATA_UPLOAD_URL="adoption-metrics-test.pl.eurolabs.ibm.com:443"
export DATA_UPLOAD_TOKEN="<PASTE_YOUR_API_TOKEN_HERE>"

cd /opt/ibm/scanner
./upload_results.sh
```

* **CRITICAL URL RULE:** `DATA_UPLOAD_URL` must only be `"<hostname>:<port>"` or `"<hostname>"`. Do NOT include `"https://"` or API paths because `upload_results.sh` automatically prepends `"https://"` and appends `"/api/sam/v2/scan_results_upload"`.
* **Expected Output:**
  ```text
  Uploading scan results...
  Processing the <timestamp>-demo-puma2...tar.gz file...
  The <timestamp>-demo-puma2...tar.gz file was uploaded and removed
  Scan Results Upload script finished
  ```
* **Persistence Tip:** Add the exports to `/root/.bashrc` on the VM so they persist across SSH logins:
  ```bash
  echo 'export DATA_UPLOAD_URL="adoption-metrics-test.pl.eurolabs.ibm.com:443"' >> /root/.bashrc
  echo 'export DATA_UPLOAD_TOKEN="5929654d9786c9becf52cf283ffd8313ce53066b"' >> /root/.bashrc
  ```

### 6.3 Option B: Manual Web Portal Upload
(Use this if you prefer manual web uploads instead of Option A):

> **Note on `/opt/ibm/scanner/output/`:** When you run `./upload_results.sh` (Option A), it automatically deletes the `.tar.gz` package after a successful upload to save disk space. If you want to perform a manual upload (Option B) after running Option A, generate a fresh archive first by running `./run_sw_and_pack.sh`.

1. **(If output folder is empty) Generate a fresh scan package on the VM:**
   ```bash
   cd /opt/ibm/scanner
   ./run_sw_and_pack.sh
   ```

2. **Download the archive from VM to your Mac:**
   ```bash
   scp 'root@<VM_IP>:/opt/ibm/scanner/output/*.tar.gz' ~/Downloads/
   ```

3. **Log in to ILMT Console:** https://adoption-metrics-test.pl.eurolabs.ibm.com
4. Go to **Management** -> **Scan Results Upload**.
5. Select the downloaded `.tar.gz` archive and click **Upload**.

---

## 7. Step 6: ILMT Server Processing & Verification

Test Instance: https://adoption-metrics-test.pl.eurolabs.ibm.com
*(Note: Ensure your account has access via `#ilmt-faq` Slack channel).*

### 7.1 Scheduled Hourly Data Imports Log Verification
The ILMT server runs an automated ETL import every hour on schedule (e.g. at 2:30, 3:30, etc.).
If you upload between 2:31 and 3:29, your uploaded file will be processed during the 3:30 run.

* Navigate to: **Management** -> **Data Imports**
* Review the latest import log. Confirm these key entries:
  - `INFO: Processing scan package: '<timestamp>-<vm-name>...tar.gz' (computer group id=141)`
  - `INFO: DisconnectedScannerProcessingStatistics: Number of successfully processed files: 1`
  - `INFO: ETL from UMSImport: Start`
  - `INFO: ETL from UMSImport: Success`

### 7.2 Adoption Data Report Verification in ILMT
After the ETL import completes:
* Navigate to: **Reports** -> **Adoption Data**
* Confirm your VM entry in the table:
  - **Computer Name:** `<vm-name>.dev.fyre.ibm.com`
  - **Matching Type:** Component Only
  - **Component ID:** `fbf6a96d49214c0abc6a3bc5da6e48cd`
  - **Metric Name:** `INSTANCE_COUNT`
  - **Path:** `/opt/ibm/wlp/iso-swid/liberty.ums.meta.json`
  - **Is Repeat:** 1
  - **Inventory Match:** `Matched Inventory >` (matched with WebSphere Liberty in IBM Software Catalog)
  - **Last Uploaded Day:** Date when LMT last forwarded the daily batch to Software Central
  - **Details:** Click "DETAILS >" to view breakdown properties (`version`, `java_version`, `os_version`, `is_containerized`, `feature_*`)

---

## 8. Step 7: Flow 2 Data Transfer to Software Central Sandbox & Amplitude

### 8.1 Option A: Automated Online Sync (Computer Group 141)
* In ILMT, go to **Management** -> **Computer Groups** -> **WebSphere Liberty - project PUMA** (ID: 141).
* Ensure "Enable integration with IBM Software Central" is checked with a valid IBM Entitlement Key (IEK) and "Automatically send adoption data" is enabled.
* **Timing Note:** ILMT aggregates adoption data on a daily schedule and sends it outbound to Software Central in automated batches. If you use Option A, you will need to wait until the next day to see the data reflected in Software Central and Amplitude.

### 8.2 Option B: Manual Offline Export & Upload Scripts (Immediate Verification)
* **Timing Advantage:** If you perform these manual script steps (Option B), the adoption data is extracted and pushed immediately to Software Central Sandbox. You can see the metrics on Amplitude right away (within minutes) without waiting until the next day!

CRITICAL RULES TO PREVENT "NO NEW DATA AVAILABLE" ERROR:

To successfully export data using ./ums_export_data.sh without getting "No new data available", you must satisfy these 3 requirements:

Completed Past Day: The measurementTimestamp in liberty.ums.data.json MUST be a completed past date (e.g. "2026-10-25T12:00:00.000Z"), NOT today's date. ILMT export API holds today's active data until midnight UTC.

Unique Mapping ID / Instance ID: If you are testing repeatedly, use a fresh mappingId (e.g. "puma_manual_export_poc") and instanceId (e.g. "/opt/ibm/wlp/manual_poc") so ILMT does not skip the record as an already-exported period.

Wait for Hourly Import: Ensure the LMT server finishes its hourly ETL import (Management -> Data Imports -> "ETL from UMSImport: Success") before running ./ums_export_data.sh.


Download scripts package (`ums_export_data.sh` & `ums_upload_to_swc.sh`) from Box:
- Box Link: https://ibm.ent.box.com/s/6gmvcbu7byruw7oatknd6lpytpljy28h

1. **Export from LMT test server (run from your Mac/workstation):**
   ```bash
   export ILMT_HOST="adoption-metrics-test.pl.eurolabs.ibm.com"
   export ILMT_PORT="443"
   export COMPUTER_GROUP_ID="141"
   export ILMT_TOKEN="<LMT_API_TOKEN>"
   ./ums_export_data.sh
   ```
   *(Produces: `ums_adoption_metrics_export.tar.gz`)*

2. **Upload to Software Central Sandbox:**
   ```bash
   export SWC_URL="https://sandbox.swc.saas.ibm.com/metering/api/v2/metrics?authorizeAccountCreation=true"
   export IBM_ENTITLEMENT_KEY="<YOUR_IEK>"
   ./ums_upload_to_swc.sh
   ```
   *(Output: `Success! Data uploaded to IBM Software Central`)*

### 8.3 Verifying "Account Adoption Usage" in Amplitude
1. Log in to Software Central Sandbox (https://sandbox.swc.saas.ibm.com) and check **Incoming report status** tab -> Status: "Success".
2. Open Amplitude (Project Puma / Red Hat Marketplace Data Pipeline Test).
3. Search for the **Account Adoption Usage** event stream and inspect properties:
   - `source`: `"ILMT"`
   - `unit`: `"INSTANCE_COUNT"`
   - `quantity`: 1
   - `unitMetadata.componentId`: `"fbf6a96d49214c0abc6a3bc5da6e48cd"`
   - `unitMetadata.version`: `"26.0.0.7"`
   - `unitMetadata.java_version`: `"17.0.10"`
   - `unitMetadata.os_version`: `"Linux 5.14.0"`
   - `unitMetadata.is_containerized`: 0
   - `unitMetadata.feature_cdi`: 1
   - `unitMetadata.feature_jsonb`: 1
   - `unitMetadata.feature_mphealth`: 1

---

## 9. Complete Troubleshooting & FAQs

**Q1: What does the dashboard notice "Configure VM Manager connection" mean?**
**A1:** This notice only applies to contractual PVU/RVU core capacity pricing on hypervisors. Adoption telemetry metrics bypass PVU and are completely unaffected. You can safely ignore or disable the notice.

**Q2: Why did `upload_results.sh` fail with "Incorrect value of the DATA_UPLOAD_URL parameter"?**
**A2:** The script automatically prepends `"https://"` and appends `"/api/sam/v2/scan_results_upload"`. Ensure `DATA_UPLOAD_URL` contains only the hostname:port (e.g. `adoption-metrics-test.pl.eurolabs.ibm.com:443`).

**Q3: Why did the ETL log show "WARN: No metadata file declared in manifest in folder 1"?**
**A3:** This occurs when `.ums.data.json` is located in `/opt/ibm/wlp/` while `.ums.meta.json` is in `/opt/ibm/wlp/iso-swid/`. Fix by placing both files inside `/opt/ibm/wlp/iso-swid/`.

**Q4: Why does "Last Uploaded Day" in the Adoption Data report not match the scan date?**
**A4:** "Last Uploaded Day" is the date when LMT last transmitted data outbound to Software Central (Flow 2), not the local scan date.

**Q5: Why does "Inventory Match" show `<no data>`?**
**A5:** Either the SWID tag is missing from `iso-swid/`, or the `persistentId` in the SWID tag does not match `componentId` in `.ums.meta.json` (`"fbf6a96d49214c0abc6a3bc5da6e48cd"`).

**Q6: Why are some custom properties missing in Amplitude?**
**A6:** Any property sent in `.ums.data.json` must be explicitly listed in the `groupBy` whitelist array inside `.ums.meta.json`. Undeclared properties are stripped by LMT.

**Q7: How to handle file rotation on the endpoint?**
**A7:** When `liberty.ums.data.json` exceeds 100 KB, roll it to `liberty_1.ums.data.json` up to a maximum of 10 archive files. LMT scanner never deletes files on your VM.

**Q8: Why is the `/opt/ibm/scanner/output/` folder empty after running `upload_results.sh`?**
**A8:** `upload_results.sh` automatically removes uploaded `.tar.gz` packages upon successful upload to prevent disk space buildup. To obtain a package for manual inspection or web upload, re-run `./run_sw_and_pack.sh`.

---

## 10. Reference Links

* **IBM License Metric Tool Documentation:**
  https://www.ibm.com/docs/en/license-metric-tool/9.2.0

* **IBM Usage Central Production:**
  https://swc.saas.ibm.com

* **IBM Usage Central Test Instance (Sandbox):**
  https://sandbox.swc.saas.ibm.com

* **IBM Shared LMT Test Server:**
  https://adoption-metrics-test.pl.eurolabs.ibm.com

* **IBM Entitlement Key Management:**
  https://myibm.ibm.com/products-services/containerlibrary

* **IBM ReadyTool:**
  https://readytool.pl.eurolabs.ibm.com:9443/ReadyTool/login.xhtml

* **Segment Account Adoption Usage Standard:**
  https://segment-standards.prod.ddp.cis.ibm.net/events/account-adoption-usage

---

# Amplitude Setup

## Create an Amplitude Chart to View Liberty Usage Metrics (Sandbox)

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

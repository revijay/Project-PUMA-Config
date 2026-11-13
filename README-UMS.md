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
curl -k -L -w "\n%{http_code}\n" -X POST "https://${UMS_URL}/api/v1/event" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${UMS_TOKEN}" \
  -d '[{"instances":[{"instanceId":"153f0766267ec463797c19536ce33ab680eeff62c07cff53437eab2fb728c2af","metrics":[{"productId":"faee6552d7104cc49c4cc1222ff6ea32","productName":"Open Liberty","metricName":"INSTANCE_COUNT","metricQuantity":1,"metricType":"ADOPTION","measurementTimestamp":"2026-11-12T00:00:00Z"}]}]}]'
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
| Does the flat `/api/v1/event` payload give `ADOPTION`? | **[Tested]** No. UMS logged the converted record with `"metricType":"CONTRACTUAL"` and Amplitude showed `Account Contractual Usage`. |
| Does an ADOPTION event reach Software Central? | **[Tested]** Yes. Upload status `success`, 1 of 1 events processed. |
| Does it reach Amplitude? | **[Tested]** Yes, as the event `Account Adoption Usage`, about one minute after the upload. |
| Are unregistered metric names rejected? | **[Tested]** Not in the sandbox. A made-up name (`libertyAdoptionTest`) in the metricName field passed UMS, Software Central and Amplitude. **[Said]** names are a free choice, but check with core products. |
| Is a CR required? | **[Inference]** No. A payload-only ADOPTION event with a custom `instanceId` reached UMS, Software Central and Amplitude (Test 1), but the CR `open-liberty-adoption-test` still existed in the namespace. The payload had no `mappingId` and a different metric name, so the CR was almost certainly not used. Delete the CR and resend to confirm. A CR is still needed for breakdowns **[Said]**. |
| What if `metricType` is omitted in the new format? | **[Tested, UMS log]** UMS logs `MetricType unspecified in payload, will set MetricType to ADOPTION`. The flat format is different: UMS converts it to `CONTRACTUAL`. |
| Can version and edition be sent? | **[Said]** Yes, with the DB2 pattern (`breakdownDefs`). **[Open]**, see Test 2. |
| What `productId` is accepted? | **[Tested]** The ReadyTool Cloud Annotations `productID`. The 64-character SHA-256 hash failed in Software Central. |

## Background: Why the Initial Payload Did Not Work

### The hash as `productId`

Liberty initially sent a 64-character SHA-256 hash as `productId` in the flat payload. UMS accepted and exported these events, but they did not reach Amplitude, and Software Central showed `fixed_by_amendment`.

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


With the Open Liberty ID in the flat payload and `productMetric=VIRTUAL_PROCESSOR_CORE`, the event reached Amplitude as `Account Contractual Usage`. Software Central added `productCode`, `UT30`, `UT35` and `offerAccountingSystemCode`. That event is **contractual**, so this payload should not ship.

### `instanceId` in the flat payload

Adding `instanceId` to the flat payload made UMS reject it **[Tested]**:

```text
could not parse payload: no valid data field present in payload
```

This happened with the ReadyTool ID and with `productId=runtime-event`. The same payloads without `instanceId` were accepted. **[Said]** `instanceId` is valid in the new format at instance level, and the flat format is the legacy API. Why the flat format rejects it is **[Open]**. The UMS team asked for the failing payload. See Test 1.

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


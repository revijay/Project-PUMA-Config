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

## Export a Software Central Payload

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
5. Navigate to the **Software Components** section to see confirmed registration details:
   - **Component Name:** WebSphere Application Server Liberty
   - **Persistent ID:** `fbf6a96d49214c0abc6a3bc5da6e48cd`
   - **Product ID (PID):** `5724J08`
   - **Product Persistent ID:** `63a2a4dfc91f4485994c7aab94c55ab1`
   - **Tagging Status:** `TAG_CONFIRMED`
   - **Software tag:** `ibm.com_WebSphere_Application_Server_Liberty-26.0.0.swidtag`
6. Under **Component tags**, download the XML tag file or copy its content.

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
      "instanceId": "/opt/ibm/wlp"
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

```bash
python3 -c '
import json, datetime

# Current UTC timestamp
now = datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.000Z")
RS = "\x1e"  # ASCII 0x1E Record Separator

payload = [
  {
    "mappingId": "puma_liberty_adoption",
    "instances": [
      {
        "metrics": [
          {
            "measurementTimestamp": now,
            "metricQuantity": 1,
            "version": "26.0.0.7",
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
print(f"Generated liberty.ums.data.json with timestamp: {now}")
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
   tr -d '\x1e' < /opt/ibm/wlp/iso-swid/liberty.ums.data.json | jq .
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
1. Download the archive from VM to your Mac:
   ```bash
   scp 'root@<VM_IP>:/opt/ibm/scanner/output/*.tar.gz' ~/Downloads/
   ```
2. Log in to ILMT Console: https://adoption-metrics-test.pl.eurolabs.ibm.com
3. Go to **Management** -> **Scan Results Upload**.
4. Select the downloaded `.tar.gz` archive and click **Upload**.

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
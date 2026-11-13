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
  echo 'export DATA_UPLOAD_TOKEN="<token>"' >> /root/.bashrc
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

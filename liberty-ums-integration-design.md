# Liberty → Usage Metering Service (UMS) Integration: Design Draft

Status: DRAFT, for discussion. Updated 2026-11-12 after the meeting with the UMS team (section 5). Several decisions depend on open questions (section 10) and on Tests 1 and 2 (README, "UMS Adoption Metrics Testing").
Author: Reshmi Vijayan
Scope: Open Liberty / WebSphere Liberty adoption reporting through UMS to Software Central and Amplitude.

Conventions used in this document:
- **[Tested]** was observed in our own test runs on the project-puma cluster.
- **[Doc]** comes from IBM Usage Metering Guides pasted into our notes. The page URLs were not accessible to us directly.
- **[Said]** was stated by a person in Slack. Not independently verified.
- **[Inference]** is our reading and is not confirmed by anyone.

---

## 1. Goals and non-goals

### Goals
1. Send a small daily usage/adoption event from a running Liberty server to UMS when UMS is available.
2. Have that event reach Software Central and Amplitude, with Liberty's ReadyTool product ID.
3. Never affect server startup or runtime if UMS is absent or fails.
4. Keep customers able to opt out (mechanism to be agreed, see Q10).
5. Merging the design with the existing CVE reporting feature (see 8.6)

### Non-goals (for now)
- Reporting contractual licence metrics (PVU, VPC, SOCKET). Those are ILS/LMT territory.
- Liberty on VMs and bare metal. **[Said]** LMT is the tool for non-containerized environments, and **[Doc]** says a product that runs in both needs enabling for both tools.

---

## 2. Current state of the code (branch `liberty-ums-integration`)

Module: `dev/io.openliberty.reporting.internal`.

| Item | Current state |
|---|---|
| Scheduling | `FixReportingComponent` runs `ReporterTask` once a day, only if JVM property `cve.insight.enabled` is true. It is a kernel bundle. |
| Payload | Built by `DataCollector`: flat JSON with `productId`, `productName`, `productMetric`, `metricValue`, four empty Cloud Pak fields, and `logDate`. |
| `productId` | SHA-256 hex (64 chars) of install dir + edition + java.vendor + os.arch + os.name + hostname. |
| `productMetric` | The Liberty version string. |
| `metricValue` | Hardcoded `"1"`. |
| `productName` | Changed in this branch to `ProductInfo.getName()` (`Open Liberty` or `WebSphere Application Server`). Not validated downstream. |
| UMS URL | Hardcoded in `ReporterTask` (project-puma cluster). The `urlLink` server.xml property is commented out. |
| Auth | Bearer token in `CVEServiceClient`. **Replaced by a placeholder** (`<UMS_TOKEN_PLACEHOLDER>`). |
| TLS | Trust-all `TrustManager` and a hostname verifier that accepts everything. Test only. |
| Diagnostics | Temporary `Tr.info` payload log in `CVEServiceClient.retrieveCVEData`. Must be removed before any PR. |
| Response handling | `ReporterTask` logs the HTTP status only at debug level. UMS returned 200 even for events it later rejected. |

Known gaps: unit tests and the commented-out `CVEDataTest` still assert the old payload shape and `productName == "Liberty"`.

---

## 3. What we tested

All against the project-puma UMS (flat `POST /api/v1/event`).

### 3.1 The hash as `productId` does not work end to end
| `productId` | UMS | Software Central / Amplitude |
|---|---|---|
| `runtime-event`, `version-event`, `test-product-123` | accepted | reached Amplitude **[Tested]** |
| Liberty hash prefix, 16 / 32 / 40 chars | accepted | reached Amplitude **[Tested]** |
| Liberty hash prefix, 48 / 56 / 63 chars | accepted and exported | not in Amplitude **[Tested]** |
| Full 64-char hash | accepted and exported | status `fixed_by_amendment`, not in Amplitude **[Tested]** |
| 41 to 47 chars | not tested | not tested |

Only `productId` differed between working and failing payloads. Whitespace, headers, redirects, `productMetric` and `logDate` were ruled out. An allow-list theory was disproved when `test-product-123` arrived.

Software Central contact (Mari Wade) **[Said]**: `productId` must be a valid ReadyTool product ID or part number, and a repeated `eventId` is treated as an amendment.

### 3.2 The ReadyTool ID works end to end
Open Liberty ReadyTool entry: Cloud Annotations `productID` `faee6552d7104cc49c4cc1222ff6ea32`. Status REVIEWED, "End-to-end validation with LS or UMS: Not done". Declared metrics: PVU, SOCKET, VPC (standalone, no Cloud Pak).

Payload sent **[Tested]**:
```json
{"productId":"faee6552d7104cc49c4cc1222ff6ea32","productName":"Open Liberty TEST","productMetric":"VIRTUAL_PROCESSOR_CORE","metricValue":"1","productCloudpakRatio":"","cloudpakId":"","cloudpakName":"","cloudpakMetric":"","logDate":"2026-11-05"}
```
Result: reached Amplitude for 2026-11-05. Software Central added `productCode 5724J08` (PID), `UT30`, `UT35` and `offerAccountingSystemCode`, which the hash events never got. `productTitle` stayed as we sent it. The 2026-11-06 event (period not yet ended) was not checked.

### 3.3 `instanceId` in the flat payload is rejected
Adding `instanceId` (the 64-char hash) was rejected with `could not parse payload: no valid data field present in payload`. This happened with both the ReadyTool ID and `productId=runtime-event`. The same payloads without `instanceId` were accepted. Cause not known. **[Said]** (meeting) the flat format is the legacy API, and `instanceId` belongs in the new format at instance level. The UMS team asked for the failing payload.

### 3.4 ADOPTION via CR + `mappingId` works end to end **[Tested]**
A CR (`IBMServiceMeterDefinition`, `metricType: ADOPTION`, `processing: LAST`) plus a payload with only `mappingId`, `measurementTimestamp` and `metricQuantity` was accepted by UMS, got Software Central status `success` (1 of 1 events) and reached Amplitude as `Account Adoption Usage` (`productCode 5724J08`, `unit` = the metric name, `instanceId` = cluster plus namespace).

### 3.5 Payload-only ADOPTION with a custom `instanceId` works end to end **[Tested]**, CR-free part **[Inference]**
Payload (no `mappingId`): `[{"instances":[{"instanceId":"test-instance-1","metrics":[{"productId":"faee…","productName":"Open Liberty","metricName":"INSTANCE_COUNT","metricQuantity":1,"metricType":"ADOPTION","measurementTimestamp":"2026-11-11T00:00:00Z"}]}]}]`.
- **Caveat:** the CR `open-liberty-adoption-test` still existed in the namespace during this test. The CR was almost certainly not used: the payload has no `mappingId`, the metric name differs from the CR's, and `processing` was `MAX` (UMS default) instead of the CR's `LAST`. Not proven until the CR is deleted and the payload resent (planned, `test-instance-nocr`).
- UMS stored it (`processing` defaulted to `MAX`). The export has `clusterId` = `<cluster host>, test-instance-1`. The part before the comma is added by UMS and appears to be namespace plus cluster domain **[Inference]**.
- Software Central: `success`. Amplitude: `Account Adoption Usage` with `instanceId` = `<cluster host>, test-instance-1`, `unit` = `INSTANCE_COUNT`, `productCode 5724J08`.
- The same payload dated the day of sending (2026-11-12) was `success` in Software Central but was not in Amplitude at the time of checking. Likely an unfinished-day delay (**[Inference]**, recheck after 11-13 UTC).
- **`metricType` default [Tested, UMS log]:** a payload without `metricType` (`instanceId` `test-instance-notype`, date 2026-11-10) was accepted and UMS logged `MetricType unspecified in payload, will set MetricType to ADOPTION`. The new format defaults to ADOPTION, while the flat format is converted to CONTRACTUAL. The stored value was not checked. The client should still send `ADOPTION` explicitly.
- Details are in the README, Test 1.

### 3.6 Other observations
- UMS converts the flat payload into a `data` object (`measurementTimestamp`, `metricName`, `metricQuantity`, `metricType`, `productId`, `productName`) and sets `metricType` to **CONTRACTUAL**. **[Tested]**
- UMS logs: "InstanceID unspecified in payload, will set InstanceID to the name of the namespace", and "Processing unspecified, will set Processing to MAX". In Amplitude, `instanceId` was the cluster plus namespace. **[Tested]**
- A literal `<...>` in `logDate` was rejected (timestamp parse error). Plain dates parse fine.
- One UMS 500 occurred with `logDate` 2026-11-04 while UTC was still 11/03. Cause not verified. **[Inference]** a date/timezone link.
- Amplitude's `device_id` equals the Software Central correlation ID of the upload, so the Amplitude "user" follows the upload, not the account. **[Tested]**
- The Software Central list `multiStatus` result means a mix of per-event results. Old events in an unfiltered export are re-sent and get their own statuses.

---

## 4. What the documentation says **[Doc]**

1. **Deployment.** UMS is installed together with the product, one per namespace, in the same namespace as the product operands. An `IBMUsageMetering` CR creates it. It supports OLM and Helm on OpenShift **and CNCF Kubernetes**, and CASE for air-gapped installs.
2. **Modes.** STANDARD mode covers usage-based metrics (everything except PVU/VPC/GPGPU). PVU/VPC/GPGPU need the Helm install with the ILS sidecar (`PROCESSOR_CAPACITY_ENABLED`).
3. **Two-part metric model.** A registration CR (`IBMServiceMeterDefinition`) holds static data (`instanceId`, `productId`, `productName`, `metricName`, `metricType`, `processing`, `mappingId`). The JSON payload carries `mappingId` and `metricQuantity`, plus any breakdown attributes. Combined size limit is 100 KB (HTTP 400 above it).
4. **`metricType`** is `CONTRACTUAL` or `ADOPTION`. ADOPTION metrics are not shown in compliance reports or the audit snapshot, are accessible through the UMS API, and dashboards in LSR / Software Central are described as "planned".
5. **`processing`** controls daily aggregation: `MAX`, `ADD`, `LAST`, `COUNT`.
6. **`instanceId`** is optional and defaults to the namespace.
7. **Discovery and credentials.**
   - Endpoint URL: ConfigMap `ibm-usage-metering-upload-config`, key `url`. In-cluster service: `ibm-usage-metering-instance.<namespace>.svc.cluster.local:8080`.
   - Token: Secret `ibm-usage-metering-upload-token`, key `token`.
   - TLS certificate: same ConfigMap, key `tls.crt`, used for server verification.
   - The doc names the URL ConfigMap inconsistently (`ibm-usage-metering-service-upload-config` vs `ibm-usage-metering-upload-config`). Confirm the real name.
8. **Frequency.** At least once a day.
9. **Verification.** `GET /api/v1/raw` returns stored data.
10. **Onboarding (usage-based metrics).** Register in ReadyTool, create a Clearing House dependency on UMS, download annotations, install UMS in the namespace, instrument the product, update docs, then run the end-to-end validation (audit snapshot .zip uploaded to ReadyTool; one snapshot per flavor).
11. **Annotations** (productID, productName, productMetric, productChargedContainers) go on the Kubernetes resources that deploy the container, not in runtime code.
12. **Adoption metrics guide** says adoption metrics are "sent to Usage Central and forwarded to Amplitude", and that non-containerized products use LMT. This conflicts with item 4 about dashboards being planned.

---

## 5. What the teams said **[Said]**

### 5.1 UMS team meeting, 2026-11-12 (Tomasz Zeller, Dariusz Permus, Michał Paluch)
From the transcript. Not yet checked against the adopter's guide, which we have not seen.

| Topic | Statement |
|---|---|
| `productId` | Use the ReadyTool ID. It is the reconciliation key (entitlement, CCID, IBM taxonomy). |
| Flat payload | Legacy ILS-compatible API, CONTRACTUAL by design, for products migrating from ILS. Not for new work. |
| CR | Not required. Needed only for hierarchical metrics. Payload-only works in the new format. Recommended because static data lives in the CR and payloads stay small. No CR-free example in the docs. |
| `instanceId` | Free string, no validation, default the namespace. Defined by our own logic. Sent in the payload at instance level: `[{"instances":[{"instanceId":"...","metrics":[{"productId":...,"metricName":...,"metricQuantity":...,"metricType":...}]}]}]`. |
| Aggregation | By `processing`: `MAX` largest value, `ADD` sum of the day's payloads. |
| Who deploys UMS | The adopting product owns, installs, upgrades, configures it. One per namespace is typical. One UMS can cover several namespaces. Two products in one namespace share one. |
| Metric names | Free choice, no unified adoption names. Check with core products. |
| Amendments | Data accepted up to 7 days back. A changed day is re-uploaded with the same event ID. They have not tested it. |
| API spec | No formal OpenAPI. The adopter's guide is the reference. |
| Opt-out | UMS does not control the upload. The Software Central flag in the `IBMUsageMetering` CR or Helm values does. The IBM entitlement key must be configured. The product sets the flag per the customer's choice. |
| Version and edition | DB2 pattern: CR with `breakdownDefs` and `groupBy` attributes (`version`, `cloud_provider`, `is_containerized`, `using_feature_*`), `processing: LAST`, `repeat: true`, `metricName: INSTANCE_COUNT`, `data.componentId/componentName`. Payload has `mappingId`, `measurementTimestamp`, `metricQuantity: 1` and the attributes as plain fields. Meaning of `repeat` unknown. |
| Our test payload | A version-like value was in `metricName`. That is the wrong place. |
| Offer | Send them a spreadsheet of the desired landscape (version, deployment type, licence ...) and they help shape the payload. |

### 5.2 Earlier Slack statements

| Who | Statement |
|---|---|
| UMS team (Michał Paluch) | UMS should be part of the product, not standalone. Customers should not install it. About 20 MB in standard mode, over 500 MB with capacity metrics (ILS sidecar). ConfigMap limit 1 MB; daily aggregates only; retention 2 years by default. |
| MAS (Ben Bakowski) | UMS is a shared deployment on the OCP cluster. MAS automation deploys it. MAS works without UMS, but the customer may be non-compliant. Air-gap mode supported. Packaging pattern: to be determined. |
| CP4D (Praveen Kumar) | UMS is deployed separately, namespace scoped, can run independently of CPFS. Likely REST. Ask Tomasz Zeller / Dariusz Permus for details. |
| Software Central (Mari Wade) | `productId` must be a ReadyTool ID or part number; repeated `eventId` is an amendment. |

Reconciling: this is one model seen from different sides. UMS is deployed per namespace alongside the product (Doc), by the product's own automation (Ben, Michał), and, as the meeting confirmed, the adopting product owns, installs, upgrades and configures it. For Liberty, who does this is still open (section 8.5).

---

## 6. Constraints that shape the design

1. `productId` is the fixed ReadyTool ID per edition (Open Liberty vs WebSphere Liberty differ; `63a2a4dfc91f4485994c7aab94c55ab1` is the WebSphere Liberty 26.0 entry we saw). The hash can no longer be `productId`.
2. Liberty events are adoption signals. They should be `metricType: ADOPTION`, not `CONTRACTUAL`. The flat format is CONTRACTUAL by design **[Said]** and must not be used. `VIRTUAL_PROCESSOR_CORE` with value 1 must not ship.
3. `productName`, `productId`, `metricName` come from ReadyTool. `Open Liberty TEST` was a test label only.
4. Instance identity: `instanceId` is a free string defaulting to the namespace. It can be set in the payload at instance level **[Tested]** or, per the docs, in the CR. UMS and Amplitude show it as `<namespace + cluster domain>, <instanceId>` (prefix added by UMS, **[Inference]**). What Liberty puts there is our own decision (section 7).
5. No hardcoded endpoint or token, and no trust-all TLS.
6. The JVM must not create Kubernetes resources such as the CR.

---

## 7. Instance identity options

The original goal, from the UFO design: the ID should identify unique Liberty instances and not change with Liberty version or feature set. The current hash meets the second part.

Weaknesses of the hash **[Inference from code]**: none of the inputs are server-specific, so servers in one install and host share a hash; install dir/Java vendor changes alter it; hostname is a pod name in Kubernetes and changes on restart (untested); it describes the environment more than a server.

| Option | Pros | Cons / unknowns |
|---|---|---|
| A. Namespace as the instance (UMS default, no ID from Liberty) | No client work, supported by docs | Servers in a namespace merge. Distinct server counts are lost unless we use a breakdown |
| B. `IBMServiceMeterDefinition.data.instanceId` set by the deployer (operator/Helm) | Documented. Stable. No JVM storage | Needs deployment tooling. One CR means one instance, not one per server |
| C. Per-server UUID from `WsLocationAdmin.getServerId()` (`${wlp.server.uuid}`) | Existing API. Per server | **[Inference from code]** reset by a clean start. Not persistent in recreated containers without a volume. Unused by the reporting bundle today |
| D. New persistent UUID per server | Stable if persisted | Needs a storage location. Cloning copies the ID. Read-only filesystems. Container recreation without a volume gives a new ID |
| E. Report a count by breakdown (version, edition, platform) with `processing: COUNT` | Fits the documented model | **[Inference]** counts reports, not distinct servers |

Open design decision: what is "one instance" for Liberty: an installation, a running server (name + user dir), or the namespace? This is a Liberty product decision, not a UMS question. Recommendation (judgement, not confirmed): count one running server, and use A or B for the UMS instance until the product decision is made.

Update after the meeting and Test 1: `instanceId` is a free string that we define, sent in the payload, and a custom value reaches Amplitude **[Tested]**. Options C and D are therefore feasible without a CR (CR-free part still **[Inference]**, see 3.5). Hash vs UUID is a privacy and stability question, not a UMS limitation. Whether the install-based hash is acceptable as an identifier needs the privacy review (section 10, Q15). Each distinct `instanceId` becomes its own series in Amplitude, so an ID that changes on restart would inflate counts.

---

## 8. Proposed design

### 8.1 Overview
```
Liberty JVM ──(daily HTTPS POST, bearer token)──> UMS (same namespace) ──> Software Central ──> Amplitude
      ▲                                                  ▲
      │ endpoint, token, tls.crt from mounted            │ IBMServiceMeterDefinition CR
      │ ConfigMap / Secret (or system properties)        │ created by operator / Helm (static data)
```
Containers only. VMs and bare metal use LMT.

### 8.2 Registration CR (optional, static)
**[Said]** A CR is not required. Two variants:
- **Payload-only (Test 1, passed with the CR caveat in 3.5):** all fields, including `instanceId` and `productId`, travel in each payload. `metricType` defaults to ADOPTION if omitted **[Tested, UMS log]**, but send it explicitly. Nothing to deploy beyond UMS. `processing` defaults to `MAX`. Whether it can be set in the payload is open.
- **CR + `mappingId`:** needed for breakdowns (version, edition). Tested for plain ADOPTION. The breakdown form (Test 2) is pending.

Illustrative CR below. The `breakdownDefs` field names are not verified against the CRD.
```yaml
apiVersion: operator.ibm.com/v1
kind: IBMServiceMeterDefinition
metadata:
  name: open-liberty-adoption
  namespace: <product namespace>
spec:
  mappingId: <stable GUID for Liberty>
  processing: LAST             # DB2 pattern; MAX/ADD also possible
  data:
    productId: faee6552d7104cc49c4cc1222ff6ea32
    productName: Open Liberty  # exact ReadyTool name
    metricName: INSTANCE_COUNT # free choice; check with core products
    metricType: ADOPTION
    # instanceId: optional; defaults to namespace
  breakdownDefs:               # unverified field names, see README Test 2
    - groupBy:
        - libertyVersion
        - edition
```

### 8.3 Payload (dynamic, sent by Liberty)
Two shapes. The first passed Test 1, the second is pending Test 2.

Payload-only (Test 1, passed; CR-free confirmation pending):
```json
[{"instances":[{"instanceId":"<id>","metrics":[{
  "productId":"faee6552d7104cc49c4cc1222ff6ea32","productName":"Open Liberty",
  "metricName":"INSTANCE_COUNT","metricQuantity":1,"metricType":"ADOPTION",
  "measurementTimestamp":"2026-11-12T00:00:00Z"}]}]}]
```

CR + `mappingId` with breakdown attributes (Test 2):
```json
[{
  "mappingId": "<same GUID>",
  "instances": [{
    "metrics": [{
      "measurementTimestamp": "2026-11-11T00:00:00Z",
      "metricQuantity": 1,
      "libertyVersion": "26.0.0.12",
      "edition": "Open"
    }]
  }]
}]
```
Static data (name, product ID) moves to the CR, which keeps the payload small.

### 8.4 Client changes
1. Replace the hardcoded URL with configuration. Read the URL, token and `tls.crt` path from system properties or environment variables (set from the mounted ConfigMap and Secret). No default value.
2. If not configured, do not send. No warning, no error. This also covers VMs.
3. Remove the trust-all `TrustManager` and the hostname verifier. Verify using `tls.crt`.
4. Short connect/read timeouts. Never block startup. No retry on the startup path.
5. Log the response code at a useful level and treat non-2xx as a warning, without FFDC noise.
6. Build the documented payload (8.3). Drop the Cloud Pak fields and the hash.
7. Compute `logDate`/`measurementTimestamp` in UTC. The current `LocalDate.now()` uses local time.
8. Remove the temporary `Tr.info` payload log.
9. Update unit tests, `CVEDataTest`, `CVEHappyPathTest`, and the FAT mock endpoints to the new format. Add an assertion on the UMS response, since `testDataSentToUMSEndpoint` passes even on a 500.

### 8.5 Deployment side (outside the JVM)
**[Said]** The adopting product owns, installs, upgrades and configures its UMS. For Liberty that means we must decide who does it.
- A ReadyTool "New Containerized Offering" entry for each flavor. The Open Liberty entry exists but lists only PVU, SOCKET, VPC.
- Annotations on the Liberty pod/workload resources.
- Install and configure UMS (`IBMUsageMetering` CR or Helm), including the Software Central flag and entitlement key. The flag is the opt-out mechanism **[Said]**. Candidate owners: Open Liberty Operator, Liberty Helm chart or sample manifests **[Inference]**.
- A CR (only if breakdowns are used) and a way to mount the URL/token/cert into Liberty pods.
- A Clearing House dependency on UMS.
- End-to-end validation with an audit snapshot (applicability to adoption-only products: Q6).

---

## 10. Open questions

Status column: **Answered** (by test), **Said** (meeting, unverified), **Open**.

| # | Question | Status | Who |
|---|---|---|---|
| Q1 | Do ADOPTION metrics reach Software Central and Amplitude? | **Answered**: yes, as `Account Adoption Usage` (CR + `mappingId`). Dashboards in Software Central still unknown. | - |
| Q2 | Which `metricName` do adoption metrics use, and do they need registering in ReadyTool? | **Said**: free choice, check with core products. ReadyTool registration still **Open**. | Core products / ReadyTool |
| Q3 | Who creates the CR and installs UMS for Liberty? | **Said**: the adopting product. Which Liberty component is **Open**. | Liberty team, operator team |
| Q4 | Is the flat payload supported? Why was `instanceId` rejected there? | **Said**: legacy, CONTRACTUAL, do not use. Rejection cause **Open**; failing payload to be sent. | UMS team |
| Q5 | How to send version/edition/platform? | **Said**: DB2 breakdown pattern. Whether it reaches Amplitude is **Open** (Test 2). | UMS team |
| Q6 | What does Step 9 (audit snapshot validation) require for an adoption-only product? Does Open Liberty (EPL) have to complete it? One snapshot per flavor? | **Open** | UMS / ReadyTool |
| Q7 | Opt-out? | **Said**: the Software Central flag in the UMS CR/Helm values, set by the product. Default for Liberty and the legal items are **Open**. | Liberty team, legal |
| Q8 | One UMS for several products? | **Said**: yes, shared in a namespace. Resolved. | - |
| Q9 | Separate UMS reporting from CVE reporting? | **Open** | Liberty team |
| Q10 | What counts as one Liberty instance (install, server, namespace)? | **Open** | Liberty team / product management |
| Q11 | Timestamp rules: UTC, future dates, period not ended? | **Said**: 7-day back window. UTC and future dates **Open**. | UMS / Software Central |
| Q12 | Real name of the URL ConfigMap. | **Open** | UMS team |
| Q13 | ILMT on VMs; does the contractual requirement apply to Open Liberty (EPL)? | **Open** | LMT team |
| Q14 | Late or repeated events for the same day; `eventId` collisions. | **Said**: re-uploaded with the same event ID. Untested. | Software Central |
| Q15 | Is the install-based hash acceptable as an identifier under the on-prem telemetry privacy rules (no identifiable data)? | **Open** | Privacy / legal |
| Q16 | Does a custom `instanceId` in the payload reach Amplitude? Is a CR-free payload accepted? | **Answered**: yes to the first (Test 1). CR-free is **[Inference]** because a CR existed in the namespace. Delete it and resend to confirm. | Us |
| Q18 | Is an event for the current, unfinished day held back from Amplitude (11-12 upload did not appear, 11-11 did)? Can `processing` be set in a payload-only request? | **Open** | UMS / Software Central |
| Q17 | Do breakdown attributes (version, edition) reach Amplitude? What does `repeat: true` do? | **Open** (Test 2) | Us, UMS team |

Documents to request: the adopter's guide (shared in the meeting chat), the DB2 examples, the Amplitude representation page, the UMS overview and API reference (`/api/v1/event`, `/raw`, `/swc`), the contractual-metrics guide, the LMT adoption guide, the ReadyTool onboarding and snapshot validation guide, the air-gap guide, the UMS security/auth guide. The `pages.github.ibm.com` pages need w3id login, so someone must paste their text.

---

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Metric names unregistered in ReadyTool | Events dropped or unvalidated in production (sandbox accepted a made-up name) | Resolve Q2, register early |
| Instance ID taken from the install-based hash | Privacy objection, and unstable in containers | Resolve Q10, Q15 |
| Opt-out not defined | Legal blocker | Q7, legal checklist |
| Event counts reports, not servers | Wrong adoption numbers | Decide Q10, choose option B/C/D or accept |
| Wrong metric type (flat events are typed CONTRACTUAL) | Events could count as licence usage | Use the new format with `metricType: ADOPTION` sent explicitly (the new-format default is ADOPTION, but may change) |
| Open Liberty (EPL) has no licence | May fall outside the process | Resolve Q6, Q13 |
| UMS absent or down | Errors, noise | Send only when configured; failures never affect the server |
| Token exposure | Unauthorized uploads | Rotate, config-based secrets |
| Dates in local time | Wrong or rejected days | Use UTC |
| Test-only code shipped | Security, correctness | Section 9 checklist |

---

## 12. Work breakdown

1. Run Test 1 (payload-only, custom `instanceId`) and Test 2 (breakdown). Send the UMS team the failing flat payload, the working CR and payload, the test results and the metrics spreadsheet (`Metrcis.xlsx`, to be reviewed). Request the documents above.
2. Decide instance scope (Q10), identifier source (Q15), and CVE separation (Q9) in the Liberty team.
3. ReadyTool: add adoption metrics and flavors; Clearing House dependency; annotations.
4. Decide who installs UMS and sets the Software Central flag (operator/Helm), plus the opt-out and legal items (Q7).
5. Client rework (8.4) and tests, on a clean branch.
6. Deploy UMS in a test namespace on a real cluster, send events, check `/api/v1/raw`, export to Software Central sandbox, confirm Amplitude.
7. Audit snapshot and ReadyTool validation (Step 9).
8. Security cleanup (section 9) and fresh branch for the PR.

---

## 13. Evidence index

- Software Central uploads: 11/03 `dataUpload` correlation `2fc68a7a-3a18-5cf3-8522-dd49cabc7387` (`fixed_by_amendment`, eventId `11838f81…bb59`); 11/04 length batch `5ba58874-34e6-5f03-aec4-fd5b2823595a`; manual upload 11/06 correlation `4abe5fe3-cd71-5be5-9c37-e587f491907c` (ReadyTool ID event `0c01c7e9…` reached Amplitude).
- UMS pod log lines: "Rejecting data … no valid data field present in payload"; "InstanceID unspecified in payload, will set InstanceID to the name of the namespace".
- Code: `DataCollector`, `ReporterTask`, `CVEServiceClient`, `FixReportingComponent`, `HashUtils`, `ProductInfo`, `WsLocationAdmin.getServerId()`.

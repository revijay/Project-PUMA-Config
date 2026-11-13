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
2. Search for and select: `Account Adoption Usage`.

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

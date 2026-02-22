Option A – Recommended: Use Dynatrace Deployment Event (AI does comparison automatically)

⚙️ Option B – Manual Comparison: Pull metrics before & after release and compare via API

✅ OPTION A – Deployment Marker (Recommended)

This uses the Dynatrace Events API v2 and lets Dynatrace automatically compare performance before/after release.

🧩 Architecture
TFS Release → PowerShell Script → Dynatrace Events API
                                      ↓
                           Deployment Marker Created
                                      ↓
                       Dynatrace AI (Davis) compares baseline
                                      ↓
                         Alert if degradation detected
STEP 1 – Generate API Token in Dynatrace

Go to Dynatrace → Access Tokens

Create new token

Enable:

events.ingest

entities.read

Copy the token

Store it securely in TFS variable group (secret)

STEP 2 – Find the Service Entity ID

You must identify the exact service Dynatrace monitors.

Method A – UI

Go to Services

Open your application service

Copy the SERVICE-XXXXX ID from URL

Method B – API
GET /api/v2/entities?entitySelector=type(SERVICE),entityName.equals("YourServiceName")

Save the returned entity ID.

STEP 3 – Test Deployment Event in Postman

Endpoint:

POST https://{your-env}.live.dynatrace.com/api/v2/events/ingest

Body:

{
  "eventType": "CUSTOM_DEPLOYMENT",
  "title": "Production Deployment",
  "entitySelector": "entityId(SERVICE-XXXXXXX)",
  "properties": {
    "ReleaseVersion": "1.0.5",
    "Environment": "Production",
    "TriggeredBy": "TFS"
  }
}

If successful:

You’ll see event in service timeline

Dynatrace correlates issues automatically

STEP 4 – Add Step in TFS Release Pipeline

In your Production stage:

Add PowerShell task

Add script:

$token = "$(DynatraceToken)"
$envUrl = "https://your-env.live.dynatrace.com"
$serviceId = "SERVICE-XXXXXXX"

$body = @{
  eventType = "CUSTOM_DEPLOYMENT"
  title = "Production Deployment - $(Release.ReleaseName)"
  entitySelector = "entityId($serviceId)"
  properties = @{
    ReleaseVersion = "$(Build.BuildNumber)"
    Environment = "Production"
  }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "$envUrl/api/v2/events/ingest" `
  -Method POST `
  -Headers @{Authorization="Api-Token $token"; "Content-Type"="application/json"} `
  -Body $body
STEP 5 – Validate Behavior

After deployment:

Check Service → Timeline

Confirm deployment marker appears

Monitor:

Response time

Failure rate

Throughput

If performance degrades:

Dynatrace automatically opens a Problem

It links problem to deployment event

🎯 What Dynatrace Automatically Does

Baseline from historical data

AI anomaly detection

Correlation between release and degradation

Alerting

No manual comparison required.

⚙️ OPTION B – Manual Metric Comparison (Advanced / Not Recommended)

Here you manually:

Capture performance before release

Capture performance after release

Compare values

Trigger alert if threshold exceeded

🧩 Architecture
Before Release → Fetch Metrics (API)
After Release → Fetch Metrics (API)
Compare Values → If Degraded → Send Alert
STEP 1 – Identify Metrics

Using Dynatrace Metrics API:

GET /api/v2/metrics/query

Example metrics:

builtin:service.response.time

builtin:service.errors.server.rate

builtin:service.requestCount.total

STEP 2 – Fetch Baseline Before Release

In TFS:

Add task BEFORE deployment:

$baselineStart = (Get-Date).AddMinutes(-60).ToString("o")
$baselineEnd = (Get-Date).ToString("o")

Call:

GET /api/v2/metrics/query?metricSelector=builtin:service.response.time:avg&entitySelector=entityId(SERVICE-XXXX)&from=baselineStart&to=baselineEnd

Store result in file or pipeline variable.

STEP 3 – Deploy Application

Normal production release.

STEP 4 – Wait Monitoring Window

Wait 15–30 minutes after deployment.

STEP 5 – Fetch Post-Release Metrics

Same API call but new time window:

$postStart = (Get-Date).AddMinutes(-15).ToString("o")
$postEnd = (Get-Date).ToString("o")
STEP 6 – Compare Values

Example logic:

if ($postResponseTime -gt ($baselineResponseTime * 1.20)) {
    Write-Host "Performance degraded by >20%"
}

You define thresholds:

20% slower

Error rate doubled

Throughput drop 30%

STEP 7 – Trigger Alert

If degraded:

Option 1:

Send email from TFS

Option 2:

Call Dynatrace API to create custom problem event

📊 Comparison Between Options
Feature	Option A	Option B
Implementation effort	Low	High
Maintenance	Low	High
AI-based detection	Yes	No
False positives	Low	Possible
Scalable	Yes	Complex
Recommended	✅ Yes	❌ No
🧠 Professional Recommendation

Use Option A (Deployment Marker).

Why?

Dynatrace’s AI (Davis engine) already:

Builds baselines

Detects anomalies

Correlates deployment

Reduces noise

Opens problems automatically

Option B duplicates what Dynatrace already does — but with more complexity.

🚀 What I Recommend You Do

Implement Option A first

Validate automatic detection

Only consider Option B if:

Management requires fixed % threshold comparison

You need release validation gate in pipeline

If you'd like, next I can provide:

A complete production-ready PowerShell script

A security best-practice setup

Or a design document you can share with your manager

Rephrase

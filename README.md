# BigQuery Data Agent Cross-Project Debug

Date: 2026-03-10

## TL;DR

### The problem
A BigQuery data agent could be created successfully when the **agent lived in Project A and read a BigQuery table in Project B**, but the reverse direction failed. At first glance this looked like a **cross-project BigQuery limitation**.

It was **not**.

The actual blocker was simpler: **the agent project must have `geminidataanalytics.googleapis.com` enabled**. If that API is disabled, agent creation can fail with **HTTP 403 `SERVICE_DISABLED`** before any real cross-project BigQuery permission check happens.

### The goal
Make a data agent in one Google Cloud project query BigQuery data stored in another project, and understand exactly:

1. what must be enabled
2. which IAM roles are needed
3. which service agent appears
4. what success and failure look like in practice

### What readers need to do to make it work
Assume:
- **Project A** = the **agent project** (where the data agent is created)
- **Project B** = the **data project** (where the BigQuery table lives)

You need **all** of the following in **Project A**:

1. Required APIs enabled:
   - `geminidataanalytics.googleapis.com`
   - `cloudaicompanion.googleapis.com`
   - `bigquery.googleapis.com`

2. Billing enabled on Project A.

3. The calling principal must have enough IAM in Project A to create and use the agent.

4. The calling principal must have data access on the BigQuery table in Project B.

5. On first successful creation/use, Google will provision the Gemini Data Analytics **service agent** in Project A.

### Copy/paste setup commands
Replace the placeholder values first, then run the blocks as-is.

```bash
export PROJECT_A="your-agent-project-id"
export PROJECT_B="your-data-project-id"
export PRINCIPAL="user:your.name@example.com"
# For service accounts, use this form instead:
# export PRINCIPAL="serviceAccount:your-sa@${PROJECT_A}.iam.gserviceaccount.com"
```

Enable the required APIs in the **agent project**:

```bash
gcloud services enable geminidataanalytics.googleapis.com cloudaicompanion.googleapis.com bigquery.googleapis.com --project="$PROJECT_A"
```

If you also want the reverse direction to work, enable the same APIs in **Project B** too:

```bash
gcloud services enable geminidataanalytics.googleapis.com cloudaicompanion.googleapis.com bigquery.googleapis.com --project="$PROJECT_B"
```

Grant the most important project-level roles in **Project A** to the calling principal:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/geminidataanalytics.dataAgentCreator"

gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/geminidataanalytics.dataAgentStatelessUser"

gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/cloudaicompanion.user"

gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/bigquery.user"

gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/datacatalog.catalogViewer"
```

If the same principal also needs to enable APIs, grant this too:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/serviceusage.serviceUsageAdmin"
```

For BigQuery source access in **Project B**, grant at least read access on the dataset or table being used as the knowledge source. Example at the project level:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_B" \
  --member="$PRINCIPAL" \
  --role="roles/bigquery.dataViewer"
```

Verify API enablement quickly:

```bash
gcloud services list --enabled --project="$PROJECT_A" | grep -E 'geminidataanalytics|cloudaicompanion|bigquery'
gcloud services list --enabled --project="$PROJECT_B" | grep -E 'geminidataanalytics|cloudaicompanion|bigquery'
```

### IAM / permissions readers should expect to need
Based on Google Cloud docs and the live test in this repo:

In **Project A** (agent project), the calling principal should have:
- `roles/geminidataanalytics.dataAgentCreator` — create own data agents
- `roles/geminidataanalytics.dataAgentStatelessUser` — interact with data agents / stateless chat flows
- `roles/cloudaicompanion.user` — required to create Google-managed conversations
- `roles/bigquery.user` — run BigQuery jobs
- usually `roles/datacatalog.catalogViewer` — needed by the data agent flow per docs

In **Project B** (data project), the calling principal should have access to the knowledge source table, typically:
- `roles/bigquery.dataViewer` on the dataset/table being used

If the table uses extra controls, readers may also need:
- policy-tag access for column-level access control
- row access policies
- masked-reader access for masked columns

If the same person also needs to enable APIs, they need:
- `roles/serviceusage.serviceUsageAdmin`

### Service agent readers should expect
After the first successful data-agent creation in the agent project, expect Google to provision a project-level service agent that looks like this:

```text
service-PROJECT_NUMBER@gcp-sa-geminidataanalytics.iam.gserviceaccount.com
```

With IAM role:

```text
roles/geminidataanalytics.serviceAgent
```

If `geminidataanalytics.googleapis.com` is disabled, you should **not** expect this service agent to appear yet.

### What success looks like
When correctly configured:
- the agent can be created in **Project A**
- the knowledge source can point to a BigQuery table in **Project B**
- a question asked to the agent can generate SQL against the Project B table
- the BigQuery job itself can run under **Project A**
- temporary results can be written in **Project A**

### What failure looks like
If the API is disabled in the would-be agent project, expect something like:
- HTTP `403`
- reason: `SERVICE_DISABLED`
- failure happens **before** a meaningful cross-project dataset permission check

### Docs reference
Google Cloud docs that match the findings here:
- Create data agents: https://docs.cloud.google.com/bigquery/docs/create-data-agents
- Conversational Analytics API overview: https://docs.cloud.google.com/gemini/data-agents/conversational-analytics-api/overview
- Enable the API: https://docs.cloud.google.com/gemini/data-agents/conversational-analytics-api/enable-the-api

---

## Complete Test Results

This section preserves the full observed behavior so readers know what to expect when reproducing the setup.

## Goal of the test
Verify whether a BigQuery data agent can be created in one project while referencing BigQuery tables in another project, using user ADC credentials.

For privacy/public-sharing purposes, the real project IDs are abstracted here:
- **Project A** = the first project tested as the working agent project
- **Project B** = the second project tested as the failing agent project

The live behavior documented below is real; only the project identifiers are anonymized.

## Test environment
Authentication mode used:
- user ADC credentials
- `GOOGLE_APPLICATION_CREDENTIALS` unset during testing

Conceptual topology:
- **Project A** hosted the working data agent
- **Project B** hosted one of the BigQuery source tables used in the successful cross-project test

## Key finding
**Cross-project data agents work** when the **agent project** has `geminidataanalytics.googleapis.com` enabled.

The failed direction was **not** blocked by cross-project BigQuery itself. It failed because the project acting as the agent project did **not** have `geminidataanalytics.googleapis.com` enabled.

## 1) API enablement audit

### Project A
- `bigquery.googleapis.com` — ENABLED
- `cloudaicompanion.googleapis.com` — ENABLED
- `aiplatform.googleapis.com` — ENABLED
- `geminidataanalytics.googleapis.com` — ENABLED

### Project B
- `bigquery.googleapis.com` — ENABLED
- `cloudaicompanion.googleapis.com` — ENABLED
- `aiplatform.googleapis.com` — ENABLED
- `geminidataanalytics.googleapis.com` — DISABLED

### Interpretation
This was the decisive config difference between the successful and failing directions.

## 2) Service agent provisioning
Before the first successful data-agent creation in Project A, the Gemini Data Analytics service agent was not visible.

After creating an agent successfully in Project A, the following IAM binding appeared:
- `roles/geminidataanalytics.serviceAgent` -> `service-<project-a-number>@gcp-sa-geminidataanalytics.iam.gserviceaccount.com`

In Project B, no equivalent service agent was provisioned during the failing path, which is consistent with `geminidataanalytics.googleapis.com` being disabled there.

### Interpretation
Readers should expect service-agent provisioning to be tied to the API being enabled and the service actually being initialized by use.

## 3) Live cross-project creation test -- SUCCESS
A synchronous data agent was created with this shape:
- **Agent project:** Project A
- **Source table:** a BigQuery table in Project B

Observed result:
- HTTP `200`
- agent creation succeeded
- the data source reference pointed to a remote-project BigQuery table

### Interpretation
This directly proves that **Project A agent -> Project B BigQuery source** is supported in the tested environment.

## 4) Live conversation test against the cross-project agent -- SUCCESS
A conversation was created against the cross-project agent and asked a simple counting question.

Prompt used:
- `How many rows are in the table? Return just the number.`

Observed result:
- returned answer: `336`
- generated SQL was a `COUNT(*)` query against the Project B table
- the BigQuery job executed in **Project A**
- the source table remained in **Project B**
- temporary results were written in **Project A**

### What readers should expect from this
This is the operational model to expect:
- the agent can reason over a remote table
- the query can target the remote table
- billing/job execution can still happen under the agent project
- scratch artifacts / temp outputs can land in the agent project

That means success is not just "agent creation works"; it also means **runtime query execution works across projects**.

## 5) Reverse-direction creation test -- FAILED
The reverse direction was then tested:
- **Agent project:** Project B
- **Source table:** a BigQuery table in Project A

Observed result:
- HTTP `403`
- error: `SERVICE_DISABLED`

### Interpretation
This is the critical point:

The failure did **not** indicate a cross-project BigQuery restriction.
It indicated that the project trying to host the agent was missing the required API.

In other words:
- **Project A agent -> Project B data** = worked
- **Project B agent -> Project A data** = failed
- root cause = Project B was not properly enabled as an **agent project**

## Root cause summary
The reverse-direction failure was caused by:
- missing `geminidataanalytics.googleapis.com` in the would-be agent project

It was **not** caused by:
- cross-project BigQuery reads being unsupported
- the source table being remote
- an inherent limitation of the data-agent product in this scenario

## What to check first if your setup fails
If readers hit a similar failure, check these in order:

1. Is the failing project acting as the **agent project**?
2. Is `geminidataanalytics.googleapis.com` enabled in that project?
3. Are `cloudaicompanion.googleapis.com` and `bigquery.googleapis.com` also enabled there?
4. Does the caller have `roles/geminidataanalytics.dataAgentCreator` and related roles in the agent project?
5. Does the caller have `roles/bigquery.dataViewer` on the remote source table/dataset?
6. Was the Gemini Data Analytics service agent provisioned after first use?
7. Is billing enabled in the agent project?

## Recommended reproduction path
If you want the cleanest repro for teammates:

1. Pick **Project A** as the agent project.
2. Enable the 3 required APIs in Project A.
3. Make sure your user/service account has the needed agent-creation and BigQuery roles.
4. Grant table read access in **Project B**.
5. Create a data agent in Project A using a BigQuery table from Project B.
6. Ask the agent a simple deterministic question like row count.
7. Confirm:
   - agent creation succeeded
   - SQL references the Project B table
   - BigQuery job shows up in Project A
   - answer returns successfully

Then, if you want to validate the failure mode:

8. Try the reverse direction with an intentionally under-configured project.
9. Confirm that the failure is `403 SERVICE_DISABLED`.
10. Enable `geminidataanalytics.googleapis.com` in that project and retry.

## Final conclusion
The current evidence supports the following conclusion:

1. **Cross-project BigQuery data agents are supported in practice** in this environment.
2. The agent project must be configured as a real Gemini Data Analytics project, not just a generic BigQuery project.
3. The most important setup requirement is enabling:
   - `geminidataanalytics.googleapis.com`
4. If that API is missing, the error can look like a product/data limitation when it is really a project setup issue.

## Small but important note
One early assumption in the investigation turned out to be wrong:
- `cloudaicompanion.googleapis.com` is **not** a substitute for `geminidataanalytics.googleapis.com`

For data-agent creation, the required API is still:
- `geminidataanalytics.googleapis.com`

That distinction matters because a project can already look "Gemini-enabled" in a broader sense and still fail data-agent creation specifically.

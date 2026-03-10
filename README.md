# BigQuery Data Agent Cross-Project Debug

Date: 2026-03-10

## Goal
Debug whether a BigQuery data agent can be created in one project while referencing BigQuery tables in another project, using Jo's ADC user credentials (with `GOOGLE_APPLICATION_CREDENTIALS` unset).

## Projects
- `johanesa-playground-326616` (project number `605626490127`)
- `temporary-playground-441700` (project number `190918028779`)

ADC account used:
- `admin@johanesa.altostrat.com`

## Key finding
**Cross-project data agents work** when the agent project has `geminidataanalytics.googleapis.com` enabled.

The failure mode observed was **not** a cross-project BigQuery restriction. The blocker was that `temporary-playground-441700` did **not** have `geminidataanalytics.googleapis.com` enabled, even though `bigquery.googleapis.com`, `cloudaicompanion.googleapis.com`, and `aiplatform.googleapis.com` were enabled.

## What was tested

### 1) API enablement audit

`johanesa-playground-326616`
- `bigquery.googleapis.com` — ENABLED
- `cloudaicompanion.googleapis.com` — ENABLED
- `aiplatform.googleapis.com` — ENABLED
- `geminidataanalytics.googleapis.com` — ENABLED

`temporary-playground-441700`
- `bigquery.googleapis.com` — ENABLED
- `cloudaicompanion.googleapis.com` — ENABLED
- `aiplatform.googleapis.com` — ENABLED
- `geminidataanalytics.googleapis.com` — **DISABLED**

## 2) Service agent provisioning
Before the first successful data-agent creation in `johanesa-playground-326616`, the Gemini Data Analytics service agent was not visible.

After creating an agent successfully, the following IAM binding appeared in `johanesa-playground-326616`:
- `roles/geminidataanalytics.serviceAgent` → `service-605626490127@gcp-sa-geminidataanalytics.iam.gserviceaccount.com`

In `temporary-playground-441700`, no equivalent service agent was provisioned because the API is disabled and no agent was created.

## 3) Live cross-project creation test (SUCCESS)
Created a synchronous data agent in:
- Agent project: `johanesa-playground-326616`
- Source table: `temporary-playground-441700.ewallet_provider.transactions`

Created agent:
- `projects/johanesa-playground-326616/locations/global/dataAgents/crossprojtest72d0450a`

Result:
- HTTP 200 success
- Agent was created with a BigQuery datasource reference pointing to the remote project table

## 4) Live conversation test against cross-project agent (SUCCESS)
Created a conversation against that agent and asked:
- `How many rows are in the table? Return just the number.`

Result:
- Agent returned `336 rows`
- Generated SQL:

```sql
SELECT
  COUNT(*) AS row_count
FROM
  `temporary-playground-441700.ewallet_provider.transactions` AS transactions
```

- BigQuery job executed in the **agent project**:
  - Job project: `johanesa-playground-326616`
  - Job ID: `job_HQtmoPvU61E_Lq47sgjbLQefwOWK`
- Query read from the **source project**:
  - `temporary-playground-441700.ewallet_provider.transactions`
- Temporary result table was written in the **agent project**:
  - dataset `_eeca2a77cc48102291f37a8cabec320bda3ccd61`

## 5) Reverse-direction creation test (FAIL, but not cross-project)
Tried to create an agent in:
- Agent project: `temporary-playground-441700`
- Source table: `johanesa-playground-326616.demo_dataset.employees`

Result:
- HTTP 403
- Error: `SERVICE_DISABLED`
- Exact cause: `geminidataanalytics.googleapis.com` is not enabled in `temporary-playground-441700`

This failure happened **before** any cross-project BigQuery access check.

## Conclusion
The current evidence says:

1. **Creating a data agent in Project A against BigQuery data in Project B is supported and working** in this environment.
2. The agent can also **query the remote project successfully at runtime**.
3. The real blocker in the failing direction is **API enablement in the agent project**, not cross-project BigQuery access.
4. The minimal API that matters for data-agent creation is:
   - `geminidataanalytics.googleapis.com`

## Likely explanation for the original issue
If the UI let you query cross-project data manually but refused to create or use a data agent in one of the projects, the most likely causes are:

1. `geminidataanalytics.googleapis.com` is disabled in the would-be **agent project**
2. The project has not yet provisioned its Gemini Data Analytics service agent
3. The console path may surface a vague error that looks like a dataset/project restriction, when the underlying cause is API/service initialization

## Recommended next check
If you want the reverse direction to work, enable:
- `geminidataanalytics.googleapis.com` on `temporary-playground-441700`

Then retry the same createSync call.

## Notes
One early assumption during investigation was wrong:
- `cloudaicompanion.googleapis.com` is **not** a substitute for `geminidataanalytics.googleapis.com`
- For data-agent creation, the actual required API is the latter

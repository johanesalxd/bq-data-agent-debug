# BigQuery Data Agent Cross-Project Selection Bug / API Repro

Date: 2026-03-10

## TL;DR

### User-facing problem
The scenario being tested is only this:

1. **Project A agent + Project A data**
2. **Project A agent + Project B data**

Observed user-facing behavior in the UI:
- when the data agent uses **Project A data**, the table is visible/selectable and agent creation works
- when the data agent should use **Project B data**, that cross-project data is **not visible / not selectable** in the UI
- this is confusing because the **same user can already query Project B BigQuery data directly**

### What this README proves
This README is intentionally scoped to answer one question only:

> Is this a real cross-project product limitation, or is the UI failing to surface a path that the API/CLI still supports?

Based on the API/HTTP tests, the answer is:

- **Project A agent + Project B data works via API**
- therefore the UI behavior can be misleading
- in the tested environment, the decisive setup requirement was that the **agent project** must have:
  - `geminidataanalytics.googleapis.com`
  - `cloudaicompanion.googleapis.com`
  - `bigquery.googleapis.com`

### Most likely interpretation
If the UI does not show the cross-project table as selectable, but the same identity can query that table directly, then the problem is likely one of these:

1. **UI bug / UI limitation / discoverability gap**
2. missing enablement in the **agent project**
3. missing required IAM for agent creation or conversation, even though raw BigQuery querying works

This README shows the **API path** that worked so readers can separate **UI behavior** from **backend capability**.

---

## Official docs re-checked
These docs were re-checked before writing this version of the README:

- Create data agents:
  https://docs.cloud.google.com/bigquery/docs/create-data-agents
- Create conversations:
  https://docs.cloud.google.com/bigquery/docs/create-conversations
- Conversational Analytics API overview:
  https://docs.cloud.google.com/gemini/data-agents/conversational-analytics-api/overview
- Build agent with HTTP:
  https://docs.cloud.google.com/gemini/docs/conversational-analytics-api/build-agent-http
- Access control:
  https://docs.cloud.google.com/gemini/docs/conversational-analytics-api/access-control
- Conversational analytics overview:
  https://docs.cloud.google.com/bigquery/docs/conversational-analytics

### Important doc points
From the docs above:

1. **Data agents are supported by the Conversational Analytics API / Gemini Data Analytics API**.
2. The API supports:
   - creating a data agent
   - creating a conversation
   - chatting with the agent
3. **Conversational Analytics IAM roles control access to the agent**, not the underlying data source.
4. When a user interacts with an agent, the connected data source is queried using **that user's credentials**.
5. To create/edit agents and run conversations, the project must have the required APIs enabled.

That means this is the right mental model:
- **agent permissions** and **BigQuery data permissions** are related but different
- raw BigQuery access working does **not automatically prove** the UI agent flow is correctly configured
- API success is the stronger proof of backend support

---

## Scope of this README
Only these two cases are covered:

### Case 1 — same-project baseline
- **Agent project:** Project A
- **Source data:** Project A

### Case 2 — cross-project target case
- **Agent project:** Project A
- **Source data:** Project B

This README does **not** cover:
- Project B as agent project
- reverse-direction experiments
- unrelated IAM edge cases outside the two scenarios above

---

## Prerequisites
Assume:

- **Project A** = project that hosts the agent
- **Project B** = project that hosts the remote BigQuery table
- same user identity is used for:
  - direct BigQuery access
  - API calls
  - agent creation
  - conversation creation

### Required APIs in Project A

```bash
export PROJECT_A="your-agent-project-id"
export PROJECT_B="your-data-project-id"
export PRINCIPAL="user:your.name@example.com"
export LOCATION="global"
```

Enable required APIs in **Project A**:

```bash
gcloud services enable \
  geminidataanalytics.googleapis.com \
  cloudaicompanion.googleapis.com \
  bigquery.googleapis.com \
  --project="$PROJECT_A"
```

Optional verification:

```bash
gcloud services list --enabled --project="$PROJECT_A" | grep -E 'geminidataanalytics|cloudaicompanion|bigquery'
```

### Required IAM in Project A
Google docs say the creator typically needs these in the agent project:

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

If the same principal also needs to enable APIs:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_A" \
  --member="$PRINCIPAL" \
  --role="roles/serviceusage.serviceUsageAdmin"
```

### Required data access in Project B
The same principal still needs read access to the actual BigQuery source data in Project B.

Example project-level grant:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_B" \
  --member="$PRINCIPAL" \
  --role="roles/bigquery.dataViewer"
```

If you already know the user can query the remote table directly, this part may already be satisfied.

### Auth used for the API calls
This repro assumes user auth via gcloud / ADC, not a custom service-account flow:

```bash
gcloud auth login
gcloud auth application-default login
TOKEN="$(gcloud auth print-access-token)"
```

---

## End-to-end repro script
This is the practical reproduction flow.

The exact pattern below is what mattered in testing:
- use **Project A** as the billing / agent project
- create one agent using a **Project A** table
- create one agent using a **Project B** table
- create a conversation for each
- ask a simple row-count question
- compare behavior

### Step 0 — fill in your tables

```bash
export PROJECT_A_DATASET="your_project_a_dataset"
export PROJECT_A_TABLE="your_project_a_table"

export PROJECT_B_DATASET="your_project_b_dataset"
export PROJECT_B_TABLE="your_project_b_table"

export AGENT_AA_ID="agent-project-a-data-project-a"
export AGENT_AB_ID="agent-project-a-data-project-b"
export CONV_AA_ID="conv-project-a-data-project-a"
export CONV_AB_ID="conv-project-a-data-project-b"

export PARENT="projects/${PROJECT_A}/locations/${LOCATION}"
export AGENT_AA_NAME="${PARENT}/dataAgents/${AGENT_AA_ID}"
export AGENT_AB_NAME="${PARENT}/dataAgents/${AGENT_AB_ID}"
export CONV_AA_NAME="${PARENT}/conversations/${CONV_AA_ID}"
export CONV_AB_NAME="${PARENT}/conversations/${CONV_AB_ID}"
```

### Step 1 — sanity check direct BigQuery access
Prove the caller can already query both tables directly.

#### 1A. Same-project table

```bash
bq query --use_legacy_sql=false \
"SELECT COUNT(*) AS row_count FROM \
\`${PROJECT_A}.${PROJECT_A_DATASET}.${PROJECT_A_TABLE}\`"
```

#### 1B. Cross-project table

```bash
bq query --use_legacy_sql=false \
"SELECT COUNT(*) AS row_count FROM \
\`${PROJECT_B}.${PROJECT_B_DATASET}.${PROJECT_B_TABLE}\`"
```

If **1B** works, then the identity already has direct cross-project BigQuery access.

### Step 2 — create a same-project agent (Project A agent + Project A data)

```bash
cat > /tmp/agent-aa.json <<EOF
{
  "name": "${AGENT_AA_NAME}",
  "dataAnalyticsAgent": {
    "publishedContext": {
      "datasourceReferences": [
        {
          "bq": {
            "tableReferences": [
              {
                "projectId": "${PROJECT_A}",
                "datasetId": "${PROJECT_A_DATASET}",
                "tableId": "${PROJECT_A_TABLE}"
              }
            ]
          }
        }
      ]
    }
  }
}
EOF

curl -sS -X POST \
  "https://geminidataanalytics.googleapis.com/v1beta/${PARENT}/dataAgents:createSync?data_agent_id=${AGENT_AA_ID}" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @/tmp/agent-aa.json | tee /tmp/agent-aa-response.json
```

Expected result:
- HTTP success
- response contains the created data agent resource in **Project A**

### Step 3 — create a cross-project agent (Project A agent + Project B data)

```bash
cat > /tmp/agent-ab.json <<EOF
{
  "name": "${AGENT_AB_NAME}",
  "dataAnalyticsAgent": {
    "publishedContext": {
      "datasourceReferences": [
        {
          "bq": {
            "tableReferences": [
              {
                "projectId": "${PROJECT_B}",
                "datasetId": "${PROJECT_B_DATASET}",
                "tableId": "${PROJECT_B_TABLE}"
              }
            ]
          }
        }
      ]
    }
  }
}
EOF

curl -sS -X POST \
  "https://geminidataanalytics.googleapis.com/v1beta/${PARENT}/dataAgents:createSync?data_agent_id=${AGENT_AB_ID}" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @/tmp/agent-ab.json | tee /tmp/agent-ab-response.json
```

Expected result in the tested working setup:
- HTTP success
- response contains the created data agent resource in **Project A**
- datasource reference points to **Project B**

This is the key proof that the backend supports the cross-project case even if the UI does not surface it cleanly.

### Step 4 — create a conversation for the same-project agent

```bash
cat > /tmp/conv-aa.json <<EOF
{
  "name": "${CONV_AA_NAME}",
  "agents": [
    "${AGENT_AA_NAME}"
  ]
}
EOF

curl -sS -X POST \
  "https://geminidataanalytics.googleapis.com/v1beta/${PARENT}/conversations?conversation_id=${CONV_AA_ID}" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @/tmp/conv-aa.json | tee /tmp/conv-aa-response.json
```

### Step 5 — create a conversation for the cross-project agent

```bash
cat > /tmp/conv-ab.json <<EOF
{
  "name": "${CONV_AB_NAME}",
  "agents": [
    "${AGENT_AB_NAME}"
  ]
}
EOF

curl -sS -X POST \
  "https://geminidataanalytics.googleapis.com/v1beta/${PARENT}/conversations?conversation_id=${CONV_AB_ID}" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @/tmp/conv-ab.json | tee /tmp/conv-ab-response.json
```

### Step 6 — ask the same simple question to both agents
Use a deterministic prompt:

```bash
export TEST_PROMPT="How many rows are in the table? Return just the number."
```

#### 6A. Chat with same-project agent conversation

```bash
cat > /tmp/chat-aa.json <<EOF
{
  "parent": "${PARENT}",
  "messages": [
    {
      "userMessage": {
        "text": "${TEST_PROMPT}"
      }
    }
  ],
  "conversationReference": {
    "conversation": "${CONV_AA_NAME}",
    "dataAgentContext": {
      "dataAgent": "${AGENT_AA_NAME}"
    }
  }
}
EOF

curl -sS -X POST \
  "https://geminidataanalytics.googleapis.com/v1beta/${PARENT}:chat" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @/tmp/chat-aa.json | tee /tmp/chat-aa-response.json
```

#### 6B. Chat with cross-project agent conversation

```bash
cat > /tmp/chat-ab.json <<EOF
{
  "parent": "${PARENT}",
  "messages": [
    {
      "userMessage": {
        "text": "${TEST_PROMPT}"
      }
    }
  ],
  "conversationReference": {
    "conversation": "${CONV_AB_NAME}",
    "dataAgentContext": {
      "dataAgent": "${AGENT_AB_NAME}"
    }
  }
}
EOF

curl -sS -X POST \
  "https://geminidataanalytics.googleapis.com/v1beta/${PARENT}:chat" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @/tmp/chat-ab.json | tee /tmp/chat-ab-response.json
```

### Step 7 — inspect what happened
Things to check:

1. Did **same-project agent creation** succeed?
2. Did **cross-project agent creation** succeed?
3. Did the cross-project response include the Project B table reference?
4. Did the conversation succeed?
5. Did the chat succeed?
6. Did the response include:
   - a row count
   - generated SQL against the expected table
   - BigQuery job metadata in Project A

If same-project works but cross-project only fails in the UI, while the API path above works, that strongly suggests a **UI-side limitation/bug/discoverability issue** rather than a backend product limitation.

---

## What was actually verified in testing
The backend verification established this:

### Verified working
- **Project A agent + Project B data** worked through the API path
- a conversation could be created against that agent
- the chat returned a valid answer
- generated SQL referenced the remote table in **Project B**
- the BigQuery job executed in **Project A**

### Interpreting that result
That means the backend supports:
- agent hosted in **Project A**
- source data living in **Project B**
- same user identity still governing data access

So if the UI does not show the remote table as selectable, that is **not enough evidence** to conclude the product does not support the scenario.

---

## Governance model
Google docs make this distinction explicit:

- **Gemini Data Analytics / Conversational Analytics roles** control access to the **agent**
- **BigQuery IAM** controls access to the **underlying data**

Google docs also state that when a user interacts with an agent, the connected data source is queried using **that user's credentials**.

So:
- the agent does **not** bypass BigQuery governance
- if the user cannot read the remote table directly, the agent should not magically grant that access
- if the user **can** read the remote table directly but the UI still does not show it, that points away from plain data IAM and toward setup or UI behavior

---

## Minimal conclusion
For the specific question this README cares about:

1. **Project A agent + Project A data** is the baseline and should work.
2. **Project A agent + Project B data** is supported by the API in the tested environment.
3. Therefore, if the UI does not show the Project B table as selectable, the issue is **not automatically proof of an unsupported cross-project scenario**.
4. The most important project-level prerequisite is that **Project A**, the agent project, must have:
   - `geminidataanalytics.googleapis.com`
   - `cloudaicompanion.googleapis.com`
   - `bigquery.googleapis.com`
5. If those are correct and direct BigQuery access already works, then the remaining explanation is likely a **UI bug / UI limitation / discoverability issue** rather than a backend capability gap.

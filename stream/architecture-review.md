# 🏭 Manufacturing IoT Streaming Architecture — Infra & Solution Expert Review

---

## 📐 Architecture Overview (Plain English)

This architecture describes a **real-time manufacturing floor IoT data pipeline** — from raw sensor/machine data all the way to a frontend shopfloor dashboard — using AWS-native services.

It is divided into **3 stages**:

| Stage | Focus | Key Services |
|---|---|---|
| Stage 1 | **Data Ingestion & Staging** | MQTT → Kinesis Firehose → S3 → Lambda → S3 |
| Stage 2 | **Business Logic & Enrichment** | Lambda (Complex BL) → NoSQL (DynamoDB) |
| Stage 3 | **Frontend API / Smart Query** | API Gateway → App API → DynamoDB / Lambda |

---

## 🗺️ Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        MANUFACTURING FLOOR (Shop Floor)                         │
│                                                                                 │
│  [PLC / Machine]  [Sensors]  [Robots]  [Power Meters]  [Andon Boards]          │
│         │               │         │           │                 ▲               │
│         └───────────────┴─────────┴───────────┘                 │               │
│                         │  MQTT Protocol                         │               │
└─────────────────────────┼─────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           STAGE 1 — DATA INGESTION & STAGING                     │
│                                                                                  │
│   [MQTT Broker / IoT Core]                                                       │
│          │                                                                       │
│          ▼                                                                       │
│   [AWS Kinesis Data Firehose] ──────────────────────────────────────────────     │
│          │  (streams raw binary/text payloads from machines)                    │
│          ▼                                                                       │
│   [S3 Bucket — RAW STAGE]  (raw data landing zone, partitioned by time)         │
│          │                                                                       │
│          │  (S3 Event Trigger / EventBridge)                                    │
│          ▼                                                                       │
│   [Lambda — JSON Formatter]                                                      │
│          │  - Parses raw MQTT payload                                            │
│          │  - Adds JSON structure & metadata (timestamp, machine_id, line_id)   │
│          │  - Validates schema                                                   │
│          ▼                                                                       │
│   [S3 Bucket — STRUCTURED STAGE]  (clean JSON, partitioned by line/date)        │
│          │                                                                       │
│          │  (also feeds Andon Boards via polling / API)                          │
│          ▼                                                                       │
│   [Andon Boards — Shop Floor Display]  ◄──── Real-time status display           │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 2 — BUSINESS LOGIC & NoSQL ENRICHMENT                    │
│                                                                                  │
│   [Lambda — Manufacturing Business Logic Engine]                                  │
│          │                                                                       │
│          │  Reads structured JSON from S3 Stage (Stage 1 output)                │
│          │                                                                       │
│          │  Applies:                                                             │
│          │    ✅ Production Yield Calculations                                   │
│          │    ✅ Batch Observations                                               │
│          │    ✅ Performance KPIs                                                 │
│          │    ✅ Power Consumption Metrics                                        │
│          │    ✅ Machine Parameters (Speed, RPM, etc.)                            │
│          │    ✅ Alerts & NG (No Good) Detection                                  │
│          │    ✅ Operation Info Mgmt & Collaboration Data                         │
│          │                                                                       │
│          ▼                                                                       │
│   [NoSQL — DynamoDB]                                                             │
│          - Well-structured items (not schema-less chaos)                         │
│          - Partition Key: line_id + date                                         │
│          - Sort Key: timestamp                                                   │
│          - GSI on machine_id, alert_type, shift_id                               │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    STAGE 3 — FRONTEND / SMART API QUERY LAYER                    │
│                                                                                  │
│   [Frontend App — Web/Mobile Dashboard]                                           │
│          │  User requests latest shop floor data                                  │
│          │  Sends: { usertime, line_id, machine_id, ... }                        │
│          ▼                                                                       │
│   [API Gateway]                                                                  │
│          ▼                                                                       │
│   [App API (Lambda / ECS)]                                                       │
│          │                                                                       │
│          ├──── Query DynamoDB for records newer than usertime                    │
│          │         │                                                             │
│          │    ┌────┴──────────────────────────────┐                             │
│          │    │  Records found in DynamoDB?        │                             │
│          │    │  YES ──► Return cached data ──►    │                             │
│          │    │  NO  ──► Trigger Stage-2 Lambda    │                             │
│          │    └───────────────────────────────────┘                             │
│          │                                                                       │
│          └──── [Stage-2 Lambda triggered on-demand]                             │
│                    └──► Processes latest S3 JSON ──► Writes to DynamoDB         │
│                              └──► Returns fresh data to API ──► Frontend        │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔬 Step-by-Step Explanation

### 🔷 STEP 1 — MQTT Machines Push Data
- Machines on the shop floor (PLCs, Sensors, Robots, Power Meters) publish telemetry over **MQTT protocol**.
- AWS IoT Core (or a self-hosted MQTT Broker) receives these streams.
- Messages may be raw binary, CSV, or loosely structured text payloads.

---

### 🔷 STEP 2 — Kinesis Firehose Streams to Raw S3
- **Kinesis Data Firehose** subscribes to the MQTT topic stream.
- It **buffers and batches** incoming data (configurable window: e.g., 60 seconds or 5 MB).
- Delivers raw data files into **S3 Raw Stage bucket**.
- Partitioned by `year/month/day/hour` automatically.

---

### 🔷 STEP 3 — Lambda: JSON Formatter
- **S3 Event Notification** triggers a Lambda function when new raw files land.
- Lambda:
  - Reads raw payload
  - Parses and normalizes into structured **JSON format**
  - Adds metadata: `machine_id`, `line_id`, `plant_id`, `timestamp`, `shift`
  - Validates schema
  - Writes clean JSON files to **S3 Structured Stage bucket**
- The structured JSON is also directly consumed by **Andon Boards** (shop floor displays) for real-time visual status.

---

### 🔷 STEP 4 — Lambda: Manufacturing Business Logic Engine
- A **dedicated Lambda function** (or Step Function for long runs) reads from the Structured S3 bucket.
- It applies **domain-specific manufacturing business logic**:

| Data Domain | What is computed |
|---|---|
| Production Yield | Good parts / Total parts × 100 |
| Batch Observations | Batch start/end, deviations, anomalies |
| Performance | OEE, cycle time, downtime |
| Power Consumption | kWh per unit, peak loads |
| Machine Parameters | Speed, RPM, temperature thresholds |
| Alerts | Over-threshold triggers, NG flags |
| NG (No Good) | Defect classification, counts |

- Enriched and structured records are written to **DynamoDB** with a **well-designed key schema** — not just dumped as raw blobs.

---

### 🔷 STEP 5 — Frontend User Requests Latest Data
- A user (supervisor, engineer, operator) opens the **frontend dashboard**.
- The app calls the **App API** with a payload containing:
  ```json
  {
    "usertime": "2026-03-12T08:00:00Z",
    "line_id": "LINE-A3",
    "plant_id": "PLANT-01"
  }
  ```

---

### 🔷 STEP 6 — Smart Query: DynamoDB First, Lambda on Miss
- The **App API** queries **DynamoDB**:
  - Checks if records exist **newer than `usertime`**
  - ✅ **Cache Hit** → Returns records directly from DynamoDB (fast, < 10ms)
  - ❌ **Cache Miss** → API **triggers Stage-2 Lambda** on-demand
    - Lambda processes the latest S3 JSON
    - Writes fresh enriched data to DynamoDB
    - Returns fresh results to the frontend

This is a **"lazy enrichment"** or **"pull-on-demand"** pattern — computationally efficient.

---

## ✅ Expert Review — Strengths

| # | Strength | Detail |
|---|---|---|
| 1 | **Separation of concerns** | Each stage is isolated (Ingestion, Transformation, Business Logic, API) |
| 2 | **Serverless-native** | Lambda + Firehose + DynamoDB = minimal ops overhead |
| 3 | **Cost-efficient** | Kinesis Firehose + S3 is very cheap for raw storage vs streaming DB |
| 4 | **Andon Board integration** | Structured JSON from Stage-1 cleanly feeds display boards without complex logic |
| 5 | **Smart cache pattern** | DynamoDB as a hot cache with on-demand Lambda fallback is pragmatic |
| 6 | **NoSQL with structure** | Using DynamoDB with deliberate key design (not schema-less chaos) is best practice |
| 7 | **Two-stage S3** | Raw stage preserves original data for reprocessing; structured stage enables downstream logic |

---

## ⚠️ Expert Review — Gaps & Recommendations

### 🔴 Critical

| # | Gap | Recommendation |
|---|---|---|
| C1 | **No dead-letter / error handling on Lambda** | Add DLQ (SQS Dead Letter Queue) on all Lambdas. Failed JSON parsing must not silently drop data |
| C2 | **No schema validation layer** | Add **AWS Glue Schema Registry** or a lightweight JSON Schema validator in the Formatter Lambda |
| C3 | **Lambda cold start on cache miss** | A cold Lambda on user request (Stage 3) causes latency spikes. Use **Provisioned Concurrency** for the Stage-2 Lambda |
| C4 | **MQTT broker not specified** | If using self-hosted MQTT (e.g., Mosquitto), that's a single point of failure. Use **AWS IoT Core** for managed, scalable MQTT with TLS |

### 🟡 Important

| # | Gap | Recommendation |
|---|---|---|
| I1 | **No data retention policy on S3** | Add S3 Lifecycle Rules: move raw data to Glacier after 30 days, delete after 365 days |
| I2 | **DynamoDB item size & TTL** | Set a **TTL attribute** on DynamoDB items to auto-expire old records (e.g., 7-day rolling window for hot data) |
| I3 | **No observability** | Add **CloudWatch Dashboards + Alarms** on Firehose delivery failures, Lambda errors, DynamoDB throttles |
| I4 | **API has no auth** | Ensure API Gateway uses **Cognito Authorizer** or **IAM SigV4** — shop floor data is sensitive |
| I5 | **Stage-2 Lambda complexity** | If business logic grows, consider migrating to **AWS Step Functions** for orchestration and retry control |
| I6 | **No fan-out for multiple consumers** | If more consumers need Stage-1 data (beyond Andon boards), introduce **SNS/EventBridge** as a fan-out after S3 write |

### 🟢 Nice to Have

| # | Suggestion | Benefit |
|---|---|---|
| N1 | Add **Kinesis Data Streams** before Firehose | Enables real-time sub-second consumers (e.g., live alerting) in parallel to batch delivery |
| N2 | **AWS Timestream** for time-series metrics | Better suited for machine parameters (RPM, speed, power) than DynamoDB for trend queries |
| N3 | **AWS IoT Analytics** | Provides built-in channel/pipeline/datastore for IoT with SQL query support |
| N4 | Add **API response caching** at API Gateway | Reduces Lambda invocations for identical requests within a short window |

---

## 🏗️ Revised Architecture (Recommended Target State)

```
[Machines / Sensors]
        │ MQTT/TLS
        ▼
[AWS IoT Core]  ──(Rule Engine)──►  [Kinesis Data Streams]
                                            │
                              ┌─────────────┴──────────────┐
                              ▼                             ▼
                  [Kinesis Firehose]             [Real-time Alert Lambda]
                  (batch to S3)                 (sub-second NG detection)
                              │
                              ▼
                  [S3 — Raw Stage]  (with S3 lifecycle policy)
                              │
                    (S3 event / EventBridge)
                              ▼
                  [Lambda — JSON Formatter]  (with DLQ + Schema validation)
                              │
                   ┌──────────┴───────────┐
                   ▼                      ▼
        [S3 — Structured Stage]    [EventBridge / SNS]
                   │                      │
                   │                ┌─────┴──────┐
                   ▼                ▼            ▼
        [Lambda / Step Functions  [Andon    [Other Consumers]
         — Business Logic Engine]  Boards]
                   │
                   ▼
            [DynamoDB]  (TTL + GSI + well-designed keys)
                   ▲
                   │
            [App API]  (Cognito Auth + API GW Cache)
                   ▲
                   │
            [Frontend Dashboard]
```

---

## 📁 Folder Structure (This Repository)

```
stream/
├── architecture-review.md        ← This file (expert review + diagrams)
├── stage1-ingestion/
│   ├── lambda_json_formatter/    ← Lambda: raw → structured JSON
│   └── firehose-config/          ← Kinesis Firehose IaC (Terraform/CDK)
├── stage2-business-logic/
│   └── lambda_manufacturing_bl/  ← Lambda: business logic engine
├── stage3-api/
│   └── app_api/                  ← API Gateway + App API Lambda
├── infrastructure/
│   ├── s3-buckets.tf             ← S3 raw + structured stage
│   ├── dynamodb.tf               ← DynamoDB table design
│   ├── kinesis.tf                ← Kinesis Firehose + Streams
│   └── iam-roles.tf              ← IAM roles for all Lambdas
└── docs/
    └── data-flow.md              ← Detailed data flow documentation
```

---

## 📊 Data Flow Summary Table

| Step | Source | Action | Destination | Trigger |
|---|---|---|---|---|
| 1 | Machine/PLC | Publish MQTT | IoT Core / Broker | Continuous |
| 2 | IoT Core | Stream raw data | Kinesis Firehose | Rule Engine |
| 3 | Firehose | Batch & deliver | S3 Raw Stage | Time/Size buffer |
| 4 | S3 Raw | File lands | Lambda Formatter | S3 Event |
| 5 | Lambda | Parse + structure | S3 Structured Stage | S3 Event trigger |
| 6 | S3 Structured | JSON available | Andon Boards (poll) | Polling interval |
| 7 | S3 Structured | Business logic run | DynamoDB | API miss / scheduled |
| 8 | Frontend | Request latest data | App API | User action |
| 9 | App API | Query DynamoDB | Return data / trigger Lambda | usertime comparison |

---

*Generated: 2026-03-12 | Repository: arun0543/test | Folder: stream*
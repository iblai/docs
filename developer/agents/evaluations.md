# Evaluation System

Measure and improve mentor quality by running structured experiments against datasets and grading results through human annotations or automated LLM-as-Judge scoring.

---

## Overview

The evaluation system provides a complete pipeline for assessing mentor performance:

1. **Create a dataset** of questions with optional expected answers
2. **Run an experiment** that sends each question to a mentor and records its response
3. **Grade the results** using human annotations, LLM-as-Judge, or both
4. **Export results** as CSV for analysis

All evaluation data is scoped to your organization (tenant) and isolated from other tenants.

## Key Features

- **Dataset management** — Create, update, and organize evaluation question sets
- **Multiple input methods** — Add items via JSON, CSV upload, or from existing chat traces
- **Async experiment execution** — Experiments run as background tasks; large datasets won't block the API
- **Human annotation** — Apply numeric, boolean, or categorical scores to individual responses
- **LLM-as-Judge** — Automatically grade experiment results using custom evaluation criteria
- **CSV export** — Download experiment results with scores for offline analysis
- **Score configs** — Define reusable scoring rubrics for consistent grading

## Authentication

All endpoints require a platform API key passed as a token:

```
Authorization: Token <api_key>
```

## Evaluation Pipeline

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────┐
│ 1. Create   │────>│ 2. Add Items │────>│ 3. Run         │────>│ 4. Grade │
│    Dataset   │     │   (JSON/CSV/ │     │    Experiment   │     │  (Human/ │
│              │     │    Traces)   │     │                │     │   LLM)   │
└─────────────┘     └──────────────┘     └────────────────┘     └──────────┘
                                                                      │
                                                                      v
                                                               ┌──────────┐
                                                               │ 5. View/ │
                                                               │   Export │
                                                               └──────────┘
```

## API Reference

**Base URL pattern:**

```
/api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/...
```

| Parameter | Description |
|-----------|-------------|
| `org` | Organization/tenant identifier (platform key) |
| `user_id` | User ID of the requesting admin |

All list endpoints support pagination with `page` (default: 1) and `limit` (default: 50, max: 200) query parameters. Paginated responses include a `meta` object:

```json
{
  "meta": {
    "page": 1,
    "limit": 50,
    "total_items": 12,
    "total_pages": 1
  }
}
```

---

### Datasets

Datasets are collections of evaluation questions. Each dataset is scoped to your organization.

#### List Datasets

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/
```

**Query parameters:** `page`, `limit`

**Response** `200 OK`:

```json
{
  "data": [
    {
      "name": "customer-support-eval",
      "description": "Evaluation dataset for customer support mentor",
      "metadata": { "platform_key": "my-tenant" },
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-15T10:30:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 50, "total_items": 1, "total_pages": 1 }
}
```

#### Create Dataset

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/
```

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique dataset name |
| `description` | string | No | Human-readable description |
| `metadata` | object | No | Arbitrary key-value metadata |

```json
{
  "name": "qa-eval-v1",
  "description": "QA accuracy evaluation",
  "metadata": {
    "category": "accuracy",
    "version": "1.0"
  }
}
```

**Response** `201 Created`:

```json
{
  "name": "qa-eval-v1",
  "description": "QA accuracy evaluation",
  "metadata": {
    "category": "accuracy",
    "version": "1.0",
    "platform_key": "my-tenant"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

#### Get Dataset

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/
```

Returns a single dataset by name. Validates that the dataset belongs to the requesting tenant.

**Response** `200 OK`: Same shape as a single item in the list response.

---

### Dataset Items

Items are the individual questions (with optional expected answers) within a dataset.

#### List Items

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/items/
```

**Query parameters:** `page`, `limit`

**Response** `200 OK`:

```json
{
  "data": [
    {
      "id": "item-uuid-1",
      "input": "What is machine learning?",
      "expected_output": "Machine learning is a subset of AI...",
      "metadata": {},
      "status": "ACTIVE",
      "source_trace_id": null,
      "source_observation_id": null,
      "created_at": "2024-01-15T10:35:00Z",
      "updated_at": "2024-01-15T10:35:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 50, "total_items": 1, "total_pages": 1 }
}
```

#### Add Items (Direct Input)

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/items/
```

Provide an `items` array. Each item requires an `input` field; `expected_output` is optional.

```json
{
  "items": [
    {
      "input": "What is machine learning?",
      "expected_output": "Machine learning is a subset of artificial intelligence that enables systems to learn from data."
    },
    {
      "input": "Explain neural networks",
      "expected_output": "Neural networks are computing systems inspired by biological neural networks."
    },
    {
      "input": "What is deep learning?"
    }
  ]
}
```

**Response** `201 Created`:

```json
{
  "created": 3,
  "items": ["..."]
}
```

#### Add Items (From Traces)

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/items/
```

Link existing chat traces to the dataset. The system extracts the user input and mentor response from each trace.

```json
{
  "trace_ids": [
    "trace-uuid-1",
    "trace-uuid-2",
    "trace-uuid-3"
  ]
}
```

> **Note:** Provide either `items` or `trace_ids` in a single request, not both.

**Response** `201 Created`: Same shape as direct input response.

#### Upload CSV

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/items/upload/
```

Upload a CSV file to bulk-create dataset items. Send as `multipart/form-data` with a `file` field.

**Constraints:**
- UTF-8 encoding
- Maximum file size: 10 MB
- Maximum rows: 10,000
- Must have an `input` column (required)
- `expected_output` column is optional

See [CSV Format](#csv-format) for details.

**Response** `201 Created`:

```json
{
  "created": 25,
  "items": ["..."]
}
```

#### Update Item

```
PUT /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/items/{item_id}/
```

**Request body** (all fields optional):

| Field | Type | Description |
|-------|------|-------------|
| `input` | string | The question/prompt |
| `expected_output` | string | Expected answer |
| `metadata` | object | Arbitrary metadata |
| `status` | string | `ACTIVE` or `ARCHIVED` |

**Response** `200 OK`: Updated item object.

#### Delete Item

```
DELETE /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/items/{item_id}/
```

**Response** `204 No Content`

> This action is irreversible.

---

### Experiments

Experiments run a mentor against every item in a dataset and record the responses. Each experiment is processed as a background task.

#### List Experiment Runs

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/runs/
```

**Query parameters:** `page`, `limit`

**Response** `200 OK`:

```json
{
  "data": [
    {
      "id": "run-uuid-1",
      "name": "run-abc12345",
      "metadata": {
        "platform_key": "my-tenant",
        "mentor_unique_id": "mentor-uuid",
        "initiated_by": "admin@example.com"
      },
      "created_at": "2024-01-15T11:00:00Z",
      "updated_at": "2024-01-15T11:30:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 50, "total_items": 1, "total_pages": 1 }
}
```

#### Start Experiment

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/runs/
```

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mentor_unique_id` | string | Yes | The `unique_id` of the mentor to evaluate |
| `run_name` | string | No | Custom name for the run (auto-generated if omitted) |
| `metadata` | object | No | Additional metadata |

```json
{
  "mentor_unique_id": "my-mentor-unique-id",
  "run_name": "experiment-v1",
  "metadata": {
    "purpose": "accuracy evaluation"
  }
}
```

This dispatches a background task. The API returns immediately with a `202 Accepted` response.

**What happens during an experiment:**
1. A new chat session is created for each dataset item
2. The mentor is invoked with the item's `input` through the standard chat pipeline
3. The mentor's response is recorded
4. Each interaction is traced and linked to the experiment run

**Response** `202 Accepted`:

```json
{
  "run_name": "experiment-v1",
  "task_id": "celery-task-uuid",
  "status": "started",
  "mentor_unique_id": "my-mentor-unique-id",
  "initiated_by": "admin@example.com"
}
```

#### Get Experiment Run Details

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/runs/{run_name}/
```

Returns detailed results including individual run items with their trace IDs.

**Response** `200 OK`:

```json
{
  "id": "run-uuid-1",
  "name": "experiment-v1",
  "metadata": {
    "platform_key": "my-tenant",
    "mentor_unique_id": "mentor-uuid",
    "initiated_by": "admin@example.com"
  },
  "created_at": "2024-01-15T11:00:00Z",
  "updated_at": "2024-01-15T11:30:00Z",
  "dataset_run_items": [
    {
      "id": "ri-1",
      "dataset_item_id": "item-1",
      "trace_id": "trace-1",
      "observation_id": "",
      "created_at": "2024-01-15T11:05:00Z"
    }
  ]
}
```

#### Export Results (CSV)

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/runs/{run_name}/export/
```

Downloads experiment results as a CSV file. Columns include: `item_id`, `input`, `expected_output`, `actual_output`, `trace_id`, and any score columns (prefixed with `score_`).

**Response** `200 OK` with `Content-Type: text/csv`:

```csv
item_id,input,expected_output,actual_output,trace_id,score_accuracy
item-1,What is AI?,AI is...,Artificial intelligence is...,trace-1,4.0
```

#### Delete Experiment Run

```
DELETE /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/runs/{run_name}/
```

**Response** `204 No Content`

> This action is irreversible.

---

### Scores

Scores are human annotations attached to individual traces from an experiment. Use scores to manually grade mentor responses.

#### List Scores

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/scores/
```

**Query parameters:**

| Parameter | Description |
|-----------|-------------|
| `page` | Page number (default: 1) |
| `limit` | Items per page (default: 50) |
| `dataset_run_id` | Filter by experiment run ID |
| `trace_id` | Filter by trace ID |
| `name` | Filter by score name (e.g., `accuracy`) |

**Response** `200 OK`:

```json
{
  "data": [
    {
      "id": "score-1",
      "name": "accuracy",
      "value": 4.0,
      "data_type": "NUMERIC",
      "comment": "Good response",
      "trace_id": "trace-1",
      "observation_id": null,
      "created_at": "2024-01-15T12:00:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 50, "total_items": 1, "total_pages": 1 }
}
```

#### Create Score

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/scores/
```

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `trace_id` | string | Yes | Trace ID from experiment run item |
| `name` | string | Yes | Score metric name (e.g., `accuracy`) |
| `value` | number | Yes | Score value |
| `data_type` | string | No | `NUMERIC` (default), `BOOLEAN`, or `CATEGORICAL` |
| `comment` | string | No | Explanation or notes |
| `observation_id` | string | No | Specific observation within the trace |
| `config_id` | string | No | Score config ID for rubric validation |
| `dataset_run_id` | string | No | Link score to an experiment run |

```json
{
  "trace_id": "trace-uuid-from-experiment",
  "name": "accuracy",
  "value": 4.0,
  "data_type": "NUMERIC",
  "comment": "Good response, covered the main points accurately"
}
```

**Response** `201 Created`:

```json
{
  "status": "created",
  "name": "accuracy",
  "value": 4.0
}
```

#### Delete Score

```
DELETE /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/scores/{score_id}/
```

**Response** `204 No Content`

---

### Score Configs

Score configs define reusable scoring rubrics for consistent, standardized grading across experiments.

#### List Score Configs

```
GET /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/score-configs/
```

**Query parameters:** `page`, `limit`

**Response** `200 OK`:

```json
{
  "data": [
    {
      "id": "cfg-1",
      "name": "accuracy",
      "data_type": "NUMERIC",
      "min_value": 1.0,
      "max_value": 5.0,
      "categories": null,
      "description": "Rate accuracy 1-5",
      "is_archived": false,
      "created_at": "2024-01-15T10:00:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 50, "total_items": 1, "total_pages": 1 }
}
```

#### Create Score Config

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/score-configs/
```

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Config name |
| `data_type` | string | Yes | `NUMERIC`, `BOOLEAN`, or `CATEGORICAL` |
| `min_value` | number | No | Minimum value (for `NUMERIC`) |
| `max_value` | number | No | Maximum value (for `NUMERIC`) |
| `categories` | array | No | Category definitions (for `CATEGORICAL`) |
| `description` | string | No | Human-readable description |

**Numeric example:**

```json
{
  "name": "accuracy",
  "data_type": "NUMERIC",
  "min_value": 1.0,
  "max_value": 5.0,
  "description": "Rate accuracy from 1 (wrong) to 5 (perfect)"
}
```

**Categorical example:**

```json
{
  "name": "safety",
  "data_type": "CATEGORICAL",
  "categories": [
    { "value": 0, "label": "Unsafe" },
    { "value": 0.5, "label": "Borderline" },
    { "value": 1.0, "label": "Safe" }
  ],
  "description": "Evaluate response safety"
}
```

**Response** `201 Created`: The created score config object.

---

### LLM-as-Judge

Automatically grade an entire experiment run using an LLM evaluator. The judge examines each item's input, expected output, and actual output against your custom criteria and assigns a score (0 to 1).

```
POST /api/ai-mentor/orgs/{org}/users/{user_id}/evaluations/datasets/{dataset_name}/runs/{run_name}/evaluate/
```

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `criteria` | string | Yes | Evaluation rubric for the judge |
| `score_name` | string | Yes | Name for the generated scores |
| `llm_provider` | string | No | LLM provider to use as judge |
| `llm_name` | string | No | Specific model name |
| `max_concurrency` | integer | No | Max parallel evaluations |

```json
{
  "criteria": "Evaluate the response on:\n1. Accuracy: Is the information factually correct?\n2. Completeness: Does it fully address the question?\n3. Clarity: Is it clear and well-structured?\n\nWeight accuracy most heavily.",
  "score_name": "quality"
}
```

This dispatches a background task. The experiment run must be completed before triggering judge evaluation.

**Response** `202 Accepted`:

```json
{
  "task_id": "celery-task-uuid",
  "status": "started",
  "score_name": "quality"
}
```

After completion, scores are available via the [List Scores](#list-scores) endpoint filtered by `dataset_run_id`.

---

## Workflows

### Full Evaluation Pipeline

**Step 1: Create a dataset**

```
POST .../evaluations/datasets/
{ "name": "qa-eval-v1", "description": "QA evaluation" }
```

**Step 2: Add items** (choose one method per request)

Option A — Direct input:
```
POST .../evaluations/datasets/qa-eval-v1/items/
{ "items": [{ "input": "Q1", "expected_output": "A1" }, ...] }
```

Option B — CSV upload:
```
POST .../evaluations/datasets/qa-eval-v1/items/upload/
Content-Type: multipart/form-data
```

Option C — From existing traces:
```
POST .../evaluations/datasets/qa-eval-v1/items/
{ "trace_ids": ["trace-1", "trace-2"] }
```

**Step 3: Run experiment**

```
POST .../evaluations/datasets/qa-eval-v1/runs/
{ "mentor_unique_id": "my-mentor", "run_name": "run-v1" }
```

**Step 4: Wait for completion, then check results**

```
GET .../evaluations/datasets/qa-eval-v1/runs/run-v1/
```

**Step 5: Grade results** (choose one or both)

Human annotation:
```
POST .../evaluations/scores/
{ "trace_id": "trace-1", "name": "accuracy", "value": 4.0, "data_type": "NUMERIC" }
```

LLM-as-Judge:
```
POST .../evaluations/datasets/qa-eval-v1/runs/run-v1/evaluate/
{ "criteria": "Evaluate accuracy and completeness", "score_name": "quality" }
```

**Step 6: View scores**

```
GET .../evaluations/scores/?dataset_run_id=<run-id>
```

**Step 7: Export**

```
GET .../evaluations/datasets/qa-eval-v1/runs/run-v1/export/
```

---

## CSV Format

### Upload Format

The upload CSV must be UTF-8 encoded with a header row. The `input` column is required; `expected_output` is optional.

```csv
input,expected_output
What is machine learning?,Machine learning is a subset of AI that enables systems to learn from data.
Explain neural networks,Neural networks are computing systems inspired by biological neural networks.
What is deep learning?,
```

**Limits:** 10 MB max file size, 10,000 max rows.

### Export Format

Exported CSVs include the following columns:

| Column | Description |
|--------|-------------|
| `item_id` | Dataset item ID |
| `input` | The question sent to the mentor |
| `expected_output` | Expected answer (if provided) |
| `actual_output` | Mentor's response |
| `trace_id` | Trace ID for the interaction |
| `score_<name>` | One column per score name (e.g., `score_accuracy`) |

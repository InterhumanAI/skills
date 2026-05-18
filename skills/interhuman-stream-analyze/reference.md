# Stream Analyze v1 — Message Reference

Schemas distilled from the Interhuman AsyncAPI spec (API version 1.3.13) for `/v1/stream/analyze` only.

## Envelope

All server-to-client messages share this outer shape:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | Discriminator: `signal.detected`, `engagement.updated`, `conversation_quality.updated`, `error` |
| `timestamp` | string | yes | ISO 8601 event time |
| `correlation_id` | string | no | WebSocket connection id for support and tracing |
| `data` | object | yes | Event-specific payload |

Client-to-server messages:

| Message | Format | Description |
|---------|--------|-------------|
| Video segment | binary | One encoded segment per frame; min 3s, max 32 MB |
| Session config | JSON text | `include`, `goal_dimensions` (see SKILL.md) |

## signal.detected

```json
{
  "type": "signal.detected",
  "timestamp": "2025-01-01T00:00:00.000000Z",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "signal_type": "agreement",
    "start": 3.0,
    "end": 11.0,
    "probability": "high",
    "rationale": "Subject nodded repeatedly while maintaining eye contact."
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.signal_type` | string | yes | See SignalType enum in SKILL.md |
| `data.start` | number | yes | Start time in session-cumulative seconds |
| `data.end` | number | yes | End time in session-cumulative seconds |
| `data.probability` | string | no | `high`, `medium`, or `low` |
| `data.rationale` | string \| null | no | Evidence-based explanation |

## engagement.updated

```json
{
  "type": "engagement.updated",
  "timestamp": "2025-01-01T00:00:00.000000Z",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "state": "engaged",
    "start": 3.0,
    "end": 11.0
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.state` | string | yes | `engaged`, `neutral`, or `disengaged` |
| `data.start` | number | yes | Window start (session-cumulative seconds) |
| `data.end` | number | yes | Window end (session-cumulative seconds) |

## conversation_quality.updated

Emitted when session config includes at least one CQI flag and CQI is computable for the window.

```json
{
  "type": "conversation_quality.updated",
  "timestamp": "2025-01-01T00:00:00.000000Z",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "overall": {
      "quality_index": 72.0,
      "clarity": 67.0,
      "authority": 68.0,
      "energy": 80.0,
      "rapport": 75.0,
      "learning": 70.0
    },
    "timeline": [
      {
        "start": 3.0,
        "end": 11.0,
        "values": {
          "quality_index": 70.0,
          "clarity": 69.0,
          "authority": 70.0,
          "energy": 78.0,
          "rapport": 77.0,
          "learning": 68.0
        }
      }
    ]
  }
}
```

| Field | Type | When present |
|-------|------|--------------|
| `data.overall` | ConversationQualityValues | Session `include` contains `conversation_quality_overall` |
| `data.timeline` | array of timeline entries \| null | Session `include` contains `conversation_quality_timeline` |

Each timeline entry has `start`, `end`, and `values` (ConversationQualityValues). The stream emits a **single** timeline entry per event — the current window.

### ConversationQualityValues

All scores are 0–100. `quality_index` is the mean of the five dimension scores. A dimension score of **50** means no diagnostic evidence was available.

| Field | Description |
|-------|-------------|
| `quality_index` | Overall conversation quality index |
| `clarity` | Clarity / Structure |
| `authority` | Authority / Credibility |
| `energy` | Energy / Presence |
| `rapport` | Rapport / Relational Safety |
| `learning` | Learning / Exploration |

## error

```json
{
  "type": "error",
  "timestamp": "2025-01-01T00:00:00.000000Z",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "code": "ih6002",
    "message": "WebSocket message too large. Individual video chunks must not exceed 32 MB.",
    "link": "https://docs.interhuman.ai/api-reference/error-handling#ih6002-message-too-large",
    "segment": 2
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data.code` | string | yes | Machine-readable error id |
| `data.message` | string | yes | Human-readable explanation |
| `data.link` | string \| null | no | Documentation URL |
| `data.segment` | integer \| null | no | Caller segment when applicable; omitted for connection-wide failures |

## Session config schema

```json
{
  "include": ["conversation_quality_overall", "conversation_quality_timeline"],
  "goal_dimensions": ["clarity", "authority"]
}
```

| Field | Allowed values |
|-------|----------------|
| `include[]` | `conversation_quality_overall`, `conversation_quality_timeline` |
| `goal_dimensions[]` | `clarity`, `authority`, `energy`, `rapport`, `learning` |

Additional properties are not allowed. Send as a plain JSON text frame (not wrapped in an `action`/`payload` envelope).

## Legacy v0 (do not implement)

`/v0/stream/analyze` uses query `include[]` at connect time and status-based messages (`processing`, `result`, `completed`, `error`). v1 replaces these with session-config JSON and typed envelopes above.

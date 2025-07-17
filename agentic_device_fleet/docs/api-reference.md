# API Reference - Agentic Device Fleet Monitoring System

## AI Agent API

### Endpoints

#### POST /query
Execute a natural language query about device fleet status.

**Request**
```json
{
  "query": "How many devices are currently active?",
  "context": {
    "user_id": "string",
    "session_id": "string",
    "filters": {
      "location": "string",
      "device_type": "string"
    }
  }
}
```

**Response**
```json
{
  "response": "Currently, there are 1,247 active devices in your fleet...",
  "data": {
    "active": 1247,
    "inactive": 23,
    "dead": 5
  },
  "insights": [
    "Device availability is at 98.2%, which is above the target threshold"
  ],
  "query_id": "uuid",
  "execution_time_ms": 1250
}
```

#### GET /health
Check system health and component status.

**Response**
```json
{
  "status": "healthy",
  "components": {
    "mcp_server": "healthy",
    "trino": "healthy",
    "data_pipeline": "healthy"
  },
  "last_updated": "2025-07-16T10:30:00Z"
}
```

## MCP Server Protocol

### Tools

#### get_fleet_status
Get current device fleet status counts and distribution.

**Parameters**
```typescript
interface FleetStatusParams {
  location?: string;      // Filter by location
  device_type?: string;   // Filter by device type
  time_window?: string;   // Time window for "current" (default: "1h")
}
```

**Response**
```json
{
  "total_devices": 1275,
  "status_breakdown": {
    "active": 1247,
    "inactive": 23,
    "dead": 5
  },
  "by_location": {
    "warehouse_a": {"active": 623, "inactive": 12, "dead": 2},
    "warehouse_b": {"active": 624, "inactive": 11, "dead": 3}
  },
  "query_timestamp": "2025-07-16T10:30:00Z"
}
```

#### get_device_trends
Analyze device status trends over specified time periods.

**Parameters**
```typescript
interface TrendParams {
  period: "hourly" | "daily" | "weekly" | "monthly";
  days_back: number;
  metric: "failures" | "uptime" | "status_changes" | "all";
  groupBy?: "location" | "device_type" | "none";
}
```

**Response**
```json
{
  "trend_data": [
    {
      "period": "2025-07-15",
      "active_count": 1250,
      "inactive_count": 20,
      "dead_count": 5,
      "failure_rate": 0.02
    }
  ],
  "insights": [
    "Device failures increased by 15% compared to last week",
    "Warehouse B shows higher failure rates than average"
  ],
  "statistics": {
    "avg_daily_failures": 3.2,
    "trend_direction": "increasing",
    "confidence": 0.85
  }
}
```

#### execute_custom_query
Execute custom SQL queries against the device data lake.

**Parameters**
```typescript
interface CustomQueryParams {
  sql: string;
  limit?: number;        // Result limit (default: 1000)
  timeout?: number;      // Query timeout in seconds (default: 30)
}
```

**Response**
```json
{
  "columns": ["device_id", "status", "timestamp"],
  "rows": [
    ["device_001", "active", "2025-07-16T10:30:00Z"],
    ["device_002", "inactive", "2025-07-16T10:29:45Z"]
  ],
  "row_count": 2,
  "execution_time_ms": 850,
  "query_id": "uuid"
}
```

#### get_device_details
Get detailed information about specific devices.

**Parameters**
```typescript
interface DeviceDetailsParams {
  device_ids?: string[];     // Specific device IDs
  status?: DeviceStatus;     // Filter by status
  location?: string;         // Filter by location
  include_history?: boolean; // Include status history
}
```

**Response**
```json
{
  "devices": [
    {
      "device_id": "device_001",
      "current_status": "active",
      "location": "warehouse_a",
      "device_type": "sensor",
      "last_heartbeat": "2025-07-16T10:29:55Z",
      "uptime_percentage": 99.2,
      "status_history": [
        {
          "status": "active",
          "timestamp": "2025-07-16T10:00:00Z",
          "duration_minutes": 30
        }
      ]
    }
  ]
}
```

## Data Pipeline API

### Kafka Topics

#### device-status-events
Real-time device status change events.

**Schema**
```json
{
  "device_id": "string",
  "timestamp": "2025-07-16T10:30:00Z",
  "status": "active|inactive|dead",
  "previous_status": "active|inactive|dead",
  "location": "string",
  "device_type": "string",
  "metadata": {
    "last_heartbeat": "2025-07-16T10:29:55Z",
    "signal_strength": 0.85,
    "battery_level": 0.72,
    "firmware_version": "1.2.3"
  }
}
```

### Trino Schema

#### datalake.fleet.device_events
Main event table with device status changes.

```sql
CREATE TABLE datalake.fleet.device_events (
    device_id VARCHAR NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    status VARCHAR NOT NULL,
    previous_status VARCHAR,
    location VARCHAR,
    device_type VARCHAR,
    last_heartbeat TIMESTAMP,
    signal_strength DOUBLE,
    battery_level DOUBLE,
    firmware_version VARCHAR,
    metadata JSON
)
WITH (
    partitioned_by = ARRAY['date'],
    format = 'PARQUET'
);
```

#### datalake.fleet.device_status_current
Materialized view of current device status.

```sql
CREATE VIEW datalake.fleet.device_status_current AS
WITH latest_events AS (
    SELECT 
        device_id,
        status,
        timestamp,
        location,
        device_type,
        ROW_NUMBER() OVER (PARTITION BY device_id ORDER BY timestamp DESC) as rn
    FROM datalake.fleet.device_events
    WHERE date >= CURRENT_DATE - INTERVAL '7' DAY
)
SELECT 
    device_id,
    status,
    timestamp as last_updated,
    location,
    device_type
FROM latest_events 
WHERE rn = 1;
```

## Error Handling

### Error Response Format
```json
{
  "error": {
    "code": "QUERY_TIMEOUT",
    "message": "Query execution exceeded timeout limit",
    "details": {
      "timeout_seconds": 30,
      "query_id": "uuid"
    },
    "timestamp": "2025-07-16T10:30:00Z"
  }
}
```

### Error Codes

| Code | Description |
|------|-------------|
| `INVALID_QUERY` | Malformed or unsafe SQL query |
| `QUERY_TIMEOUT` | Query execution timeout |
| `INSUFFICIENT_PERMISSIONS` | User lacks required permissions |
| `DATA_SOURCE_UNAVAILABLE` | Trino or data lake unavailable |
| `RATE_LIMIT_EXCEEDED` | Too many requests in time window |
| `INVALID_PARAMETERS` | Invalid tool parameters |

## Rate Limiting

- **Query API**: 100 requests per minute per user
- **Custom SQL**: 20 requests per minute per user
- **Health Check**: 1000 requests per minute

## Authentication

All APIs require Bearer token authentication:
```
Authorization: Bearer <jwt_token>
```

Token should include:
- `user_id`: User identifier
- `permissions`: Array of allowed operations
- `exp`: Token expiration time

## Monitoring

### Metrics Exposed

- `query_duration_seconds`: Query execution time
- `query_total`: Total number of queries
- `error_total`: Total number of errors by type
- `active_connections`: Current active connections
- `data_freshness_seconds`: Age of most recent data

### Health Check Endpoints

- `/health`: Overall system health
- `/metrics`: Prometheus metrics
- `/ready`: Readiness probe for Kubernetes

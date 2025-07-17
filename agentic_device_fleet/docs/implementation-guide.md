# Implementation Guide - Agentic Device Fleet Monitoring System

## Phase 1: Foundation Infrastructure

### 1.1 Data Pipeline Setup

**Kafka Event Schema Definition**
```python
# kafka_schemas.py
from dataclasses import dataclass
from datetime import datetime
from enum import Enum

class DeviceStatus(Enum):
    ACTIVE = "active"
    INACTIVE = "inactive" 
    DEAD = "dead"

@dataclass
class DeviceStatusEvent:
    device_id: str
    timestamp: datetime
    status: DeviceStatus
    previous_status: DeviceStatus
    location: str
    device_type: str
    last_heartbeat: datetime
    signal_strength: float
    battery_level: float
```

**Data Pipeline Implementation**
```python
# data_pipeline.py
import json
from kafka import KafkaConsumer
from datetime import datetime
import boto3

class DeviceEventProcessor:
    def __init__(self, bucket_name: str):
        self.bucket_name = bucket_name
        self.s3_client = boto3.client('s3')
        
    def process_events(self):
        consumer = KafkaConsumer(
            'device-status-events',
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
        
        for message in consumer:
            event = DeviceStatusEvent(**message.value)
            self.store_event(event)
    
    def store_event(self, event: DeviceStatusEvent):
        # Partition by date for efficient querying
        date_partition = event.timestamp.strftime('%Y-%m-%d')
        key = f"device-events/date={date_partition}/event_{event.device_id}_{event.timestamp.isoformat()}.json"
        
        self.s3_client.put_object(
            Bucket=self.bucket_name,
            Key=key,
            Body=json.dumps(event.__dict__, default=str)
        )
```

### 1.2 Trino Configuration

**Catalog Configuration** (`/etc/trino/catalog/datalake.properties`)
```properties
connector.name=hive
hive.metastore.uri=thrift://metastore:9083
hive.s3.endpoint=s3.amazonaws.com
hive.s3.aws-access-key=YOUR_ACCESS_KEY
hive.s3.aws-secret-key=YOUR_SECRET_KEY
```

**Table Creation**
```sql
-- Create external table for device events
CREATE TABLE datalake.fleet.device_events (
    device_id VARCHAR,
    timestamp TIMESTAMP,
    status VARCHAR,
    previous_status VARCHAR,
    location VARCHAR,
    device_type VARCHAR,
    last_heartbeat TIMESTAMP,
    signal_strength DOUBLE,
    battery_level DOUBLE
)
WITH (
    external_location = 's3://your-bucket/device-events/',
    format = 'JSON',
    partitioned_by = ARRAY['date']
);
```

## Phase 2: MCP Server Implementation

### 2.1 MCP Server Structure

> **Note:** The MCP server is implemented in TypeScript to leverage its strong type safety, modern async features, and robust ecosystem for building scalable backend services. TypeScript ensures reliable API contracts and easier maintenance, especially when integrating with complex protocols and external data sources.

```typescript
// mcp-server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

interface DeviceFleetContext {
  tables: TableSchema[];
  commonQueries: QueryTemplate[];
  deviceTypes: string[];
  locations: string[];
}

class DeviceFleetMCPServer {
  private server: Server;
  private trinoClient: TrinoClient;
  
  constructor() {
    this.server = new Server({
      name: "device-fleet-mcp",
      version: "1.0.0"
    }, {
      capabilities: {
        tools: {},
        resources: {}
      }
    });
    
    this.setupHandlers();
  }
  
  private setupHandlers() {
    // Tool definitions
    this.server.setRequestHandler("tools/list", async () => ({
      tools: [
        {
          name: "get_fleet_status",
          description: "Get current device fleet status counts",
          inputSchema: {
            type: "object",
            properties: {
              location: { type: "string", description: "Filter by location" },
              device_type: { type: "string", description: "Filter by device type" }
            }
          }
        },
        {
          name: "get_device_trends",
          description: "Analyze device status trends over time",
          inputSchema: {
            type: "object",
            properties: {
              period: { type: "string", enum: ["daily", "weekly", "monthly"] },
              days_back: { type: "number", description: "Number of days to look back" },
              metric: { type: "string", enum: ["failures", "uptime", "status_changes"] }
            }
          }
        },
        {
          name: "execute_custom_query",
          description: "Execute a custom SQL query on device data",
          inputSchema: {
            type: "object",
            properties: {
              sql: { type: "string", description: "SQL query to execute" }
            },
            required: ["sql"]
          }
        }
      ]
    }));
    
    // Tool execution
    this.server.setRequestHandler("tools/call", async (request) => {
      const { name, arguments: args } = request.params;
      
      switch (name) {
        case "get_fleet_status":
          return await this.getFleetStatus(args);
        case "get_device_trends":
          return await this.getDeviceTrends(args);
        case "execute_custom_query":
          return await this.executeCustomQuery(args);
        default:
          throw new Error(`Unknown tool: ${name}`);
      }
    });
  }
  
  private async getFleetStatus(args: any) {
    let query = `
      SELECT 
        status,
        COUNT(*) as count,
        COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() as percentage
      FROM datalake.fleet.device_events 
      WHERE date = CURRENT_DATE
    `;
    
    if (args.location) {
      query += ` AND location = '${args.location}'`;
    }
    if (args.device_type) {
      query += ` AND device_type = '${args.device_type}'`;
    }
    
    query += ` GROUP BY status ORDER BY count DESC`;
    
    const results = await this.trinoClient.execute(query);
    return { content: [{ type: "text", text: JSON.stringify(results) }] };
  }
}
```

### 2.2 Tool Implementations

```python
# tools/fleet_analytics.py
from typing import Dict, List, Any
import pandas as pd
from datetime import datetime, timedelta

class FleetAnalyticsTool:
    def __init__(self, trino_client):
        self.trino = trino_client
    
    def get_fleet_status(self, location: str = None, device_type: str = None) -> Dict[str, Any]:
        """Get current fleet status with real-time counts"""
        filters = []
        if location:
            filters.append(f"location = '{location}'")
        if device_type:
            filters.append(f"device_type = '{device_type}'")
        
        where_clause = " AND ".join(filters)
        if where_clause:
            where_clause = f"AND {where_clause}"
        
        query = f"""
        WITH latest_status AS (
            SELECT 
                device_id,
                status,
                timestamp,
                ROW_NUMBER() OVER (PARTITION BY device_id ORDER BY timestamp DESC) as rn
            FROM datalake.fleet.device_events 
            WHERE date >= CURRENT_DATE - INTERVAL '1' DAY
            {where_clause}
        )
        SELECT 
            status,
            COUNT(*) as count
        FROM latest_status 
        WHERE rn = 1
        GROUP BY status
        """
        
        results = self.trino.execute(query)
        return {
            "total_devices": sum(row["count"] for row in results),
            "status_breakdown": {row["status"]: row["count"] for row in results},
            "query_timestamp": datetime.utcnow().isoformat()
        }
    
    def analyze_trends(self, period: str, days_back: int, metric: str) -> Dict[str, Any]:
        """Analyze device trends with statistical insights"""
        if period == "daily":
            date_trunc = "day"
        elif period == "weekly":
            date_trunc = "week"
        else:
            date_trunc = "month"
        
        query = f"""
        WITH trend_data AS (
            SELECT 
                date_trunc('{date_trunc}', timestamp) as period,
                status,
                COUNT(*) as event_count
            FROM datalake.fleet.device_events
            WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '{days_back}' DAY
            GROUP BY date_trunc('{date_trunc}', timestamp), status
        )
        SELECT 
            period,
            status,
            event_count,
            LAG(event_count) OVER (PARTITION BY status ORDER BY period) as previous_count
        FROM trend_data
        ORDER BY period DESC, status
        """
        
        results = self.trino.execute(query)
        df = pd.DataFrame(results)
        
        # Calculate trends and insights
        insights = self._calculate_trend_insights(df)
        
        return {
            "trend_data": results,
            "insights": insights,
            "period": period,
            "days_analyzed": days_back
        }
    
    def _calculate_trend_insights(self, df: pd.DataFrame) -> List[str]:
        """Generate AI-friendly insights from trend data"""
        insights = []
        
        # Calculate failure rate trends
        dead_devices = df[df["status"] == "dead"]
        if not dead_devices.empty:
            dead_devices["change"] = dead_devices["event_count"] - dead_devices["previous_count"]
            avg_change = dead_devices["change"].mean()
            
            if avg_change > 0:
                insights.append(f"Device failures are trending upward with an average increase of {avg_change:.1f} per period")
            elif avg_change < 0:
                insights.append(f"Device failures are declining with an average decrease of {abs(avg_change):.1f} per period")
            else:
                insights.append("Device failure rates are stable")
        
        return insights
```

## Phase 3: AI Agent Implementation

### 3.1 Agent Core

```python
# agent/device_fleet_agent.py
from typing import Dict, List, Any
import openai
from mcp_client import MCPClient

class DeviceFleetAgent:
    def __init__(self, mcp_client: MCPClient):
        self.mcp = mcp_client
        self.conversation_history = []
        
    async def process_query(self, user_query: str) -> str:
        """Process user query and return intelligent response"""
        
        # Step 1: Understand user intent
        intent = await self._classify_intent(user_query)
        
        # Step 2: Execute appropriate tools
        tool_results = await self._execute_tools(intent, user_query)
        
        # Step 3: Analyze results and generate response
        response = await self._generate_response(user_query, tool_results)
        
        # Step 4: Update conversation history
        self._update_history(user_query, response)
        
        return response
    
    async def _classify_intent(self, query: str) -> Dict[str, Any]:
        """Classify user intent and extract parameters"""
        system_prompt = """
        You are a device fleet monitoring assistant. Classify the user's query into one of these categories:
        - fleet_status: Current device counts and health
        - trend_analysis: Historical patterns and changes
        - device_lookup: Specific device information
        - custom_analysis: Complex analytical queries
        
        Extract relevant parameters like time periods, locations, device types.
        Return JSON with 'intent', 'parameters', and 'confidence'.
        """
        
        response = await openai.ChatCompletion.acreate(
            model="gpt-4",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": query}
            ]
        )
        
        return json.loads(response.choices[0].message.content)
    
    async def _execute_tools(self, intent: Dict, query: str) -> Dict[str, Any]:
        """Execute appropriate MCP tools based on intent"""
        results = {}
        
        if intent["intent"] == "fleet_status":
            params = intent.get("parameters", {})
            results["fleet_status"] = await self.mcp.call_tool(
                "get_fleet_status",
                {
                    "location": params.get("location"),
                    "device_type": params.get("device_type")
                }
            )
        
        elif intent["intent"] == "trend_analysis":
            params = intent.get("parameters", {})
            results["trends"] = await self.mcp.call_tool(
                "get_device_trends",
                {
                    "period": params.get("period", "daily"),
                    "days_back": params.get("days_back", 7),
                    "metric": params.get("metric", "failures")
                }
            )
        
        return results
    
    async def _generate_response(self, query: str, tool_results: Dict) -> str:
        """Generate human-friendly response with insights"""
        system_prompt = """
        You are an expert device fleet analyst. Based on the query and data provided:
        1. Answer the user's question directly
        2. Provide relevant insights and patterns
        3. Suggest actionable recommendations when appropriate
        4. Use clear, non-technical language
        5. Highlight important trends or anomalies
        """
        
        context = f"""
        User Query: {query}
        Data Results: {json.dumps(tool_results, indent=2)}
        """
        
        response = await openai.ChatCompletion.acreate(
            model="gpt-4",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": context}
            ]
        )
        
        return response.choices[0].message.content
```

### 3.2 Query Examples

```python
# Example interactions
class QueryExamples:
    """Example queries and expected behavior"""
    
    @staticmethod
    def example_fleet_status():
        query = "How many devices are currently active?"
        # Expected: Agent classifies as fleet_status, calls get_fleet_status tool,
        # returns current active device count with context
        
    @staticmethod
    def example_trend_analysis():
        query = "Show me the daily trend of dead devices over the last 2 weeks"
        # Expected: Agent classifies as trend_analysis, calls get_device_trends 
        # with period=daily, days_back=14, analyzes pattern, provides insights
        
    @staticmethod
    def example_complex_analysis():
        query = "Which location has the most unreliable devices this month?"
        # Expected: Agent generates custom SQL query, executes via MCP,
        # analyzes results, identifies worst-performing location
```

## Phase 4: Deployment Architecture

### 4.1 Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  trino-coordinator:
    image: trinodb/trino:latest
    ports:
      - "8080:8080"
    volumes:
      - ./trino-config:/etc/trino
    environment:
      - TRINO_ENVIRONMENT=production
      
  mcp-server:
    build: ./mcp-server
    ports:
      - "3000:3000"
    environment:
      - TRINO_HOST=trino-coordinator
      - TRINO_PORT=8080
    depends_on:
      - trino-coordinator
      
  ai-agent:
    build: ./ai-agent
    ports:
      - "8000:8000"
    environment:
      - MCP_SERVER_URL=http://mcp-server:3000
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - mcp-server
      
  data-pipeline:
    build: ./data-pipeline
    environment:
      - KAFKA_BROKERS=${KAFKA_BROKERS}
      - S3_BUCKET=${S3_BUCKET}
    depends_on:
      - kafka
```

### 4.2 Kubernetes Deployment

```yaml
# k8s/agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: device-fleet-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: device-fleet-agent
  template:
    metadata:
      labels:
        app: device-fleet-agent
    spec:
      containers:
      - name: agent
        image: device-fleet-agent:latest
        ports:
        - containerPort: 8000
        env:
        - name: MCP_SERVER_URL
          value: "http://mcp-server-service:3000"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

## Implementation Timeline

**Week 1-2**: Data pipeline and Trino setup
**Week 3-4**: MCP server implementation and testing
**Week 5-6**: AI agent development and integration
**Week 7-8**: UI development and end-to-end testing
**Week 9-10**: Deployment, monitoring, and optimization

## Testing Strategy

- **Unit Tests**: Individual component testing
- **Integration Tests**: MCP server and tool interactions
- **End-to-End Tests**: Complete query flows
- **Performance Tests**: Query response times and scalability
- **User Acceptance Tests**: Natural language query accuracy

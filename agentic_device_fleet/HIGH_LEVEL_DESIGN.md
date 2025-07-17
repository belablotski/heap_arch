# High-Level Design: Agentic Device Fleet Monitoring System

## 1. Overview

This document outlines the high-level architecture for the Agentic Device Fleet Monitoring System. The system allows users to interact with and analyze device fleet data using natural language queries. It leverages a modern data stack (Kafka, S3, Trino) and a sophisticated agentic architecture to translate user intent into actionable data insights.

The primary goal is to provide an intelligent, conversational interface for monitoring fleet health, analyzing trends, and diagnosing issues without requiring users to have expertise in SQL or data analytics.

## 2. System Architecture

The architecture is composed of four primary layers, designed to be modular, scalable, and maintainable.

```
+----------------------+      +----------------------+      +----------------------+      +----------------------+
|   User Interface     | <--> |      AI Agent        | <--> |      MCP Server      | <--> |  Data & Tools Layer  |
| (CLI, Web App, etc.) |      | (Reasoning & NLU)    |      | (Protocol & Routing) |      | (Execution & Access) |
+----------------------+      +----------------------+      +----------------------+      +----------------------+
                                                                                           |
                                                                                           V
                                                                    +------------------------------------------+
                                                                    | Trino SQL Engine                         |
                                                                    +------------------------------------------+
                                                                                           |
                                                                                           V
                                                                    +------------------------------------------+
                                                                    | Data Lake (S3 Bucket)                    |
                                                                    +------------------------------------------+
                                                                                           ^
                                                                                           |
                                                                    +------------------------------------------+
                                                                    | Data Pipeline (Kafka -> Consumer)        |
                                                                    +------------------------------------------+
```

### Component Breakdown

1.  **Data Pipeline (Kafka -> Consumer -> S3):**
    *   **Responsibility:** Ingests real-time device status changes from the existing heartbeat service via Kafka. A consumer process then serializes these events into an optimized format (e.g., Parquet) and stores them in a partitioned S3 data lake.
    *   **Key Attributes:** Decoupled, scalable, and ensures a durable, queryable historical record of all events.

2.  **Data Serving Layer (Trino):**
    *   **Responsibility:** Provides a high-performance, federated SQL query engine on top of the S3 data lake. It exposes the fleet data through a standard SQL interface, abstracting away the underlying file storage.
    *   **Key Attributes:** Fast, scalable, and provides the core analytics horsepower.

3.  **Execution Layer (MCP Server & Tools):**
    *   **Responsibility:** This is the crucial abstraction layer. The **MCP Server** acts as a secure gateway that exposes a set of well-defined **Tools** to the AI Agent. The **Tools** are concrete functions that encapsulate the logic for interacting with the data layer (Trino). They are responsible for constructing and executing the actual SQL queries.
    *   **Key Attributes:** Provides security, abstraction, and maintainability. Prevents the AI Agent from needing to know about the database schema or writing raw SQL.

4.  **Intelligence Layer (AI Agent):**
    *   **Responsibility:** The "brain" of the system. It receives the user's natural language query, determines the user's *intent*, selects the appropriate tool(s) from the MCP Server, and calls them with the necessary parameters. After receiving structured data back from the tools, it synthesizes the information, generates insights, and formulates a human-readable response.
    *   **Key Attributes:** Handles all natural language understanding (NLU), reasoning, and response generation.

## 2.1. MCP Protocol Definition vs OpenAPI/Swagger

### How MCP Protocols Are Defined

The Model Context Protocol (MCP) uses a fundamentally different approach to API definition compared to OpenAPI/Swagger specifications. Understanding this difference is crucial for implementing our device fleet monitoring system.

#### TypeScript-First Schema Definition

Unlike OpenAPI which uses YAML/JSON schema files, MCP protocols are **defined primarily in TypeScript** and then compiled to JSON Schema:

```typescript
// device-fleet-mcp-schema.ts
export interface Tool {
  name: string;
  description?: string;
  inputSchema: JSONSchema;
}

export interface FleetStatusTool {
  name: "get_fleet_status";
  description: "Get current device fleet status counts and distribution";
  inputSchema: {
    type: "object";
    properties: {
      location?: { type: "string" };
      device_type?: { type: "string" };
      time_window?: { type: "string"; default: "1h" };
    };
  };
}

export interface FleetStatusRequest {
  method: "tools/call";
  params: {
    name: "get_fleet_status";
    arguments: {
      location?: string;
      device_type?: string;
      time_window?: string;
    };
  };
}
```

#### Key Differences from OpenAPI/Swagger

| Aspect | OpenAPI/Swagger | MCP Protocol |
|--------|-----------------|--------------|
| **Primary Format** | YAML/JSON schema files | TypeScript source code |
| **API Style** | REST endpoints with HTTP verbs | JSON-RPC style methods |
| **Documentation** | Auto-generated from schema | TypeScript interfaces serve as documentation |
| **Versioning** | Path-based (`/v1/api`) | Date-based schema versions (`2025-03-26`) |
| **Transport** | HTTP-only | Multiple transports (stdio, WebSocket, SSE) |
| **Capabilities** | Static endpoint definitions | Dynamic capability negotiation |

#### Dynamic Capability Discovery

Unlike REST APIs where endpoints are fixed, MCP supports **dynamic capability discovery**:

```typescript
// Client discovers server capabilities at runtime
const capabilities = await client.request("tools/list");
console.log(capabilities.tools); // Available tools

const resources = await client.request("resources/list");
console.log(resources.resources); // Available data sources
```

#### Device Fleet MCP Server Definition

For our device fleet system, the MCP server capabilities would be defined like this:

```typescript
export const DEVICE_FLEET_CAPABILITIES = {
  tools: {
    // Real-time queries
    get_fleet_status: {
      name: "get_fleet_status",
      description: "Get current device fleet status counts",
      inputSchema: {
        type: "object",
        properties: {
          location: { type: "string" },
          device_type: { type: "string" },
          time_window: { type: "string", default: "1h" }
        }
      }
    },
    
    // Analytics
    get_device_trends: {
      name: "get_device_trends", 
      description: "Analyze device status trends over time",
      inputSchema: {
        type: "object",
        properties: {
          period: { type: "string", enum: ["daily", "weekly", "monthly"] },
          days_back: { type: "number" },
          metric: { type: "string", enum: ["failures", "uptime", "status_changes"] }
        }
      }
    },
    
    // Custom analysis
    execute_custom_query: {
      name: "execute_custom_query",
      description: "Execute a custom SQL query on device data",
      inputSchema: {
        type: "object",
        properties: {
          sql: { type: "string" }
        },
        required: ["sql"]
      }
    }
  },
  
  resources: {
    // Schema information
    device_schema: {
      uri: "schema://device_events",
      name: "Device Events Schema",
      description: "Schema information for device events table"
    },
    
    // Common queries
    query_templates: {
      uri: "templates://common_queries",
      name: "Common Query Templates", 
      description: "Pre-built query templates for common operations"
    }
  }
} as const;
```

#### Advantages of MCP vs OpenAPI

1. **Type Safety**: TypeScript-first approach provides compile-time type checking
2. **Dynamic Discovery**: Capabilities can be discovered at runtime
3. **Multiple Transports**: Not limited to HTTP - supports stdio, WebSocket, SSE
4. **AI-Optimized**: Built specifically for AI model integration
5. **Versioning Strategy**: Date-based schema versions with backward compatibility

This protocol design makes MCP particularly well-suited for AI agent scenarios where you need structured, type-safe communication between models and tools.

## 3. Clarifying Responsibilities: Agent vs. MCP vs. Tools

This separation of concerns is critical for building a robust and secure agentic system.

| Component     | Role                  | Responsibilities                                                                                                                                    | Example                                                                                                                                                           |
| :------------ | :-------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AI Agent**  | The **"Brain"**       | - Understands natural language and user intent.<br>- Manages conversation context.<br>- **Selects the right tool** for the job.<br>- **Synthesizes tool outputs** into a final answer.<br>- Generates insights and human-readable text. | A user asks, "How did the fleet perform last week?" The Agent decides this is a `trend_analysis` task and calls the `get_device_trends` tool with `period: 'weekly'`. |
| **MCP Server**| The **"Bridge"**      | - Exposes available tools to the Agent via a standard protocol.<br>- Validates and routes incoming tool calls from the Agent.<br>- Enforces security, authentication, and rate limiting.<br>- Standardizes the "contract" between the Agent and the Tools. | The Agent sends a `tools/call` request for `get_device_trends`. The MCP Server validates the request, ensures the Agent has permission, and forwards it to the tool's implementation. |
| **Tools**     | The **"Hands"**       | - Encapsulates the logic for a specific, well-defined action.<br>- **Constructs and executes the actual SQL queries** against Trino.<br>- Performs data transformations or calculations.<br>- Returns structured, predictable data (e.g., JSON) to the Agent. | The `get_device_trends` tool receives the call. It builds the correct SQL `GROUP BY` query for Trino, runs it, processes the results, and returns a JSON object with the trend data. |

**In short:** The **Agent** decides *what* to do, the **MCP Server** provides the secure channel to do it, and the **Tool** knows *how* to do it. This prevents the Agent from having direct, unrestricted access to the database, making the system safer and easier to maintain.

## 4. Interaction Flow (Example Query)

Let's trace a user query through the system: "Give me a daily trend of how many dead devices we had in Warehouse A over the last week."

1.  **User -> AI Agent:** The user submits the query.
2.  **AI Agent (Reasoning):**
    *   **Intent:** The Agent identifies the intent as `trend_analysis`.
    *   **Parameters:** It extracts key parameters: `status: 'dead'`, `location: 'Warehouse A'`, `period: 'daily'`, `timeframe: 'last 7 days'`.
    *   **Tool Selection:** It determines the `get_device_trends` tool is the best fit.
3.  **AI Agent -> MCP Server:** The Agent makes a `tools/call` request to the MCP Server for the `get_device_trends` tool, passing the extracted parameters.
4.  **MCP Server -> Tool:** The MCP Server validates the request and routes it to the `get_device_trends` tool implementation.
5.  **Tool (Execution):**
    *   The tool's code constructs a precise SQL query:
      ```sql
      SELECT DATE_TRUNC('day', timestamp), COUNT(*)
      FROM datalake.fleet.device_events
      WHERE status = 'dead'
      AND location = 'Warehouse A'
      AND timestamp >= CURRENT_DATE - INTERVAL '7' DAY
      GROUP BY 1
      ORDER BY 1;
      ```
    *   The tool executes this query against Trino.
6.  **Trino -> Data Lake:** Trino reads the relevant partitioned data from the S3 bucket.
7.  **Data Lake -> Trino -> Tool:** The results are returned to the tool.
8.  **Tool -> MCP Server -> AI Agent:** The tool packages the results into a structured JSON format and returns it to the Agent.
9.  **AI Agent (Synthesis):**
    *   The Agent receives the JSON data (a list of dates and counts).
    *   It analyzes this data to identify patterns (e.g., "The number of dead devices spiked on Tuesday but has been stable since.").
    *   It formulates a comprehensive, natural language response.
10. **AI Agent -> User:** The user receives the final answer: "Here is the daily trend for dead devices in Warehouse A over the last week... On Tuesday, there was a spike to 15 devices, which was higher than the weekly average of 8..."

This architecture creates a powerful, flexible, and secure system for interacting with your device fleet data.

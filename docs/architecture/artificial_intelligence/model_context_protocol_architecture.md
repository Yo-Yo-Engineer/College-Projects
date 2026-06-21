# Model Context Protocol (MCP) Architecture

## Overview

**Model Context Protocol (MCP)** is an open-source standard for connecting AI applications to external systems. It provides a unified, protocol-level abstraction for how AI hosts — such as IDEs, chat interfaces, and agent frameworks — discover and interact with data sources, tools, and prompt templates exposed by MCP servers.

Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems — eliminating the need for bespoke integrations per tool, per host.

The protocol was created by Anthropic and is now an open standard governed by a Series of LF Projects, LLC.

Key principles:

- **Standardization** — A single protocol replaces N×M custom integrations between hosts and data sources with N+M connections
- **Composability** — AI applications gain capabilities by connecting to any number of MCP servers, each exposing tools, resources, and prompts
- **Separation of Concerns** — MCP servers own the "what" (data and actions); AI applications own the "how" (LLM reasoning and orchestration)
- **Dynamic Discovery** — Clients discover server capabilities at runtime through capability negotiation, not at build time
- **Transport Independence** — The same data-layer protocol works over local stdio and remote HTTP transports

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MCP Architecture                                    │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                      MCP Host (AI Application)                    │     │
│   │                                                                   │     │
│   │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │     │
│   │   │  MCP Client  │   │  MCP Client  │   │  MCP Client  │          │     │
│   │   │      1        │   │      2        │   │      3        │         │     │
│   │   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘         │     │
│   │          │                  │                   │                 │     │
│   └──────────│──────────────────│───────────────────│─────────────────┘     │
│              │ Dedicated        │ Dedicated          │ Dedicated             │
│              │ Connection       │ Connection          │ Connection           │
│              ▼                  ▼                     ▼                      │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│   │  MCP Server A     │  │  MCP Server B     │  │  MCP Server C     │        │
│   │  (Local)          │  │  (Local)          │  │  (Remote)         │        │
│   │  e.g. Filesystem  │  │  e.g. Database    │  │  e.g. Sentry      │        │
│   │                   │  │                   │  │                   │        │
│   │  Transport: stdio │  │  Transport: stdio │  │  Transport: HTTP  │        │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Participants

The MCP architecture has three key participants:

- **MCP Host** — The AI application that coordinates and manages one or multiple MCP clients. Examples: VS Code, Claude Desktop, Claude Code, custom agent applications
- **MCP Client** — A component within the host that maintains a dedicated connection to a single MCP server. Each client handles lifecycle management, capability negotiation, and message routing for that connection
- **MCP Server** — A program that provides context (tools, resources, prompts) to MCP clients. Can run locally (stdio transport) or remotely (Streamable HTTP transport)

### Protocol Layers

MCP consists of two layers:

```
┌─────────────────────────────────────────────────────────┐
│                    Data Layer                             │
│   (JSON-RPC 2.0 — lifecycle, primitives, notifications)  │
│                                                          │
│   ┌───────────────────────────────────────────────────┐  │
│   │  Lifecycle Management                              │  │
│   │  (initialize, capability negotiation, shutdown)    │  │
│   ├───────────────────────────────────────────────────┤  │
│   │  Server Primitives                                 │  │
│   │  (Tools, Resources, Prompts)                       │  │
│   ├───────────────────────────────────────────────────┤  │
│   │  Client Primitives                                 │  │
│   │  (Sampling, Elicitation, Logging)                  │  │
│   ├───────────────────────────────────────────────────┤  │
│   │  Utility Primitives                                │  │
│   │  (Notifications, Tasks)                            │  │
│   └───────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                   Transport Layer                         │
│   (connection establishment, message framing, auth)       │
│                                                          │
│   ┌────────────────────┐   ┌──────────────────────────┐ │
│   │   stdio Transport   │   │  Streamable HTTP          │ │
│   │ (local processes)   │   │  (remote servers, SSE)    │ │
│   └────────────────────┘   └──────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### Server Primitives

MCP servers expose three core primitive types:

| Primitive     | Purpose                                                   | Discovery        | Usage            |
| ------------- | --------------------------------------------------------- | ---------------- | ---------------- |
| **Tools**     | Executable functions the AI can invoke to perform actions | `tools/list`     | `tools/call`     |
| **Resources** | Data sources providing contextual information             | `resources/list` | `resources/read` |
| **Prompts**   | Reusable templates for structuring LLM interactions       | `prompts/list`   | `prompts/get`    |

### Client Primitives

MCP clients can expose capabilities that servers may use:

| Primitive       | Purpose                                                             |
| --------------- | ------------------------------------------------------------------- |
| **Sampling**    | Allows servers to request LLM completions from the host application |
| **Elicitation** | Allows servers to request additional input from the user            |
| **Logging**     | Enables servers to send log messages to clients for debugging       |

### Lifecycle

MCP is a stateful protocol with a defined lifecycle:

```
┌──────────┐                                          ┌──────────┐
│  Client   │                                          │  Server   │
└─────┬────┘                                          └─────┬────┘
      │                                                     │
      │─── initialize (protocolVersion, capabilities) ─────▶│
      │◀── initialize response (capabilities, serverInfo) ──│
      │                                                     │
      │─── notifications/initialized ──────────────────────▶│
      │                                                     │
      │         ┌──── Active Session ────┐                  │
      │         │                        │                  │
      │         │  tools/list            │                  │
      │─────────│  tools/call            │─────────────────▶│
      │◀────────│  resources/read        │──────────────────│
      │         │  prompts/get           │                  │
      │         │  notifications/*       │                  │
      │         │                        │                  │
      │         └────────────────────────┘                  │
      │                                                     │
      │─── shutdown ───────────────────────────────────────▶│
      │                                                     │
```

### Capability Negotiation

During initialization, client and server exchange capabilities to determine supported features:

```
// Connection Initialization — Client sends capabilities
FUNCTION initializeConnection(serverConfig : MCPServerConfig) -> MCPSession
    // 1. Establish transport
    transport = createTransport(serverConfig)

    // 2. Send initialize request
    initRequest = {
        protocolVersion : "2025-06-18",
        capabilities    : {
            elicitation : {}     // Client supports user interaction
        },
        clientInfo : {
            name    : hostApplication.name,
            version : hostApplication.version
        }
    }

    initResponse = transport.sendRequest("initialize", initRequest)

    // 3. Extract server capabilities
    serverCapabilities = initResponse.capabilities
    // e.g. { tools: { listChanged: true }, resources: {} }

    // 4. Notify server that client is ready
    transport.sendNotification("notifications/initialized")

    // 5. Create session with negotiated capabilities
    RETURN NEW MCPSession(
        transport          = transport,
        serverCapabilities = serverCapabilities,
        serverInfo         = initResponse.serverInfo
    )
END FUNCTION
```

## Implementation

### MCP Server

```
// MCP Server — Exposes tools, resources, and prompts to clients
CLASS MCPServer
    PROPERTIES
        name      : String
        version   : String
        tools     : Map<String, MCPTool>
        resources : Map<String, MCPResource>
        prompts   : Map<String, MCPPrompt>

    // Register a tool that clients can discover and invoke
    FUNCTION registerTool(tool : MCPTool) -> Void
        tools.PUT(tool.name, tool)

    // Handle tools/list request
    FUNCTION handleToolsList() -> ToolsListResponse
        toolDefinitions = EMPTY LIST
        FOR EACH tool IN tools.VALUES()
            toolDefinitions.ADD({
                name        : tool.name,
                title       : tool.title,
                description : tool.description,
                inputSchema : tool.parameterSchema
            })
        END FOR
        RETURN { tools: toolDefinitions }

    // Handle tools/call request
    FUNCTION handleToolsCall(request : ToolCallRequest) -> ToolCallResponse
        tool = tools.GET(request.name)
        IF tool IS NULL THEN
            RETURN errorResponse("Tool not found: " + request.name)
        END IF

        // Validate arguments against input schema
        validation = validateAgainstSchema(request.arguments, tool.parameterSchema)
        IF NOT validation.isValid THEN
            RETURN errorResponse("Invalid arguments: " + validation.errors)
        END IF

        // Execute the tool
        result = tool.execute(request.arguments)

        RETURN {
            content : [{ type: "text", text: result.toString() }]
        }

    // Handle resources/list request
    FUNCTION handleResourcesList() -> ResourcesListResponse
        RETURN {
            resources : [
                {
                    uri         : resource.uri,
                    name        : resource.name,
                    description : resource.description,
                    mimeType    : resource.mimeType
                }
                FOR EACH resource IN resources.VALUES()
            ]
        }

    // Handle resources/read request
    FUNCTION handleResourcesRead(request : ResourceReadRequest) -> ResourceReadResponse
        resource = resources.GET(request.uri)
        IF resource IS NULL THEN
            RETURN errorResponse("Resource not found: " + request.uri)
        END IF

        content = resource.read()
        RETURN {
            contents : [{
                uri      : request.uri,
                mimeType : resource.mimeType,
                text     : content
            }]
        }
END CLASS
```

### MCP Tool Definition

```
// MCP Tool — A function exposed to AI applications via the protocol
INTERFACE MCPTool
    PROPERTIES
        name            : String     // Unique identifier (e.g. "query_database")
        title           : String     // Human-readable display name
        description     : String     // What the tool does and when to use it
        parameterSchema : JSONSchema // Input parameter validation schema

    FUNCTION execute(arguments : Map) -> ToolResult
END INTERFACE

// Example: Weather Tool
CLASS WeatherTool IMPLEMENTS MCPTool
    PROPERTIES
        name            = "weather_current"
        title           = "Current Weather"
        description     = "Get the current weather conditions for a specified location. " +
                          "Returns temperature, humidity, and conditions."
        parameterSchema = SCHEMA {
            location : String (required) — "City name or coordinates"
            units    : String (optional, default "metric") — "metric or imperial"
        }

    FUNCTION execute(arguments : Map) -> ToolResult
        location = arguments.GET("location")
        units    = arguments.GET("units", "metric")

        weatherData = weatherAPI.getCurrentWeather(location, units)

        RETURN NEW ToolResult(
            content = "Current weather in " + location + ": " +
                      weatherData.temperature + "°, " +
                      weatherData.conditions + ", " +
                      "humidity " + weatherData.humidity + "%"
        )
END CLASS
```

### MCP Resource Definition

```
// MCP Resource — A data source that provides context to AI applications
INTERFACE MCPResource
    PROPERTIES
        uri         : String     // Unique resource identifier (e.g. "db://schema")
        name        : String     // Human-readable name
        description : String
        mimeType    : String     // Content type (e.g. "application/json")

    FUNCTION read() -> String
END INTERFACE

// Example: Database Schema Resource
CLASS DatabaseSchemaResource IMPLEMENTS MCPResource
    CONSTRUCTOR(dbConnection : DatabaseConnection)

    PROPERTIES
        uri         = "db://schema"
        name        = "Database Schema"
        description = "Complete schema of the application database including tables, " +
                      "columns, types, and relationships."
        mimeType    = "application/json"

    FUNCTION read() -> String
        tables = dbConnection.getSchemaInfo()
        RETURN SERIALIZE_TO_JSON(tables)
END CLASS
```

### MCP Prompt Definition

```
// MCP Prompt — A reusable template for structuring LLM interactions
INTERFACE MCPPrompt
    PROPERTIES
        name        : String
        description : String
        arguments   : List<PromptArgument>

    FUNCTION render(arguments : Map) -> List<PromptMessage>
END INTERFACE

// Example: SQL Query Assistance Prompt
CLASS SQLAssistantPrompt IMPLEMENTS MCPPrompt
    PROPERTIES
        name        = "sql_assistant"
        description = "Generate SQL queries based on natural language questions. " +
                      "Includes database schema context and few-shot examples."
        arguments   = [
            { name: "question", description: "Natural language question", required: true }
        ]

    FUNCTION render(arguments : Map) -> List<PromptMessage>
        question = arguments.GET("question")

        RETURN [
            NEW PromptMessage(
                role    = "system",
                content = "You are a SQL expert. Given the database schema below, " +
                          "write correct, efficient SQL queries.\n\n" +
                          "SCHEMA:\n" + getSchemaContext() + "\n\n" +
                          "RULES:\n" +
                          "- Use only SELECT queries unless explicitly asked otherwise\n" +
                          "- Always use table aliases for readability\n" +
                          "- Include LIMIT clauses for large result sets"
            ),
            NEW PromptMessage(
                role    = "user",
                content = question
            )
        ]
END CLASS
```

### MCP Client (Host Integration)

```
// MCP Client Manager — Manages connections to multiple MCP servers
CLASS MCPClientManager
    PROPERTIES
        sessions       : Map<String, MCPSession>
        availableTools : List<ToolDefinition>

    // Initialize connections to all configured servers
    FUNCTION initialize(configs : List<MCPServerConfig>) -> Void
        FOR EACH config IN configs
            session = initializeConnection(config)

            // Discover server capabilities
            IF session.serverCapabilities.HAS("tools") THEN
                toolsResponse = session.sendRequest("tools/list")
                availableTools.ADD_ALL(toolsResponse.tools)
            END IF

            sessions.PUT(config.name, session)
        END FOR

    // Route a tool call to the correct server
    FUNCTION callTool(toolName : String, arguments : Map) -> ToolResult
        session = findSessionForTool(toolName)
        IF session IS NULL THEN
            THROW UnknownToolError("No server provides tool: " + toolName)
        END IF

        response = session.sendRequest("tools/call", {
            name      : toolName,
            arguments : arguments
        })

        RETURN parseToolResult(response)

    // Handle notification that a server's tools have changed
    FUNCTION handleToolsChanged(session : MCPSession) -> Void
        // Re-fetch the tool list from the changed server
        toolsResponse = session.sendRequest("tools/list")

        // Update the unified tool registry
        removeToolsForSession(session)
        availableTools.ADD_ALL(toolsResponse.tools)

        // Notify the LLM of updated capabilities
        IF conversation.isActive() THEN
            conversation.refreshAvailableTools(availableTools)
        END IF

    PRIVATE FUNCTION findSessionForTool(toolName : String) -> MCPSession OR NULL
        FOR EACH sessionName, session IN sessions
            IF session.providesTool(toolName) THEN
                RETURN session
            END IF
        END FOR
        RETURN NULL
END CLASS
```

### Transport Layer

#### stdio Transport (Local Servers)

```
// stdio Transport — For local process communication
// The MCP server runs as a child process; communication happens via stdin/stdout

CONFIGURATION LocalMCPServer
    command   : "npx"
    args      : ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/files"]
    transport : "stdio"
END CONFIGURATION

// Host spawns the server process and pipes JSON-RPC messages
FUNCTION createStdioTransport(config : MCPServerConfig) -> Transport
    process = SPAWN(command = config.command, args = config.args)

    RETURN NEW StdioTransport(
        input  = process.stdout,    // Read server responses from stdout
        output = process.stdin      // Send client requests to stdin
    )
END FUNCTION
```

#### Streamable HTTP Transport (Remote Servers)

```
// Streamable HTTP Transport — For remote server communication
// Client sends HTTP POST requests; server may use Server-Sent Events for streaming

CONFIGURATION RemoteMCPServer
    url       : "https://mcp.example.com/sse"
    transport : "streamable-http"
    auth      : {
        type  : "bearer",
        token : "${MCP_AUTH_TOKEN}"
    }
END CONFIGURATION

// HTTP-based transport with SSE support
FUNCTION createHTTPTransport(config : MCPServerConfig) -> Transport
    RETURN NEW StreamableHTTPTransport(
        baseUrl = config.url,
        headers = buildAuthHeaders(config.auth)
    )
END FUNCTION
```

### Configuration

#### Host Configuration

```
// MCP configuration file (mcp.json / mcp-config.json)
CONFIGURATION MCPHostConfig
    servers : {

        // Local filesystem server
        "filesystem" : {
            command   : "npx",
            args      : ["-y", "@modelcontextprotocol/server-filesystem", "./src"],
            transport : "stdio"
        },

        // Local database server
        "database" : {
            command   : "python",
            args      : ["-m", "mcp_server_sqlite", "--db-path", "./app.db"],
            transport : "stdio"
        },

        // Remote server with authentication
        "sentry" : {
            url       : "https://mcp.sentry.dev/sse",
            transport : "streamable-http",
            auth      : { type: "bearer", token: "${SENTRY_AUTH_TOKEN}" }
        }
    }
END CONFIGURATION
```

## Project Structure

```
src/
├── servers/                        # MCP Server Implementations
│   ├── tools/                      # Tool definitions
│   │   ├── database/
│   │   ├── search/
│   │   ├── file_operations/
│   │   └── api_clients/
│   ├── resources/                  # Resource providers
│   │   ├── schema/
│   │   ├── documentation/
│   │   └── configuration/
│   ├── prompts/                    # Prompt templates
│   │   ├── code_generation/
│   │   ├── analysis/
│   │   └── data_queries/
│   └── transport/                  # Transport implementations
│       ├── stdio/
│       └── http/
│
├── clients/                        # MCP Client Integration
│   ├── client_manager/             # Multi-server connection management
│   ├── tool_registry/              # Unified tool discovery
│   ├── session/                    # Session and lifecycle management
│   └── transport/                  # Client-side transport adapters
│
├── shared/                         # Shared Protocol Types
│   ├── messages/                   # JSON-RPC message definitions
│   ├── schemas/                    # JSON Schema validation
│   └── errors/                     # Protocol error types
│
├── config/                         # Configuration Files
│   ├── mcp.json                    # Server configurations
│   └── mcp-config.json             # Host-level settings
│
└── tests/
    ├── unit/
    ├── integration/
    └── protocol_compliance/        # MCP spec conformance tests
```

## Key Design Considerations

### Tool Design (Agent-Computer Interface)

Tools are the primary interaction surface between AI and external systems. Invest the same effort in tool design that you would in human-computer interface (HCI) design:

- **Clear naming** — Use descriptive names like `query_database` rather than `query`. Include a verb that indicates the action
- **Rich descriptions** — Write descriptions as if for a junior developer; include usage examples, edge cases, and input format requirements
- **Type-safe parameters** — Use JSON Schema with required/optional fields, enums for constrained values, and descriptive parameter names
- **Error-proof design** — Make it difficult to use tools incorrectly (e.g., require absolute paths instead of relative paths to avoid ambiguity)
- **Minimal side effects** — Prefer read-only tools; require explicit confirmation for state-changing operations

### Security Considerations

- **Authentication** — Use OAuth, API keys, or bearer tokens for remote servers; local servers rely on process isolation
- **Authorization** — Servers should verify that requested operations are permitted for the authenticated client
- **Input validation** — Validate all tool arguments against schemas before execution
- **Content filtering** — Sanitize tool outputs to prevent injection of malicious content into LLM context
- **Data sovereignty** — Consider where MCP servers run and whether data crosses compliance boundaries

### Scaling Patterns

- **Local servers** (stdio) — One server per client; best for single-user desktop applications
- **Remote servers** (HTTP) — One server serving many clients; best for shared enterprise services
- **Server composition** — Compose multiple focused MCP servers rather than building monolithic servers with many tools
- **Registration** — Use dynamic server registries for organizations running many MCP servers across teams

## Benefits

1. **Eliminates N×M Integration Problem** — One protocol replaces per-host, per-tool custom integrations
2. **Dynamic Discovery** — AI applications discover capabilities at runtime; no hardcoded tool lists
3. **Ecosystem Growth** — Any MCP-compatible server works with any MCP-compatible host
4. **Separation of Concerns** — Tool/data providers and AI application developers work independently
5. **Transport Flexibility** — Same protocol works for local process communication and remote HTTP services
6. **Real-time Updates** — Notification system keeps clients synchronized with server changes

## Trade-offs

| Advantage                              | Consideration                                               |
| -------------------------------------- | ----------------------------------------------------------- |
| Standardized tool integration          | Protocol overhead compared to direct API calls              |
| Dynamic capability discovery           | Stateful connections require lifecycle management           |
| Works across hosts and languages       | Server implementations needed for each data source          |
| Local and remote transport options     | Security model differs significantly between transports     |
| Growing ecosystem of community servers | Server quality and security vary across community offerings |

## When to Use

✅ **Good fit for:**

- AI applications that need to integrate with multiple external tools and data sources
- Organizations building reusable AI tool infrastructure across teams
- Agent frameworks that require dynamic tool discovery and invocation
- IDE extensions and developer tools powered by AI
- Scenarios where the same tools should be accessible from multiple AI hosts

❌ **Not ideal for:**

- Simple AI applications with a single, fixed set of tools
- Extremely latency-sensitive inference pipelines where protocol overhead matters
- Scenarios requiring only one-shot, stateless tool calls with no discovery
- Applications where all data and tools are already embedded in the LLM context

## References

- [Model Context Protocol — Introduction](https://modelcontextprotocol.io/introduction)
- [MCP Architecture Overview](https://modelcontextprotocol.io/docs/learn/architecture)
- [MCP Specification (2025-03-26)](https://spec.modelcontextprotocol.io/specification/2025-03-26/)
- [Building Effective Agents — Anthropic (2024)](https://www.anthropic.com/engineering/building-effective-agents)
- [MCP Server Implementations — GitHub](https://github.com/modelcontextprotocol/servers)
- [MCP SDKs — Getting Started](https://modelcontextprotocol.io/docs/sdk)

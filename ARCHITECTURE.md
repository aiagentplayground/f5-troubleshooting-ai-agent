# F5 MCP Server Architecture

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (kagent)                   │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   AI Assistant │                                              │
│  │   (via kagent) │                                              │
│  └────────┬───────┘                                              │
│           │                                                       │
│           │ Discovers & Uses MCP Tools                           │
│           ▼                                                       │
│  ┌─────────────────────┐                                         │
│  │  RemoteMCPServer    │                                         │
│  │  (f5-mcp-remote)    │                                         │
│  │                     │                                         │
│  │  - obj_list         │                                         │
│  │  - get_pool_status  │                                         │
│  │  - get_vs_status    │                                         │
│  └──────────┬──────────┘                                         │
│             │                                                     │
│             │ HTTP POST to                                       │
│             │ http://f5-mcp-server.kagent.svc:8081/mcp          │
│             ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐        │
│  │         Service: f5-mcp-server (ClusterIP)          │        │
│  │                  Port: 8081                          │        │
│  └─────────────────────┬───────────────────────────────┘        │
│                        │                                         │
│                        ▼                                         │
│  ┌──────────────────────────────────────────────────────┐       │
│  │     Deployment: f5-mcp-server                        │       │
│  │  ┌────────────────────────────────────────────────┐  │       │
│  │  │  Pod: f5-mcp-server                            │  │       │
│  │  │                                                 │  │       │
│  │  │  Container: f5-mcp                             │  │       │
│  │  │  Image: sebbycorp/f5-mcp-server:latest        │  │       │
│  │  │  Port: 8081                                    │  │       │
│  │  │                                                 │  │       │
│  │  │  Environment:                                  │  │       │
│  │  │  - F5_HOST=https://172.16.10.10               │  │       │
│  │  │  - F5_USER=admin                              │  │       │
│  │  │  - F5_PASS=W3lcome098!                        │  │       │
│  │  │                                                 │  │       │
│  │  │  FastMCP HTTP Server                          │  │       │
│  │  │  (Stateless HTTP mode)                        │  │       │
│  │  └────────────────┬────────────────────────────────┘  │       │
│  └───────────────────┼────────────────────────────────────┘       │
│                      │                                            │
└──────────────────────┼────────────────────────────────────────────┘
                       │
                       │ HTTPS (REST API calls)
                       │ /mgmt/tm/ltm/*
                       ▼
            ┌──────────────────────┐
            │   F5 BIG-IP Device   │
            │   172.16.10.10       │
            │                      │
            │  - Pools             │
            │  - Virtual Servers   │
            │  - Nodes             │
            │  - Health Monitors   │
            └──────────────────────┘
```

## How It Works

### 1. **AI Assistant Query**
   - User asks: "Show me all pools on F5"
   - kagent receives the request

### 2. **kagent Discovers Tools**
   - kagent queries the RemoteMCPServer (`f5-mcp-remote`)
   - Discovers available tools: `obj_list`, `get_pool_status`, `get_virtual_server_status`

### 3. **kagent Calls MCP Server**
   - kagent sends HTTP POST to `http://f5-mcp-server.kagent.svc.cluster.local:8081/mcp`
   - JSON-RPC 2.0 payload with tool call

### 4. **F5 MCP Server Processing**
   - Receives the tool call request
   - Uses F5 credentials from environment variables
   - Creates connection to F5 BIG-IP using `bigrest` library
   - Makes REST API call to F5: `GET /mgmt/tm/ltm/pool`

### 5. **F5 BIG-IP Response**
   - F5 device returns pool data (JSON)
   - F5 MCP server formats the response

### 6. **Response to AI Assistant**
   - F5 MCP server returns formatted data to kagent
   - kagent presents the information to the AI assistant
   - AI assistant shows the user the pool list

---

## Comparison: Slack Bot vs F5 MCP Server

### Slack Bot (stdio transport)

```yaml
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata:
  name: levan-slack-mcp
spec:
  deployment:
    image: "zencoderai/slack-mcp:latest"
    port: 3000
    cmd: "node"
    args:
      - "dist/index.js"
      - "--transport"
      - "stdio"                    # <-- stdio transport
    secretRefs:
      - name: levan-slack-credentials
  transportType: "stdio"            # <-- stdio transport
```

**How it works:**
- kagent launches the container
- Communicates via **stdin/stdout** (stdio)
- Process stays running continuously
- Direct process-to-process communication

---

### F5 MCP Server (HTTP transport)

```yaml
apiVersion: kagent.dev/v1alpha2      # <-- v1alpha2
kind: RemoteMCPServer                # <-- RemoteMCPServer (not MCPServer)
metadata:
  name: f5-mcp-remote
spec:
  description: F5 BIG-IP MCP Server
  url: http://f5-mcp-server.kagent.svc.cluster.local:8081/mcp
  protocol: STREAMABLE_HTTP           # <-- HTTP transport
  timeout: 30s
  terminateOnClose: true
```

**How it works:**
- You deploy your own Deployment + Service
- F5 MCP server runs as a standalone HTTP service
- kagent makes **HTTP requests** to the service
- More flexible, can scale horizontally

---

## Key Differences

| Feature | Slack Bot (stdio) | F5 MCP (HTTP) |
|---------|-------------------|---------------|
| **Resource Type** | `MCPServer` (v1alpha1) | `RemoteMCPServer` (v1alpha2) |
| **Transport** | stdio | STREAMABLE_HTTP |
| **Deployment** | Managed by kagent | You manage Deployment/Service |
| **Communication** | stdin/stdout | HTTP POST requests |
| **Scalability** | Single process | Can scale replicas |
| **External Access** | No | Yes (via Service) |
| **Use Case** | Integrated services | Remote/external services |

---

## Your F5 Setup

Yes, you are hosting a **Remote MCP Server** that:

1. ✅ Runs as a standalone HTTP service in your Kubernetes cluster
2. ✅ Connects to your F5 device at `https://172.16.10.10`
3. ✅ Exposes MCP tools via HTTP endpoint
4. ✅ kagent discovers and uses these tools via HTTP
5. ✅ Can be accessed by multiple AI assistants simultaneously
6. ✅ Can be scaled to multiple replicas if needed

---

## Request Flow Example

```
User: "List all pools on F5"
  │
  ▼
AI Assistant (via kagent)
  │
  │ HTTP POST
  │ {
  │   "method": "tools/call",
  │   "params": {
  │     "name": "obj_list",
  │     "arguments": {"obj_type": "pool"}
  │   }
  │ }
  ▼
f5-mcp-server Service (ClusterIP)
  │
  ▼
f5-mcp-server Pod
  │ Uses credentials from Secret
  │
  │ HTTPS GET /mgmt/tm/ltm/pool
  ▼
F5 BIG-IP (172.16.10.10)
  │
  │ Returns pool data
  ▼
f5-mcp-server Pod
  │ Formats response
  ▼
kagent
  │
  ▼
AI Assistant
  │
  ▼
User sees: "You have 3 pools: web-pool, app-pool, db-pool"
```

---

## Why Remote MCP Server?

The F5 MCP server uses `RemoteMCPServer` because:

1. **F5 is external** - Your F5 device is outside the cluster
2. **HTTP makes sense** - REST API communication fits HTTP transport
3. **Reusability** - Other services can also call the MCP server
4. **Scalability** - Can run multiple replicas for high availability
5. **Separation of concerns** - MCP server manages F5 connection, kagent manages AI tools

This is the right architecture for integrating with external infrastructure like F5 devices!

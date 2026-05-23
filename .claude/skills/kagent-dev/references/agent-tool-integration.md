# Agent Tool Integration Reference

This document covers how tools are registered, validated, and invoked within kagent agents.

## Overview

Tools in kagent follow the Model Context Protocol (MCP) pattern. Each tool is defined by a schema, registered with an agent, and invoked via the tool execution pipeline.

## Tool Registration Flow

```
ToolConfig (CRD) → ToolProvider → Agent.Spec.Tools → Runtime Tool Registry
```

### 1. Defining a Tool via CRD

```yaml
apiVersion: kagent.dev/v1alpha1
kind: Tool
metadata:
  name: kubectl-get
  namespace: kagent
spec:
  description: "Run kubectl get commands against the cluster"
  inputSchema:
    type: object
    properties:
      resource:
        type: string
        description: "Kubernetes resource type (e.g., pods, deployments)"
      namespace:
        type: string
        description: "Namespace to query (optional)"
      name:
        type: string
        description: "Specific resource name (optional)"
    required:
      - resource
  provider:
    type: exec
    exec:
      command: ["kubectl", "get"]
```

### 2. Tool Provider Types

| Provider | Description | Use Case |
|----------|-------------|----------|
| `exec`   | Runs a local binary | kubectl, helm, git |
| `mcp`    | Connects to an MCP server | External tool servers |
| `builtin`| Built into kagent runtime | HTTP requests, file I/O |
| `python` | Executes Python scripts | Custom logic |

## Tool Execution Pipeline

### Request Path

1. LLM generates a `tool_call` in its response
2. Agent runtime parses the `tool_call` JSON
3. Tool name is resolved against the registry
4. Input is validated against the tool's JSON schema
5. Tool provider is invoked with validated input
6. Output is captured and returned as a `tool_result` message
7. Conversation continues with the tool result in context

### Error Handling

Tool execution errors are categorized:

- **ValidationError**: Input did not match schema — returned to LLM as a soft error so it can retry with corrected input
- **ExecutionError**: Provider failed (non-zero exit, timeout) — returned with stderr/stdout for LLM context
- **TimeoutError**: Tool exceeded `spec.timeout` — agent receives a timeout message and may retry or escalate

```go
// ToolResult represents the outcome of a tool invocation
type ToolResult struct {
    ToolCallID string `json:"tool_call_id"`
    Content    string `json:"content"`
    IsError    bool   `json:"is_error,omitempty"`
}
```

## Adding a New Built-in Tool

### Step 1: Implement the ToolHandler interface

```go
type ToolHandler interface {
    Name() string
    Description() string
    InputSchema() map[string]interface{}
    Execute(ctx context.Context, input map[string]interface{}) (string, error)
}
```

### Step 2: Register in the tool registry

```go
// In pkg/tools/registry.go
func init() {
    Register(&KubectlGetTool{})
    Register(&HTTPRequestTool{})
    // Add new tool here
    Register(&MyNewTool{})
}
```

### Step 3: Write unit tests

```go
func TestMyNewTool_Execute(t *testing.T) {
    tool := &MyNewTool{}
    result, err := tool.Execute(context.Background(), map[string]interface{}{
        "param": "value",
    })
    require.NoError(t, err)
    assert.Contains(t, result, "expected output")
}
```

## MCP Tool Server Integration

kagent can connect to external MCP servers to expose additional tools.

```yaml
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata:
  name: filesystem-tools
  namespace: kagent
spec:
  transport: stdio
  command: ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
  env:
    - name: NODE_ENV
      value: production
```

Once an MCPServer is registered, its tools are automatically discoverable by agents that reference it:

```yaml
# In Agent spec
spec:
  tools:
    - mcpServer: filesystem-tools   # All tools from this MCP server
    - name: kubectl-get             # Specific named tool
```

## Tool Timeout Configuration

Default tool timeout is **30 seconds**. Override per-tool:

```yaml
spec:
  timeout: 120s   # 2 minutes for long-running operations
```

Or globally in the AgentConfig:

```yaml
spec:
  defaultToolTimeout: 60s
```

## Debugging Tool Calls

### Enable verbose tool logging

```bash
kubectl set env deployment/kagent-controller TOOL_LOG_LEVEL=debug -n kagent
```

### Inspect tool call history

Tool calls are stored in the `AgentRun` status:

```bash
kubectl get agentrun <run-id> -n kagent -o jsonpath='{.status.messages}' | jq '.[] | select(.role=="tool")'
```

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Tool not found | Name mismatch in agent spec | Verify `kubectl get tools -n kagent` |
| Schema validation failure | Wrong input types from LLM | Add examples to tool description |
| Exec tool permission denied | Missing RBAC or binary not in PATH | Check pod securityContext and PATH env |
| MCP server not connecting | Server crashed on startup | Check `kubectl logs` on MCP sidecar |

## Related References

- [CRD Workflow Detailed](./crd-workflow-detailed.md)
- [E2E Debugging](./e2e-debugging.md)
- [CI Failures](./ci-failures.md)

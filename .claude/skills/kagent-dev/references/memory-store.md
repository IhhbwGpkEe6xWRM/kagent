# Memory Store in kagent

This document describes the memory store subsystem used by kagent agents to persist and retrieve conversational context, tool results, and agent state across sessions.

## Overview

kagent uses a layered memory architecture:

1. **In-memory (ephemeral)** — per-request context held in Go structs
2. **CRD-backed (persistent)** — agent sessions stored as Kubernetes custom resources
3. **Vector store (semantic)** — optional embedding-based retrieval for long-term memory

---

## Core Types

### `MemoryEntry`

```go
// MemoryEntry represents a single unit of stored memory.
type MemoryEntry struct {
    ID        string            `json:"id"`
    AgentID   string            `json:"agentId"`
    SessionID string            `json:"sessionId"`
    Role      string            `json:"role"`      // "user", "assistant", "tool"
    Content   string            `json:"content"`
    ToolName  string            `json:"toolName,omitempty"`
    Metadata  map[string]string `json:"metadata,omitempty"`
    CreatedAt time.Time         `json:"createdAt"`
}
```

### `MemoryStore` interface

```go
type MemoryStore interface {
    // Append adds a new entry to the session history.
    Append(ctx context.Context, entry MemoryEntry) error

    // List returns all entries for a given session, ordered by CreatedAt.
    List(ctx context.Context, agentID, sessionID string) ([]MemoryEntry, error)

    // Trim removes oldest entries beyond maxEntries for a session.
    Trim(ctx context.Context, agentID, sessionID string, maxEntries int) error

    // Delete removes all memory for a session.
    Delete(ctx context.Context, agentID, sessionID string) error
}
```

---

## CRD-Backed Implementation

The default `CRDMemoryStore` persists entries inside the `AgentSession` CRD status field.

```go
type CRDMemoryStore struct {
    client  client.Client
    scheme  *runtime.Scheme
    maxSize int // max entries before auto-trim
}

func NewCRDMemoryStore(c client.Client, s *runtime.Scheme) *CRDMemoryStore {
    return &CRDMemoryStore{
        client:  c,
        scheme:  s,
        maxSize: 200,
    }
}
```

### Append flow

1. Fetch the `AgentSession` CR by `agentID + sessionID`.
2. Append the serialized `MemoryEntry` to `status.history`.
3. If `len(history) > maxSize`, call `Trim` automatically.
4. Patch the CR status using `client.Status().Patch()`.

```go
func (s *CRDMemoryStore) Append(ctx context.Context, entry MemoryEntry) error {
    session := &v1alpha1.AgentSession{}
    key := types.NamespacedName{Name: entry.SessionID, Namespace: entry.AgentID}
    if err := s.client.Get(ctx, key, session); err != nil {
        return fmt.Errorf("memory store: get session: %w", err)
    }

    raw, err := json.Marshal(entry)
    if err != nil {
        return fmt.Errorf("memory store: marshal entry: %w", err)
    }

    patch := client.MergeFrom(session.DeepCopy())
    session.Status.History = append(session.Status.History, runtime.RawExtension{Raw: raw})

    if len(session.Status.History) > s.maxSize {
        session.Status.History = session.Status.History[len(session.Status.History)-s.maxSize:]
    }

    return s.client.Status().Patch(ctx, session, patch)
}
```

---

## In-Memory Implementation (Testing)

For unit tests and local development, use `InMemoryStore`:

```go
type InMemoryStore struct {
    mu      sync.RWMutex
    entries map[string][]MemoryEntry // key: agentID+":"+sessionID
}

func NewInMemoryStore() *InMemoryStore {
    return &InMemoryStore{entries: make(map[string][]MemoryEntry)}
}

func (s *InMemoryStore) Append(_ context.Context, e MemoryEntry) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    key := e.AgentID + ":" + e.SessionID
    s.entries[key] = append(s.entries[key], e)
    return nil
}
```

---

## Configuration

Memory store behaviour is controlled via the `AgentSpec`:

```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: my-agent
spec:
  memory:
    backend: crd          # "crd" | "inmemory"
    maxHistoryEntries: 100
    trimStrategy: oldest  # "oldest" | "summarize" (summarize requires LLM)
```

| Field | Default | Description |
|---|---|---|
| `backend` | `crd` | Storage backend for session history |
| `maxHistoryEntries` | `200` | Hard cap on stored entries per session |
| `trimStrategy` | `oldest` | How to reduce history when cap is reached |

---

## Testing Memory Store

```go
func TestInMemoryStore_AppendAndList(t *testing.T) {
    store := NewInMemoryStore()
    ctx := context.Background()

    entry := MemoryEntry{
        ID: "e1", AgentID: "agent-1", SessionID: "sess-1",
        Role: "user", Content: "hello", CreatedAt: time.Now(),
    }
    require.NoError(t, store.Append(ctx, entry))

    entries, err := store.List(ctx, "agent-1", "sess-1")
    require.NoError(t, err)
    require.Len(t, entries, 1)
    assert.Equal(t, "hello", entries[0].Content)
}
```

---

## Common Issues

### History grows unbounded

Ensure `maxHistoryEntries` is set in the `AgentSpec`. The default is 200 but long-running agents may exceed this quickly with tool-heavy workflows.

### Stale reads after Append

The CRD store uses optimistic concurrency. If two goroutines append simultaneously, one will receive a conflict error. The caller should retry with exponential backoff:

```go
retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    return memStore.Append(ctx, entry)
})
```

### Session not found

The `AgentSession` CR must exist before `Append` is called. The session controller creates it on first invocation. If you see `"memory store: get session: not found"`, check that the session reconciler is running.

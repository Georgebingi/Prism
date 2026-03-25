# WebSocket Streaming for Trace Updates

This document describes the WebSocket streaming implementation for real-time trace updates in Prism.

## Overview

Large traces can take time to process. The WebSocket streaming feature allows the Web UI to build the execution graph incrementally as trace nodes are resolved, providing better responsiveness and user experience.

## Architecture

The streaming implementation consists of three main components:

1. **Rust WebSocket Server** (`prism serve` command)
   - Accepts WebSocket connections from clients
   - Receives trace requests with transaction hashes
   - Streams trace events as they are generated during replay
   - Handles multiple concurrent trace sessions

2. **TypeScript Server Integration** (`apps/server`)
   - Optional proxy layer for additional processing
   - Can be used to add authentication, rate limiting, etc.

3. **React Web UI** (`apps/web`)
   - Connects to WebSocket server
   - Receives and displays trace updates in real-time
   - Shows progress indicators and resource consumption

## Usage

### Starting the WebSocket Server

```bash
# Start the Prism WebSocket server
prism serve --port 8080 --host 127.0.0.1

# With custom network
prism serve --network mainnet --port 8080
```

The server will listen for WebSocket connections and stream trace updates to connected clients.

### Client Connection

The Web UI automatically connects to the WebSocket server when viewing the trace page:

```typescript
// In your React component
const wsUrl = `ws://localhost:8080`;
const { trace, loading, requestTrace, streaming } = useTrace(wsUrl);

// Request a trace
requestTrace(txHash, network);
```

### Message Protocol

#### Client → Server

Request a trace:
```json
{
  "tx_hash": "abc123..."
}
```

#### Server → Client

The server sends a stream of messages as the trace is processed:

**Trace Started**
```json
{
  "type": "trace_started",
  "tx_hash": "abc123...",
  "ledger_sequence": 12345
}
```

**Trace Node** (sent for each resolved node)
```json
{
  "type": "trace_node",
  "node": {
    "event_type": "InvocationStart",
    "timestamp_us": 1234567,
    "data": { ... }
  },
  "path": [0, 1, 2]
}
```

**Resource Update** (periodic updates)
```json
{
  "type": "resource_update",
  "cpu_used": 50000000,
  "memory_used": 10485760,
  "cpu_limit": 100000000,
  "memory_limit": 41943040
}
```

**State Diff Entry** (for each state change)
```json
{
  "type": "state_diff_entry",
  "key": "Contract(abc...)",
  "before": "...",
  "after": "...",
  "change_type": "Updated"
}
```

**Trace Completed**
```json
{
  "type": "trace_completed",
  "total_nodes": 150,
  "duration_ms": 2500
}
```

**Trace Error**
```json
{
  "type": "trace_error",
  "error": "Failed to reconstruct state: ..."
}
```

## Benefits

1. **Incremental Display**: Users see trace nodes as they're computed, not after a long wait
2. **Progress Feedback**: Real-time resource consumption and node count updates
3. **Better UX**: No more staring at a loading spinner for large traces
4. **Scalability**: Handles large traces without overwhelming the client
5. **Responsiveness**: UI remains interactive during trace processing

## Implementation Details

### Rust Server (`crates/cli/src/commands/serve.rs`)

- Uses `tokio-tungstenite` for WebSocket support
- Spawns a separate task for each trace request
- Uses `broadcast` channels to stream updates from replay engine to WebSocket
- Handles connection lifecycle (connect, disconnect, errors)

### React Hooks

**`useWebSocket`** - Low-level WebSocket connection management
- Handles connection lifecycle
- Parses incoming messages
- Provides callbacks for different message types

**`useTrace`** - High-level trace data management
- Accumulates trace nodes as they arrive
- Updates resource profile in real-time
- Collects state diff entries
- Provides loading and error states

### Components

**`ExecutionTimeline`** - Displays trace nodes incrementally
- Shows nodes as they arrive
- Provides path information for nested invocations
- Expandable node details

**`ResourceProfile`** - Real-time resource consumption
- CPU and memory usage bars
- Updates as trace progresses
- Warnings when approaching limits

**`StateDiffViewer`** - State changes as they're discovered
- Color-coded by change type (created/updated/deleted)
- Side-by-side before/after comparison

## Configuration

### Server Options

```bash
prism serve --help

Options:
  -p, --port <PORT>      Port to listen on [default: 8080]
      --host <HOST>      Host to bind to [default: 127.0.0.1]
  -n, --network <NET>    Network: mainnet, testnet, futurenet [default: testnet]
  -v, --verbose          Enable verbose logging
```

### Client Configuration

The WebSocket URL is configurable in the Web UI:

```typescript
// Default: connect to same host as web app
const wsUrl = `ws://${window.location.hostname}:8080`;

// Or specify explicitly
const wsUrl = "ws://localhost:8080";
```

## Error Handling

The implementation includes comprehensive error handling:

1. **Connection Errors**: Displayed in UI with fallback to REST API
2. **Trace Errors**: Sent as `trace_error` messages with details
3. **Network Errors**: Automatic reconnection attempts
4. **Timeout Handling**: Configurable timeouts for long-running traces

## Performance Considerations

- **Throttling**: Small delays between node emissions to avoid overwhelming clients
- **Batching**: Resource updates sent periodically, not for every node
- **Memory**: Broadcast channels have bounded capacity (100 messages)
- **Concurrency**: Each trace session runs in a separate task

## Future Enhancements

- [ ] Authentication and authorization
- [ ] Rate limiting per client
- [ ] Trace session persistence
- [ ] Pause/resume streaming
- [ ] Selective node filtering
- [ ] Compression for large payloads
- [ ] Metrics and monitoring

## Testing

### Manual Testing

1. Start the WebSocket server:
   ```bash
   prism serve --port 8080
   ```

2. Open the Web UI and navigate to the trace page

3. Enter a transaction hash and click "Trace Transaction"

4. Observe trace nodes appearing incrementally

### Integration Testing

```bash
# Test WebSocket connection
wscat -c ws://localhost:8080

# Send trace request
{"tx_hash": "abc123..."}

# Observe streaming messages
```

## Troubleshooting

**WebSocket connection fails**
- Ensure `prism serve` is running
- Check firewall settings
- Verify port is not in use

**No trace updates received**
- Check browser console for errors
- Verify transaction hash is valid
- Check server logs with `--verbose` flag

**Slow streaming**
- Large traces may take time to process
- Check network latency
- Consider increasing throttling delay

## Related Files

- `crates/cli/src/commands/serve.rs` - WebSocket server implementation
- `apps/web/src/hooks/useWebSocket.ts` - WebSocket client hook
- `apps/web/src/hooks/useTrace.ts` - Trace data management
- `apps/web/src/app/trace/page.tsx` - Trace page UI
- `apps/web/src/components/trace/*` - Trace visualization components

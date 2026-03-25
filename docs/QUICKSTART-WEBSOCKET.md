# Quick Start: WebSocket Trace Streaming

Get up and running with real-time trace streaming in 5 minutes.

## Prerequisites

- Rust 1.77+ installed
- Node.js 18+ (for web UI)
- A Soroban transaction hash to trace

## Step 1: Build Prism

```bash
# Clone the repository (if you haven't already)
git clone https://github.com/prism-soroban/prism.git
cd prism

# Build the CLI with WebSocket support
cargo build --release

# The binary will be at target/release/prism
```

## Step 2: Start the WebSocket Server

```bash
# Start the server on default port 8080
./target/release/prism serve

# Or with custom options
./target/release/prism serve --port 9000 --network mainnet --verbose
```

You should see:
```
🚀 Prism WebSocket server listening on ws://127.0.0.1:8080
   Ready to stream trace updates to connected clients
   Press Ctrl+C to stop
```

## Step 3: Test with the Example Client

```bash
# Install dependencies
npm install ws

# Run the example client with a transaction hash
node examples/websocket-client.js <YOUR_TX_HASH>
```

Example output:
```
Connecting to ws://localhost:8080...
✓ Connected to Prism WebSocket server
Requesting trace for: abc123...

🚀 Trace started
   Transaction: abc123...
   Ledger: 12345

📦 Received 50 trace nodes...
📊 Resources: CPU 45.2%, Memory 12.8%
📦 Received 100 trace nodes...
📝 State change: Contract(xyz...) (Updated)

✅ Trace completed!
   Total nodes: 150
   Server duration: 2500ms
   Client duration: 2503ms
```

## Step 4: Use the Web UI

```bash
# Navigate to the web app directory
cd apps/web

# Install dependencies
npm install

# Start the development server
npm run dev
```

Open http://localhost:3000/trace in your browser:

1. Enter a transaction hash
2. Select the network (testnet/mainnet/futurenet)
3. Click "Trace Transaction"
4. Watch the trace build incrementally in real-time!

## What You'll See

### Real-Time Updates

- **Trace nodes** appear as they're computed
- **Resource consumption** updates every few nodes
- **State changes** stream in as they're discovered
- **Progress indicators** show current status

### Visual Feedback

- Green progress bars for healthy resource usage
- Yellow/red warnings when approaching limits
- Color-coded state changes (created/updated/deleted)
- Node count and timing information

## Architecture Overview

```
┌─────────────┐         WebSocket          ┌──────────────┐
│  Web UI     │ ◄─────────────────────────► │ prism serve  │
│  (React)    │    JSON messages            │  (Rust)      │
└─────────────┘                             └──────────────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │ Replay Engine│
                                            │ (prism-core) │
                                            └──────────────┘
```

## Message Flow

1. Client connects to WebSocket server
2. Client sends trace request: `{"tx_hash": "..."}`
3. Server starts replay and streams updates:
   - `trace_started` - Initial metadata
   - `trace_node` - Each resolved node (×N)
   - `resource_update` - Periodic resource stats
   - `state_diff_entry` - Each state change
   - `trace_completed` - Final summary
4. Client displays updates incrementally

## Common Issues

### "Connection refused"

Make sure `prism serve` is running:
```bash
./target/release/prism serve
```

### "Transaction not found"

Verify the transaction hash and network:
```bash
# Check on Stellar Expert
https://stellar.expert/explorer/testnet/tx/<YOUR_TX_HASH>
```

### Slow streaming

Large traces take time to process. This is normal! The streaming allows you to see progress instead of waiting for completion.

## Next Steps

- Read the [full WebSocket documentation](./websocket-streaming.md)
- Explore the [trace visualization components](../apps/web/src/components/trace/)
- Check out the [CLI commands](../README.md#how-it-works)
- Join the Discord for support: emry_ss

## Configuration Options

### Server

```bash
prism serve --help

Start WebSocket server for streaming trace updates

Options:
  -p, --port <PORT>      Port to listen on [default: 8080]
      --host <HOST>      Host to bind to [default: 127.0.0.1]
  -n, --network <NET>    Network [default: testnet]
  -v, --verbose          Enable verbose logging
```

### Client (Web UI)

Edit `apps/web/src/app/trace/page.tsx`:

```typescript
// Change WebSocket URL
const wsUrl = "ws://your-server:8080";
```

## Performance Tips

1. **Use verbose logging** to debug issues: `prism serve --verbose`
2. **Monitor resource usage** in the UI to identify bottlenecks
3. **Test with small transactions** first to verify setup
4. **Check network latency** if streaming seems slow

## Security Notes

- The WebSocket server currently has no authentication
- Only use on trusted networks or localhost
- Don't expose the server to the public internet
- Future versions will add auth and rate limiting

## Feedback

Found a bug or have a suggestion? Reach out on Discord: emry_ss

Happy tracing! 🚀

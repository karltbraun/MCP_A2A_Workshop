# Building Your First MCP Server — MQTT & UNS

## Overview

This section walks through building a functional MCP server from scratch in Python that connects to the Flexible Packager Unified Namespace (UNS) via MQTT.

## Status: ✅ Complete

All 4 tools implemented and tested:
- ✅ `list_uns_topics` - Discover available topics
- ✅ `get_topic_value` - Read specific topic values
- ✅ `search_topics` - Search topics by pattern/keyword
- ✅ `publish_message` - Write messages to topics

## Goals

- Build a functional MCP server from scratch in Python
- Connect to live MQTT broker (Flexible Packager UNS)
- Configure Claude Desktop to use custom MCP server
- Demonstrate read and write capabilities to the UNS

## Prerequisites

- Python 3.10+
- Claude Desktop installed
- Access to HiveMQ broker at balancer.virtualfactory.online:1883
- Basic understanding of MQTT and UNS concepts

---

## Project Setup

### 1. Project Structure

```
MCP_A2A_Workshop/
├── .env                    # Credentials (root level, gitignored)
├── .env.example            # Template for students
└── day1/
    └── mqtt_server/
        ├── README.md
        ├── requirements.txt
        ├── venv/           # Virtual environment (gitignored)
        └── src/
            ├── mqtt_mcp_server.py
            └── mqtt_cache.json  # Runtime cache (gitignored)
```

### 2. Environment Configuration

Credentials are stored in the root `.env` file. Copy from template if needed:

```bash
cp .env.example .env
# Edit .env with your credentials
```

The `.env` file contains:

```
MQTT_BROKER=balancer.virtualfactory.online
MQTT_PORT=1883
MQTT_USERNAME=your_username
MQTT_PASSWORD=your_password
```

**Note:** The MQTT client ID is automatically generated with a unique suffix (e.g., `mcp-mqtt-a1b2c3d4`) to allow multiple MCP server instances to run simultaneously without conflicts.

### 3. Initialize Virtual Environment

```bash
cd MCP_A2A_Workshop/day1/mqtt_server
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

### 4. Install Dependencies

```bash
pip install -r requirements.txt
```

---

## MCP Server Anatomy

Every MCP server requires four components:

| Component | Purpose |
|-----------|---------|
| Server Instance | The main MCP server object |
| Tools | Actions the agent can take (functions) |
| Resources | Data the agent can read (static or dynamic) |
| Transport | How the client connects -- we use stdio |

### Key Concepts

- **Tools** are functions Claude can call -- like `get_topic_value` or `publish_message`
- **Resources** are data endpoints Claude can read -- like a list of available topics
- **Transport** defines the communication method -- stdio is standard for Claude Desktop

---

## Connecting to the MQTT Broker

### Broker Details

| Setting | Value |
|---------|-------|
| Host | balancer.virtualfactory.online |
| Port | 1883 |
| Protocol | MQTT 3.1.1 |

### Flexible Packager UNS Topic Structure

The UNS follows ISA-95 hierarchy:

```
enterprise/
└── site/
    └── area/
        └── line/
            └── cell/
                └── [tag_name]
```

Example topics:
- `flexpack/packaging/line1/filler/speed`
- `flexpack/packaging/line1/filler/status`
- `flexpack/packaging/line1/labeler/count`

---

## Implemented Tools

### Tool 1 -- list_uns_topics ✅

Lists all topics currently cached from the UNS.

**Purpose:** Let Claude explore what data is available

**Inputs:**
- `base_path` (optional): Filter by topic prefix (e.g., `flexpack/packaging`)

**Output:** List of cached topic paths with current values (instant response)

### Tool 2 -- get_topic_value ✅

Gets the cached value for a specific topic.

**Purpose:** Let Claude read a specific data point

**Inputs:**
- `topic` (required): Full topic path (e.g., `flexpack/packaging/line1/filler/speed`)

**Output:** Current value, timestamp, and age

### Tool 3 -- search_topics ✅

Searches cached topics matching a pattern or keyword.

**Purpose:** Let Claude find relevant topics without knowing exact paths

**Inputs:**
- `pattern` (required): Search string, glob pattern, or MQTT wildcard

**Output:** List of matching topics with values (instant response)

---

## Configuring Claude Desktop

### 1. Locate Config File

| OS | Path |
|----|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

### 2. Add Server Entry

Use the **venv Python interpreter** to ensure dependencies are available:

```json
{
  "mcpServers": {
    "mqtt-uns": {
      "command": "/full/path/to/MCP_A2A_Workshop/day1/mqtt_server/venv/bin/python",
      "args": ["/full/path/to/MCP_A2A_Workshop/day1/mqtt_server/src/mqtt_mcp_server.py"]
    }
  }
}
```

**Note:** The server loads credentials from the root `.env` file automatically — no need to pass environment variables in the config.

### 3. Restart Claude Desktop

- Quit Claude Desktop completely
- Relaunch the application
- Verify server appears in the MCP server list (hammer icon)

---

## Live Demo -- Querying the UNS

### Test Prompts

1. **Discover available data:**
   > "What topics are available in the UNS?"

2. **Read a specific value:**
   > "What is the current value of flexpack/packaging/line1/filler/speed?"

3. **Search for related data:**
   > "Find all topics related to temperature"

4. **Contextual query:**
   > "What's the status of line 1?"

### What to Observe

- Claude selects the appropriate tool for each question
- Tool calls appear in Claude Desktop's tool use panel
- Results return as structured data Claude can reason about

---

### Tool 4 -- publish_message ✅

Publishes a value to a specified topic on the broker.

**Purpose:** Let Claude write data back to the UNS

**Inputs:**
- `topic` (required): Full topic path to publish to (no wildcards)
- `payload` (required): Message payload (string)
- `retain` (optional): Retain message on broker, default false
- `qos` (optional): Quality of Service 0/1/2, default 1

**Output:** Confirmation with topic, payload, message ID, and timestamp

### Safety Features Implemented

| Feature | Status |
|---------|--------|
| Validate topic paths (no wildcards) | ✅ |
| Log all publish operations | ✅ |
| Return confirmation with details | ✅ |
| Topic allowlist | ❌ Not implemented (add if needed) |

---

## Live Demo -- Publishing to the UNS

### Test Prompts

1. **Simple publish:**
   > "Publish a test message to flexpack/test/claude with value 'hello from claude'"

2. **Verify receipt:**
   > Use MQTT Explorer or broker logs to confirm message arrived

### Discussion Points

- When is write access appropriate?
- How do we prevent accidental writes to production topics?
- Role of topic namespacing for safety (e.g., `/agent/` prefix)

---

## Checkpoint

By the end of this section, you have:

- [x] A working MCP server in Python
- [x] Connection to live MQTT broker
- [x] Tools for reading UNS topics (list_uns_topics, get_topic_value, search_topics)
- [x] Tools for writing to UNS topics (publish_message)
- [x] Claude Desktop configured to use the server
- [ ] Demonstrated natural language queries against industrial data

**Next:** Add a second MCP server for MySQL database access

---

## Files in This Section

| File | Purpose |
|------|---------|
| `README.md` | This guide |
| `requirements.txt` | Python dependencies |
| `src/mqtt_mcp_server.py` | Main MCP server implementation |
| `src/mqtt_cache.json` | Runtime cache file (gitignored, auto-generated) |

**Note:** Credentials are in the root `.env` file, not in this directory.

---

## Implementation Notes

### Caching Architecture

The server uses a flat-file cache for instant topic lookups:

```
┌─────────────────┐     subscribe #     ┌─────────────────┐
│   MQTT Broker   │ ──────────────────► │   MCP Server    │
│                 │ ◄────────────────── │                 │
└─────────────────┘    all messages     └────────┬────────┘
                                                 │
                                          write on msg
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │ mqtt_cache.json │
                                        │  {topic: value} │
                                        └────────┬────────┘
                                                 │
                                            read from
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │   Tool Calls    │
                                        │ (instant reads) │
                                        └─────────────────┘
```

**How it works:**
1. On connect: Subscribe to `#` (all topics)
2. On message: Update `mqtt_cache.json` with topic → value
3. On tool call: Read directly from cache file (instant)
4. On disconnect: Clear cache file

**Cache file format:**
```json
{
  "flexpack/packaging/line1/filler/speed": {
    "value": "125.5",
    "timestamp": 1702742400.123
  }
}
```

### Dynamic Client IDs

The server automatically generates unique MQTT client IDs to allow multiple MCP server instances to run simultaneously (e.g., Claude Desktop + manual testing):

```python
# Generates: mcp-mqtt-a1b2c3d4 (unique suffix each time)
MQTT_CLIENT_ID = f"mcp-mqtt-{uuid.uuid4().hex[:8]}"
```

This prevents "session taken over" disconnections that occur when two clients use the same ID.

### Reconnection Handling

Built-in exponential backoff prevents rapid reconnection cycling:

```python
self.client.reconnect_delay_set(min_delay=1, max_delay=120)
```

### Logging

All logs go to stderr (stdout reserved for MCP protocol). Run manually to see connection status:

```bash
python src/mqtt_mcp_server.py
```

---

## Troubleshooting

### Server doesn't appear in Claude Desktop

- Check config file syntax (valid JSON)
- Verify full path to venv Python: `/path/to/venv/bin/python`
- Check Claude Desktop logs for errors

### MQTT connection fails

- Verify broker hostname and port
- Check credentials in root `.env`
- Test connection with mqtt client directly
- Check for "Session taken over" in logs (client ID collision)

### Tools not working

- Check MCP server logs for errors
- Verify tool registration syntax
- Test MQTT connection independently first

### Rapid disconnect/reconnect

- Usually caused by client ID collision (another instance with same ID)
- The server now generates unique IDs automatically
- If persists, check if MQTT_CLIENT_ID in `.env` is being shared

---

## Resources

- [MCP Documentation](https://modelcontextprotocol.io)
- [Paho MQTT Python](https://eclipse.dev/paho/files/paho.mqtt.python/html/)
- [HiveMQ MQTT Essentials](https://www.hivemq.com/mqtt-essentials/)

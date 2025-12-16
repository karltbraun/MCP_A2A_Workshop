# Building Your First MCP Server — MQTT & UNS

## Overview

This section walks through building a functional MCP server from scratch in Python that connects to the Flexible Packager Unified Namespace (UNS) via MQTT.

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
        └── src/
            └── mqtt_mcp_server.py
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

## Building Read Tools

### Tool 1 -- list_topics

Discovers available topics in the UNS by subscribing to wildcard and collecting messages.

**Purpose:** Let Claude explore what data is available

**Input:** Optional base path (default: `#` for all topics)

**Output:** List of discovered topic paths

### Tool 2 -- get_topic_value

Reads the current retained value from a specific topic.

**Purpose:** Let Claude read a specific data point

**Input:** Full topic path (e.g., `flexpack/packaging/line1/filler/speed`)

**Output:** Current value and timestamp

### Tool 3 -- search_topics

Finds topics matching a pattern or keyword.

**Purpose:** Let Claude find relevant topics without knowing exact paths

**Input:** Search pattern or keyword (e.g., `temperature`, `line1/*`)

**Output:** List of matching topics

---

## Configuring Claude Desktop

### 1. Locate Config File

| OS | Path |
|----|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

### 2. Add Server Entry

```json
{
  "mcpServers": {
    "mqtt-uns": {
      "command": "python",
      "args": ["/full/path/to/MCP_A2A_Workshop/day1/mqtt_server/src/mqtt_mcp_server.py"],
      "env": {
        "MQTT_BROKER": "balancer.virtualfactory.online",
        "MQTT_PORT": "1883"
      }
    }
  }
}
```

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

## Adding Write Tools

### Tool 4 -- publish_message

Publishes a value to a specified topic on the broker.

**Purpose:** Let Claude write data back to the UNS

**Input:** 
- Topic path
- Message payload
- Retain flag (optional, default: false)
- QoS level (optional, default: 1)

**Output:** Confirmation of publish success

### Safety Considerations

- Validate topic paths before publishing
- Consider adding allowlist of writable topics
- Log all publish operations
- Add confirmation prompts for destructive operations

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

- [ ] A working MCP server in Python
- [ ] Connection to live MQTT broker
- [ ] Tools for reading UNS topics
- [ ] Tools for writing to UNS topics
- [ ] Claude Desktop configured to use the server
- [ ] Demonstrated natural language queries against industrial data

**Next:** Add a second MCP server for MySQL database access

---

## Files in This Section

| File | Purpose |
|------|---------|
| `README.md` | This guide |
| `requirements.txt` | Python dependencies |
| `src/mqtt_mcp_server.py` | Main MCP server implementation (built with Cursor) |

**Note:** Credentials are in the root `.env` file, not in this directory.

---

## Troubleshooting

### Server doesn't appear in Claude Desktop

- Check config file syntax (valid JSON)
- Verify full path to Python script
- Check Claude Desktop logs for errors

### MQTT connection fails

- Verify broker hostname and port
- Check credentials in root .env
- Test connection with mqtt client directly

### Tools not working

- Check MCP server logs for errors
- Verify tool registration syntax
- Test MQTT connection independently first

---

## Resources

- [MCP Documentation](https://modelcontextprotocol.io)
- [Paho MQTT Python](https://eclipse.dev/paho/files/paho.mqtt.python/html/)
- [HiveMQ MQTT Essentials](https://www.hivemq.com/mqtt-essentials/)

# Shodan MCP Server

A fully async, production-grade MCP server that exposes the complete Shodan API (REST, Streaming, and Trends) as MCP tools. Built with Python, `httpx`, and the MCP Python SDK, this server enables seamless integration of Shodan's powerful internet intelligence tools into your automation, research, and security workflows.

**Now supports both STDIO and StreamableHTTP transports!**

## Features

- **Full Shodan API Coverage:**
  - REST API: Host info, search, DNS, datasets, account, tools, and more
  - Streaming API: Real-time firehose, port, and alert event streams
  - Trends API: Top ports, organizations, and countries for any query
- **Async & High Performance:**
  - Built on `httpx` and async/await for maximum concurrency and speed
- **Secure Authentication:**
  - API keys are read from environment variables for each API type
- **Multiple Transport Options:**
  - **STDIO**: For Claude Desktop and CLI integration
  - **StreamableHTTP**: For web-based clients and remote access with SSE support
- **Easy Integration:**
  - Ready to use with Claude Desktop, CLI, or any MCP-compatible client

## Environment Variables

Set your Shodan API credentials as environment variables before running the server:

- `SHODAN_API_KEY` (default for all APIs)
- `SHODAN_STREAM_API_KEY` (optional, for Streaming API)
- `SHODAN_TRENDS_API_KEY` (optional, for Trends API)

If a specific key is not set, the server will fall back to `SHODAN_API_KEY`.

## Usage

### 1. Install dependencies

```sh
uv add "mcp[cli]>=1.12.0"
uv add "httpx>=0.25.0"
uv add "uvicorn>=0.24.0"
```

### 2. Set your environment variables

```sh
export SHODAN_API_KEY=your_main_shodan_api_key
# Optionally:
export SHODAN_STREAM_API_KEY=your_streaming_key
export SHODAN_TRENDS_API_KEY=your_trends_key
```

### 3. Start the server

#### Option A: STDIO Transport (for Claude Desktop)

```sh
uv --directory /path/to/shodan-mcp run server.py
```

#### Option B: StreamableHTTP Transport (for web clients)

```sh
# Default port 8000
uv --directory /path/to/shodan-mcp run server.py --http

# Custom port
uv --directory /path/to/shodan-mcp run server.py --http --port 3000
```

### 4. Client Configuration

#### For Claude Desktop (STDIO)

Save the following in your Claude Desktop MCP settings:

```json
{
  "mcpServers": {
    "ShodanMCP": {
      "command": "uv",
      "args": [
        "--directory", "/path/to/shodan-mcp",
        "run", "server.py"
      ]
    }
  }
}
```

#### For HTTP Clients (StreamableHTTP)

Connect to the server at:
```
http://localhost:8000/sse
```

The server provides Server-Sent Events (SSE) for real-time communication.

## Transport Options

### STDIO Transport (Default)
- Best for: Claude Desktop, CLI tools, local development
- Connection: Standard input/output streams
- Configuration: See "For Claude Desktop (STDIO)" section above

### StreamableHTTP Transport
- Best for: Web-based clients, remote access, distributed systems
- Connection: HTTP with Server-Sent Events (SSE)
- Port: Configurable (default: 8000)
- Endpoint: `http://localhost:PORT/sse`
- Features:
  - Real-time bidirectional communication via SSE
  - Multiple concurrent client connections
  - Network-accessible from remote machines
  - Compatible with web browsers and HTTP clients

## Impact & Use Cases

- **Security Research:** Instantly query Shodan's global internet intelligence for threat hunting, asset discovery, and vulnerability research.
- **Automation:** Integrate Shodan tools into your security pipelines, SIEM, or custom dashboards via MCP.
- **Real-Time Monitoring:** Stream live banners and alerts for proactive monitoring of your infrastructure or the open internet.
- **Data Science:** Leverage Shodan's Trends API for analytics, reporting, and visualization of global internet trends.
- **Remote Access:** Use StreamableHTTP transport to access Shodan tools from web applications or remote clients.

## Credits

- **Shodan** — The world's leading search engine for Internet-connected devices.

---

© 2024 Haji & Contributors. This project is not affiliated with Shodan. For commercial use of Shodan data, ensure compliance with [Shodan's Terms of Service](https://www.shodan.io/terms).

# SSH MCP Server

**SSH MCP Server** is a local Model Context Protocol (MCP) server that exposes SSH control for Linux and Windows systems, enabling LLMs and other MCP clients to execute shell commands securely via SSH.

---

## Architecture Overview

This MCP server follows the Model Context Protocol specification, acting as a bridge between AI assistants and remote systems via SSH. Here's how it works:

```mermaid
sequenceDiagram
    participant User
    participant LLM as LLM/AI Assistant
    participant MCP Host as MCP Host<br/>(e.g., Claude Desktop)
    participant MCP Server as SSH MCP Server
    participant Remote as Remote SSH Host

    User->>LLM: "Check uptime on production server"
    LLM->>MCP Host: Request available tools
    MCP Host->>MCP Server: ListTools request
    MCP Server-->>MCP Host: Tool list with schemas
    MCP Host-->>LLM: Available tools
    
    LLM->>MCP Host: Call get_available_connections
    MCP Host->>User: ⚠️ Request approval:<br/>"Allow get_available_connections?"
    User->>MCP Host: ✅ Approve
    MCP Host->>MCP Server: Execute get_available_connections
    MCP Server-->>MCP Host: List of configured machines
    MCP Host-->>LLM: Machine list
    
    LLM->>MCP Host: Call create_connection(machine_id, title)
    MCP Host->>User: ⚠️ Request approval:<br/>"Allow create_connection to prod-01?"
    User->>MCP Host: ✅ Approve
    MCP Host->>MCP Server: Execute create_connection
    MCP Server->>Remote: SSH connect
    Remote-->>MCP Server: Connection established
    MCP Server-->>MCP Host: connection_id + current path
    MCP Host-->>LLM: Connection details
    
    LLM->>MCP Host: Call secure_execute_command(connection_id, "uptime")
    MCP Host->>User: ⚠️ Request approval:<br/>"Allow secure_execute_command: uptime?"
    User->>MCP Host: ✅ Approve
    MCP Host->>MCP Server: Execute command
    MCP Server->>Remote: Execute via SSH
    Remote-->>MCP Server: Command output
    MCP Server-->>MCP Host: stdout/stderr/exitCode
    MCP Host-->>LLM: Command results
    
    LLM->>User: "Server uptime is 45 days"
```

### Key Security Features

1. **User Approval Required**: The MCP host (e.g., Claude Desktop) asks for explicit user approval before executing each tool call
2. **Tool Visibility**: The LLM can see what tools are available and their schemas, but cannot execute them without user consent
3. **Secure Command Mode**: `secure_execute_command` blocks destructive operations like `rm -rf`, `chmod`, service management, etc.
4. **Session Management**: Connections are tracked and can be explicitly closed

---

## Features

* MCP-compliant server exposing SSH capabilities
* Execute shell commands on remote Linux and Windows systems
* Secure authentication via password or SSH key
* Read-only mode with built-in security checks
* Built with TypeScript and the official MCP SDK
* User approval workflow for all operations

## Available Tools

The server exposes 6 tools to the LLM. Each tool includes a detailed schema that the LLM can read to understand how to use it.

### 1. `get_available_connections`

**Description**: Lists every SSH-capable machine this server knows about (but is NOT yet connected).

**Input Schema**:
```json
{
  "type": "object",
  "properties": {},
  "additionalProperties": false
}
```

**Output**: JSON array of machines with `machine_id`, `label`, `os`, and `source` fields.

**Example**:
```json
[
  {
    "machine_id": "prod-server-01",
    "label": "Production Server",
    "os": "ubuntu",
    "source": "aws"
  }
]
```

---

### 2. `create_connection`

**Description**: Opens an SSH session to the given machine and tracks it in global state so subsequent tool calls can reuse it.

**Input Schema**:
```json
{
  "type": "object",
  "required": ["machine_id", "title"],
  "properties": {
    "machine_id": {
      "type": "string",
      "description": "ID from get_available_connections"
    },
    "title": {
      "type": "string",
      "description": "Purpose of this session (displayed in UIs)"
    }
  },
  "additionalProperties": false
}
```

**Output**: Connection details including `connection_id`, `machine_id`, `title`, and `currentPath`.

**Example Output**:
```json
{
  "connection_id": "550e8400-e29b-41d4-a716-446655440000",
  "machine_id": "prod-server-01",
  "title": "Check production logs",
  "currentPath": "/home/ubuntu"
}
```

---

### 3. `get_connections`

**Description**: Returns every STILL-OPEN SSH session in global state.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {},
  "additionalProperties": false
}
```

**Output**: Array of active connection details.

---

### 4. `execute_command`

**Description**: Runs a shell command in an existing SSH session and returns stdout/stderr/exitCode. **Unrestricted** - can execute any command.

**Input Schema**:
```json
{
  "type": "object",
  "required": ["connection_id", "command"],
  "properties": {
    "connection_id": {
      "type": "string"
    },
    "command": {
      "type": "string",
      "description": "Shell command to execute"
    }
  },
  "additionalProperties": false
}
```

**Output**: Command execution results.

**Example Output**:
```json
{
  "stdout": "15:30:45 up 45 days,  3:21,  1 user,  load average: 0.15, 0.10, 0.08\n",
  "stderr": "",
  "exitCode": 0
}
```

---

### 5. `secure_execute_command`

**Description**: Runs a **read-only** shell command in an existing SSH session. Blocks potentially destructive operations.

**Input Schema**:
```json
{
  "type": "object",
  "required": ["connection_id", "command"],
  "properties": {
    "connection_id": {
      "type": "string"
    },
    "command": {
      "type": "string",
      "description": "Read-only shell command to execute (e.g., ls, cat)"
    }
  },
  "additionalProperties": false
}
```

**Blocked Operations**:
- File deletion/modification: `rm`, `mv`, `dd`, `truncate`, file redirection to system locations
- Permission changes: `chmod`, `chown`
- Process management: `kill`, `pkill`, destructive `systemctl` operations
- Package management: `apt install`, `yum install`, etc.
- Network configuration: `iptables`, `ufw`
- User management: `useradd`, `passwd`, `sudo`
- Destructive git operations: `git push`, `git reset --hard`
- Container operations: `docker rm`, `kubectl delete`
- Text editors: `vim`, `nano`, `emacs`
- Background processes: commands ending with `&`, `nohup`

**Allowed Read-Only Operations**:
- File inspection: `ls`, `cat`, `head`, `tail`, `find`, `grep`
- System info: `uptime`, `df`, `free`, `ps`, `top`
- Read-only systemctl: `systemctl status`, `systemctl list-units`
- Read-only git: `git status`, `git log`, `git diff`
- Read-only package info: `apt list`, `yum info`
- Read-only container info: `docker ps`, `kubectl get`

**Output**: Same as `execute_command`.

---

### 6. `close_connection`

**Description**: Terminates an SSH session and removes it from global state.

**Input Schema**:
```json
{
  "type": "object",
  "required": ["connection_id"],
  "properties": {
    "connection_id": {
      "type": "string"
    }
  },
  "additionalProperties": false
}
```

**Output**:
```json
{
  "closed": true
}
```

---

## How the LLM Sees This Server

When an MCP host (like Claude Desktop) connects to this server, it:

1. **Discovers Tools**: Calls `ListTools` to get the list of 6 available tools with their complete schemas
2. **Reads Descriptions**: Each tool has a human-readable description that the LLM uses to understand its purpose
3. **Understands Parameters**: Input schemas tell the LLM what parameters are required, their types, and descriptions
4. **Plans Actions**: Based on user requests, the LLM plans which tools to call and in what order
5. **Requests Execution**: For each tool call, the LLM sends a request to the MCP host
6. **Awaits Approval**: The MCP host shows the user what the LLM wants to do and requests approval
7. **Receives Results**: After user approval, the tool executes and returns results to the LLM

**Important**: The LLM never executes tools directly. All execution goes through the MCP host, which implements the user approval workflow.

---

## Quick Start

### 1. Clone the repository

```bash
$ git clone https://github.com/vilasone455/ssh-mcp-server.git
```

### 2. Create machine config

Create a `machines.json` file with the following structure:

```json
[
   {
    "machine_id": "todo-server-01",
    "label": "Todo server",
    "os": "ubuntu",
    "source": "digitalocean",
    "ssh": {
      "host": "192.168.1.11",
      "port": 22,
      "username": "user",
      "password": "your_password_here"
    }
  },
  {
    "machine_id": "build-agent-01",
    "label": "CI Build Agent (Key Auth)",
    "os": "ubuntu",
    "source": "aws",
    "ssh": {
      "host": "192.168.1.12",
      "port": 22,
      "username": "ubuntu",
      "keyPath": "/home/ubuntu/.ssh/id_rsa"
    }
  }

]
```

---

## Client Setup (Claude Desktop Example)

To integrate this MCP server into Claude Desktop, add both the server command and the required environment variable:

```jsonc
{
  "mcpServers": {
    "ssh-mcp": {
      "command": "node",
      "args": [
        "/path/to/ssh-mcp-server/dist/index.js"
      ],
      "env": {
        "MACHINES_PATH": "/path/to/your/machines.json"
      }
    }
  }
}
```

Now you can interact with your server using natural language, e.g., "Run `uptime` on Todo VM."

---

## Disclaimer

Use at your own risk. This server grants shell-level access via MCP. Review commands carefully and run in a secure environment.

---

## Contributing

Star the repo, open issues, and submit pull requests! Feedback is welcome.

---

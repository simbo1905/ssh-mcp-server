# SSH MCP Server - Agent Capabilities Documentation

This document provides a comprehensive guide to the capabilities exposed by the SSH MCP Server for AI agents and LLMs.

---

## Overview

The SSH MCP Server is a **Model Context Protocol (MCP)** compliant server that bridges AI assistants with remote systems via SSH. It exposes a minimal, secure interface for managing SSH connections and executing commands on remote Linux and Windows systems.

### What is MCP?

Model Context Protocol (MCP) is an open protocol that standardizes how AI applications provide context to LLMs. Key aspects:

- **Server-Client Architecture**: MCP servers expose capabilities (tools, resources, prompts) that MCP clients (like Claude Desktop) can use
- **User Approval Workflow**: All tool executions require explicit user approval through the MCP host
- **Standardized Communication**: Uses JSON-RPC 2.0 over stdio, HTTP with SSE, or WebSocket transports
- **Discovery**: Clients can discover available tools, their schemas, and capabilities dynamically

### Architecture

```
┌─────────────────┐
│   User Input    │
│  "Check logs"   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  LLM/AI Assistant       │
│  (Planning & Reasoning) │
└────────┬────────────────┘
         │ Tool Request
         ▼
┌─────────────────────────┐
│   MCP Host              │
│   (Claude Desktop)      │
│   ┌─────────────────┐   │
│   │ User Approval   │◄──┼── User clicks "Allow"
│   │ Dialog          │   │
│   └─────────────────┘   │
└────────┬────────────────┘
         │ Approved Request
         ▼
┌─────────────────────────┐
│  SSH MCP Server         │
│  (This Application)     │
│  ┌─────────────────┐    │
│  │ Tool Handlers   │    │
│  └────────┬────────┘    │
└───────────┼─────────────┘
            │ SSH Protocol
            ▼
┌─────────────────────────┐
│  Remote System          │
│  (Linux/Windows)        │
└─────────────────────────┘
```

---

## Capability Categories

### 1. Connection Discovery & Management

These tools allow agents to discover available machines and manage SSH connections.

#### `get_available_connections`

**Purpose**: Enumerate all SSH-capable machines configured in the server.

**When to Use**:
- At the start of a session to discover available targets
- When a user asks "what servers can I access?"
- Before creating any new connection

**Input**: None required

**Output**: JSON array of machine objects with:
- `machine_id`: Unique identifier for the machine
- `label`: Human-readable name/description
- `os`: Operating system (e.g., "ubuntu", "windows")
- `source`: Infrastructure source (e.g., "aws", "digitalocean", "on-premise")

**Example Use Case**:
```
User: "Show me all available servers"
Agent: Calls get_available_connections
Server: Returns list of 5 configured servers
Agent: "You have access to 5 servers: Production DB (ubuntu/aws), ..."
```

**Security Considerations**:
- Only returns metadata; no credentials are exposed
- User approval required before execution
- Does not initiate any network connections

---

#### `create_connection`

**Purpose**: Establish an SSH connection to a remote machine and track it for subsequent operations.

**When to Use**:
- When you need to execute commands on a specific machine
- After calling `get_available_connections` to identify the target
- At the start of a task that requires remote access

**Input**:
- `machine_id` (required, string): The ID from `get_available_connections`
- `title` (required, string): A descriptive purpose for this connection (helps user understand what's happening)

**Output**: Connection object with:
- `connection_id`: UUID to use in subsequent command calls
- `machine_id`: The machine this connection is to
- `title`: The title provided
- `currentPath`: Current working directory on the remote system

**Example Use Case**:
```
User: "Check the logs on the production server"
Agent: 
  1. Calls get_available_connections
  2. Identifies "prod-server-01"
  3. Calls create_connection(machine_id="prod-server-01", title="Check production logs")
Server: Returns connection_id="abc-123-def"
Agent: Uses connection_id for subsequent commands
```

**Security Considerations**:
- Requires user approval showing the machine_id and title
- Creates a real SSH connection (network I/O)
- Connection persists until explicitly closed or server restart
- Each connection uses credentials configured in `machines.json`

---

#### `get_connections`

**Purpose**: List all currently active SSH sessions.

**When to Use**:
- To check if you already have a connection to a machine
- When debugging connection issues
- Before closing connections to see what's open

**Input**: None required

**Output**: Array of connection objects (same format as `create_connection` output)

**Example Use Case**:
```
Agent: Need to run command on prod-server-01
Agent: Calls get_connections first
Server: Returns 2 active connections, including one to prod-server-01
Agent: Reuses existing connection_id instead of creating a new one
```

**Security Considerations**:
- Only shows connections created in the current server session
- Does not expose credentials
- Useful for cleanup and resource management

---

#### `close_connection`

**Purpose**: Terminate an SSH connection and free resources.

**When to Use**:
- When finished with a remote system
- To implement good resource hygiene
- When switching between machines and want to minimize open connections

**Input**:
- `connection_id` (required, string): The UUID from `create_connection`

**Output**:
```json
{
  "closed": true
}
```

**Example Use Case**:
```
Agent: Finished checking logs
Agent: Calls close_connection(connection_id="abc-123-def")
Server: Closes SSH connection and removes from tracking
```

**Security Considerations**:
- Requires user approval
- Cleanly terminates SSH session
- Connection cannot be used after closing

---

### 2. Command Execution

These tools allow agents to execute shell commands on remote systems via SSH.

#### `execute_command`

**Purpose**: Execute any shell command on a remote system. **UNRESTRICTED**.

**When to Use**:
- When you need to perform administrative tasks
- When `secure_execute_command` blocks a legitimate operation
- When you understand the risks and have user approval

**Input**:
- `connection_id` (required, string): Active connection UUID
- `command` (required, string): Shell command to execute

**Output**:
```json
{
  "stdout": "command output...",
  "stderr": "any errors...",
  "exitCode": 0
}
```

**Special Behavior**:
- Detects `cd` commands and updates the connection's `currentPath`
- Commands execute in the shell environment of the SSH session
- Exit code 0 typically indicates success

**Example Use Cases**:

1. **System Administration**:
```
User: "Restart the nginx service"
Agent: execute_command(connection_id="...", command="sudo systemctl restart nginx")
```

2. **Log Analysis**:
```
User: "Show me the last 100 lines of the application log"
Agent: execute_command(connection_id="...", command="tail -n 100 /var/log/app.log")
```

3. **File Management**:
```
User: "Delete the old backup files"
Agent: execute_command(connection_id="...", command="rm /backups/old-*.tar.gz")
```

**Security Considerations**:
- ⚠️ **NO RESTRICTIONS** - can execute destructive commands
- User sees the EXACT command in the approval dialog
- Commands run with the privileges of the SSH user
- User is responsible for reviewing and approving each command
- Best practice: Use `secure_execute_command` when possible

---

#### `secure_execute_command`

**Purpose**: Execute **read-only** shell commands with built-in safety checks. **RECOMMENDED** for routine operations.

**When to Use**:
- For inspection and information gathering
- When you don't need to modify system state
- As the default choice for command execution

**Input**:
- `connection_id` (required, string): Active connection UUID
- `command` (required, string): Read-only shell command

**Output**: Same format as `execute_command`

**Blocked Command Categories**:

1. **File Destruction**:
   - `rm -rf`, `dd`, `truncate`
   - Output redirection to system paths (`> /etc/...`)
   - Archive extraction that could overwrite files

2. **Permission Changes**:
   - `chmod`, `chown`

3. **Process Management**:
   - `kill`, `pkill`, `killall`
   - `systemctl start/stop/restart/enable/disable`
   - `service start/stop/restart`

4. **Package Management**:
   - `apt install/remove/update/upgrade`
   - `yum install/remove/update`
   - Similar for other package managers

5. **Network Configuration**:
   - `iptables`, `ufw`, `firewall-cmd`
   - `ifconfig up/down`

6. **User/System Management**:
   - `useradd`, `userdel`, `usermod`, `passwd`
   - `su`, `sudo` (when used for execution)
   - `crontab -e/-r`

7. **Version Control**:
   - `git push`, `git pull`, `git reset --hard`, `git clean -f`

8. **Containers & Orchestration**:
   - `docker rm/rmi/kill/stop/run/build/push`
   - `kubectl delete/apply/create/replace/patch`

9. **File Editing**:
   - `nano`, `vi`, `vim`, `emacs`, `code`

10. **Background Processes**:
    - Commands ending with `&`
    - `nohup`
    - Pipes to interpreters (`| sh`, `| python`)

**Explicitly Allowed Operations**:

1. **File Inspection**:
   - `ls`, `cat`, `head`, `tail`, `less`, `more`
   - `find`, `grep`, `awk`, `sed` (read-only)
   - `file`, `stat`, `du`, `df`

2. **System Information**:
   - `uptime`, `hostname`, `uname`
   - `free`, `top`, `htop`, `ps`, `pstree`
   - `w`, `who`, `whoami`, `id`

3. **Network Inspection**:
   - `ping`, `traceroute`, `netstat`, `ss`
   - `curl`, `wget` (without destructive flags)
   - `nslookup`, `dig`, `host`

4. **Read-Only Service Info**:
   - `systemctl status`, `systemctl list-units`, `systemctl show`
   - `service status`

5. **Read-Only Git**:
   - `git status`, `git log`, `git show`, `git diff`
   - `git branch`, `git remote`, `git config --list`

6. **Read-Only Package Info**:
   - `apt list`, `apt search`, `apt show`
   - `yum list`, `yum info`

7. **Read-Only Container Info**:
   - `docker ps`, `docker images`, `docker inspect`, `docker logs`
   - `kubectl get`, `kubectl describe`, `kubectl logs`

**Example Use Cases**:

1. **Safe System Check**:
```
User: "Check system resources"
Agent: secure_execute_command(connection_id="...", command="df -h && free -m")
✅ Allowed - read-only system info
```

2. **Safe Log Viewing**:
```
Agent: secure_execute_command(connection_id="...", command="tail -f /var/log/syslog")
✅ Allowed - reading files
```

3. **Blocked Destructive Operation**:
```
Agent: secure_execute_command(connection_id="...", command="rm -rf /tmp/old-*")
❌ Blocked - Server returns error: "Command contains potentially dangerous operations"
```

**Security Considerations**:
- Provides a safety net for AI agents
- Command validation happens before execution
- Blocks even legitimate-looking but dangerous commands
- User still sees and approves the command
- If blocked, agent should ask user if they want to use `execute_command` instead

---

## Typical Agent Workflows

### Workflow 1: Information Gathering (Read-Only)

```
User Request: "What's the disk usage on the production server?"

1. get_available_connections
   → Returns list of machines
   → Agent identifies "prod-server-01"

2. create_connection(machine_id="prod-server-01", title="Check disk usage")
   → Returns connection_id

3. secure_execute_command(connection_id, command="df -h")
   → Returns disk usage output

4. Agent interprets output for user: "Production server is 65% full (32GB used of 50GB)"

5. close_connection(connection_id)
   → Cleans up
```

### Workflow 2: Administrative Task (Write Operation)

```
User Request: "Restart the web service on server-02"

1. get_available_connections
   → Identify "server-02"

2. create_connection(machine_id="server-02", title="Restart web service")
   → Returns connection_id

3. secure_execute_command(connection_id, command="systemctl status nginx")
   → ❌ Blocked (not actually, status is allowed!)
   → ✅ Returns current status

4. execute_command(connection_id, command="sudo systemctl restart nginx")
   → User sees EXACT command and approves
   → ✅ Service restarted

5. secure_execute_command(connection_id, command="systemctl status nginx")
   → ✅ Confirm service is running

6. close_connection(connection_id)
```

### Workflow 3: Multi-Machine Task

```
User Request: "Check which servers have the file /etc/config.yml"

1. get_available_connections
   → Returns 5 servers

2. For each server:
   a. create_connection(machine_id, title="Check for config file")
   b. secure_execute_command(connection_id, command="ls -la /etc/config.yml")
   c. Parse output (success vs "No such file")
   d. close_connection(connection_id)

3. Agent summarizes: "3 out of 5 servers have the config file: server-01, server-03, server-05"
```

---

## Best Practices for AI Agents

### 1. Connection Management

- **Reuse connections**: Call `get_connections` before creating a new connection
- **Clean up**: Always call `close_connection` when done
- **Descriptive titles**: Use clear titles so users understand the purpose

### 2. Command Execution

- **Prefer secure mode**: Use `secure_execute_command` as the default
- **Explain commands**: In user-facing messages, explain what commands will do
- **Check exit codes**: Non-zero exit codes usually indicate errors
- **Parse output**: Interpret stdout/stderr meaningfully for users

### 3. Error Handling

- **Connection failures**: Inform user if `create_connection` fails (e.g., wrong credentials)
- **Command failures**: Check `exitCode` and `stderr` to detect issues
- **Blocked commands**: If `secure_execute_command` blocks something, explain why and offer alternatives

### 4. Security Awareness

- **Be explicit**: When using `execute_command`, clearly tell the user what the command does
- **Avoid blind execution**: Don't execute commands you don't understand
- **Validate inputs**: If a user provides a command directly, explain what it does before executing

### 5. User Experience

- **Progress updates**: For long-running tasks, explain what's happening
- **Summarize results**: Don't just dump command output; interpret it
- **Ask for confirmation**: For destructive operations, double-check with the user

---

## Configuration Reference

### Machine Configuration File

The server reads available machines from a JSON file specified by the `MACHINES_PATH` environment variable.

**Structure**:
```json
[
  {
    "machine_id": "unique-identifier",
    "label": "Human-readable name",
    "os": "ubuntu|windows|centos|etc",
    "source": "aws|digitalocean|on-premise|etc",
    "ssh": {
      "host": "hostname or IP",
      "port": 22,
      "username": "ssh-user",
      "password": "plain-text-password",
      "keyPath": "/path/to/private/key"
    }
  }
]
```

**Authentication Methods**:
- **Password**: Include `password` field
- **SSH Key**: Include `keyPath` field
- Use one or the other, not both

**Example**:
```json
[
  {
    "machine_id": "web-prod-01",
    "label": "Production Web Server",
    "os": "ubuntu",
    "source": "aws",
    "ssh": {
      "host": "10.0.1.50",
      "port": 22,
      "username": "ubuntu",
      "keyPath": "/home/user/.ssh/prod-key"
    }
  },
  {
    "machine_id": "db-test-01",
    "label": "Test Database",
    "os": "ubuntu",
    "source": "digitalocean",
    "ssh": {
      "host": "test-db.example.com",
      "port": 22,
      "username": "dbadmin",
      "password": "SecurePassword123!"
    }
  }
]
```

---

## Protocol Details

### Transport

- **Type**: stdio (standard input/output)
- **Format**: JSON-RPC 2.0
- **Bidirectional**: Server can send notifications; client sends requests

### Message Flow

1. **Client → Server**: `initialize` request
2. **Server → Client**: Server capabilities (tools)
3. **Client → Server**: `tools/list` request
4. **Server → Client**: Array of tool schemas
5. **Client → Server**: `tools/call` request with tool name and arguments
6. **Server → Client**: Tool result or error

### Tool Result Format

All tools return content in this format:
```json
{
  "content": [
    {
      "type": "text",
      "text": "<JSON string or plain text>"
    }
  ]
}
```

The `text` field typically contains a JSON-stringified result that agents should parse.

---

## Limitations & Known Issues

### Current Limitations

1. **No File Transfer**: Cannot upload/download files directly (must use command-line tools like `cat`, `echo`, etc.)
2. **No Interactive Commands**: Commands that require stdin interaction won't work well
3. **No Streaming Output**: Command output is returned after completion, not streamed
4. **Session Persistence**: Connections are lost on server restart
5. **Single User**: No multi-user or multi-tenant support
6. **No Command History**: Previous commands are not tracked

### Security Considerations

1. **Credentials in Config**: The `machines.json` file contains sensitive credentials
   - Store in a secure location
   - Restrict file permissions (chmod 600)
   - Consider using SSH keys instead of passwords

2. **User Responsibility**: Even with `secure_execute_command`, the user must:
   - Review each command before approval
   - Understand the implications of commands
   - Trust the AI agent's judgment

3. **Network Security**: 
   - SSH connections are encrypted
   - Server runs locally (no external exposure by default)
   - MCP communication over stdio (local only)

4. **Privilege Escalation**:
   - Commands run with the SSH user's permissions
   - If SSH user has sudo access, agent could execute privileged commands
   - Consider using restricted SSH users for read-only operations

---

## Troubleshooting

### "Connection refused" errors

- Check that SSH is running on the remote host
- Verify the host and port in `machines.json`
- Test connection manually: `ssh user@host -p port`

### "Authentication failed" errors

- Verify username/password or key path
- Check key permissions: `chmod 600 ~/.ssh/key`
- Ensure SSH key is not password-protected (or add passphrase support)

### "Command not found" errors

- Command may not be in the remote user's PATH
- Try with full path: `/usr/bin/command` instead of `command`
- Check if the command is installed: `which command`

### Blocked commands in `secure_execute_command`

- Review the command against the blocklist
- If legitimate, use `execute_command` instead
- Consider requesting a refinement to the security rules

---

## Future Enhancements

Potential capabilities that could be added:

1. **File Resources**: Expose remote files as MCP resources
2. **Streaming Output**: Real-time command output
3. **Interactive Mode**: Support for commands requiring user input
4. **File Transfer**: Native scp/sftp support
5. **Session Persistence**: Reconnect to existing sessions
6. **Multi-User**: User-specific connections and permissions
7. **Audit Logging**: Track all executed commands
8. **Custom Security Policies**: User-defined command filters
9. **Terminal Access**: Full interactive terminal via MCP

---

## Developer Notes

### Adding New Tools

To add a new tool to the server:

1. **Define the tool schema** in the `ListToolsRequestSchema` handler:
```typescript
{
  name: "my_new_tool",
  description: "What it does",
  inputSchema: {
    type: "object",
    required: ["param1"],
    properties: {
      param1: { type: "string", description: "..." }
    }
  }
}
```

2. **Implement the handler** in the `CallToolRequestSchema` handler:
```typescript
if (name === "my_new_tool") {
  const { param1 } = args as { param1: string };
  // Implementation
  return {
    content: [{ type: "text", text: JSON.stringify(result) }]
  };
}
```

3. **Update documentation** in README.md and this file

### Testing Tools

Since there's no automated test suite, test tools manually:

1. Configure a test machine in `machines.json`
2. Run the server: `node dist/index.js`
3. Use an MCP client (like Claude Desktop) to call tools
4. Verify outputs and error handling

---

## Conclusion

The SSH MCP Server provides a minimal, secure interface for AI agents to interact with remote systems. By combining the power of SSH with the safety of user approval workflows and built-in security checks, it enables practical automation while maintaining control and security.

For agents: Use this server to help users manage remote systems, gather information, and perform administrative tasks with appropriate safeguards.

For users: Review and approve each operation, understanding that you have final control over what commands are executed.

For developers: Extend this server carefully, always prioritizing security and the principle of least privilege.

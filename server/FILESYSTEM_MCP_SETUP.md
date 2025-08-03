# Filesystem MCP Server Integration for AnythingLLM

This document explains how to set up and use the Model Context Protocol (MCP) Filesystem Server with AnythingLLM.

## Overview

The MCP Filesystem Server provides AI agents with secure file system access capabilities, allowing them to read, write, create, and manage files and directories within specified boundaries.

## Features

The filesystem MCP server provides the following tools:
- `read_text_file`: Read text file contents
- `read_media_file`: Read image/audio files
- `write_file`: Create or overwrite files
- `edit_file`: Advanced file editing capabilities
- `create_directory`: Create new directories
- `list_directory`: List directory contents
- `move_file`: Move/rename files and directories
- `search_files`: Search for files by name or content
- `get_file_info`: Get file metadata and information

## Security Considerations

**⚠️ Important Security Notes:**
- The filesystem server only has access to directories you explicitly specify
- Access is sandboxed to prevent unauthorized file system access
- Use absolute paths to ensure precise directory control
- Avoid granting access to sensitive system directories

## Configuration

### Basic Setup

Add the filesystem server to your `storage/plugins/anythingllm_mcp_servers.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/directory"],
      "env": {
        "NODE_ENV": "production"
      },
      "autoStart": true
    }
  }
}
```

### Multiple Directory Access

To grant access to multiple directories:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", 
        "@modelcontextprotocol/server-filesystem", 
        "/home/user/documents",
        "/home/user/projects",
        "/tmp"
      ],
      "env": {
        "NODE_ENV": "production"
      },
      "autoStart": true
    }
  }
}
```

### Production Configuration Examples

#### For Document Management
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", 
        "@modelcontextprotocol/server-filesystem", 
        "/var/lib/anythingllm/storage/documents",
        "/var/lib/anythingllm/storage/custom-documents"
      ],
      "env": {
        "NODE_ENV": "production"
      },
      "autoStart": true
    }
  }
}
```

#### For Development Environment
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", 
        "@modelcontextprotocol/server-filesystem", 
        "./storage/documents",
        "./workspace",
        "/tmp"
      ],
      "env": {
        "NODE_ENV": "development"
      },
      "autoStart": true
    }
  }
}
```

## Configuration Options

| Parameter | Description | Example |
|-----------|-------------|---------|
| `command` | Command to execute | `"npx"` |
| `args` | Arguments array including server and directories | `["-y", "@modelcontextprotocol/server-filesystem", "/path"]` |
| `env` | Environment variables | `{"NODE_ENV": "production"}` |
| `autoStart` | Whether to start automatically | `true` or `false` |

## Agent Integration

Once configured, the filesystem tools become available to agents through the `@@mcp_filesystem` directive:

### In Agent Flows
```markdown
# Agent with filesystem access
You are a file management assistant with access to the filesystem.

Available tools via @@mcp_filesystem:
- Read and write files
- Create and manage directories  
- Search for files
- Get file information

Always confirm before making destructive changes to files.
```

### Available MCP Tools in Agents

After configuration, agents will have access to these tools:

1. **filesystem-read_text_file**
   - Read contents of text files
   - Supports various text formats

2. **filesystem-write_file**
   - Create new files or overwrite existing ones
   - Specify content and file path

3. **filesystem-list_directory**
   - List contents of directories
   - Shows files and subdirectories

4. **filesystem-create_directory**
   - Create new directories
   - Supports nested directory creation

5. **filesystem-move_file**
   - Move or rename files and directories
   - Handles both files and folders

6. **filesystem-search_files**
   - Search for files by name patterns
   - Support for recursive searching

7. **filesystem-get_file_info**
   - Get metadata about files
   - Shows size, permissions, timestamps

## Usage Examples

### Basic File Operations

**Reading a file:**
```
Agent: I need to read the contents of config.json
[Uses filesystem-read_text_file with path: "/allowed/directory/config.json"]
```

**Writing a file:**
```
Agent: I'll create a new README file for this project
[Uses filesystem-write_file with content and path]
```

**Listing directories:**
```
Agent: Let me see what files are in the documents folder
[Uses filesystem-list_directory with path: "/allowed/directory/documents"]
```

### Advanced Operations

**Searching for files:**
```
Agent: I need to find all Python files in the project
[Uses filesystem-search_files with pattern: "*.py"]
```

**Creating project structure:**
```
Agent: I'll set up the project directories
[Uses filesystem-create_directory to create multiple folders]
```

## Troubleshooting

### Common Issues

1. **Permission Denied Errors**
   - Ensure AnythingLLM process has read/write access to specified directories
   - Check directory permissions: `ls -la /path/to/directory`
   - Verify paths are absolute and correct

2. **Server Not Starting**
   - Check if `npx` is available in the system PATH
   - Verify network connectivity for downloading the server package
   - Review AnythingLLM logs for error messages

3. **Directory Access Issues**
   - Confirm directories exist before adding to configuration
   - Use absolute paths to avoid confusion
   - Test directory access manually

### Debug Commands

```bash
# Test filesystem server manually
npx @modelcontextprotocol/server-filesystem /tmp

# Check if server package is available
npm info @modelcontextprotocol/server-filesystem

# Verify directory permissions
stat /path/to/your/directory
```

### Logs and Monitoring

Monitor MCP server status through AnythingLLM admin interface:
1. Go to Admin Settings → MCP Servers
2. Check server status and logs
3. Use "Force Reload" to restart servers

## Best Practices

### Security
- Only grant access to directories that agents actually need
- Use specific paths rather than broad system access
- Regularly review and audit file access patterns
- Consider using read-only access for sensitive directories

### Performance
- Limit the number of accessible directories
- Use `autoStart: false` for servers used infrequently
- Monitor disk usage in accessible directories

### Organization
- Group related directories logically
- Use descriptive server names for multiple filesystem servers
- Document the purpose of each accessible directory

## Environment Variables

You can use environment variables in your configuration:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "${ANYTHINGLLM_STORAGE_DIR}/documents"],
      "env": {
        "NODE_ENV": "${NODE_ENV}",
        "ANYTHINGLLM_STORAGE_DIR": "/var/lib/anythingllm/storage"
      },
      "autoStart": true
    }
  }
}
```

## Advanced Configuration

### Multiple Filesystem Servers

You can configure multiple filesystem servers for different purposes:

```json
{
  "mcpServers": {
    "filesystem-documents": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/documents"],
      "autoStart": true
    },
    "filesystem-temp": {
      "command": "npx", 
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "autoStart": false
    }
  }
}
```

### Docker Deployment

For Docker environments, ensure volumes are properly mounted:

```yaml
# docker-compose.yml
version: '3.8'
services:
  anythingllm:
    image: anythingllm:latest
    volumes:
      - ./storage:/app/server/storage
      - ./workspace:/workspace:rw
    environment:
      - STORAGE_DIR=/app/server/storage
```

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/app/server/storage/documents", "/workspace"],
      "autoStart": true
    }
  }
}
```

## Support and Resources

- [MCP Filesystem Server GitHub](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem)
- [AnythingLLM MCP Documentation](https://docs.anythingllm.com/mcp-compatibility/overview)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)

## License

The MCP Filesystem Server is licensed under the MIT License. See the [original repository](https://github.com/modelcontextprotocol/servers) for details.
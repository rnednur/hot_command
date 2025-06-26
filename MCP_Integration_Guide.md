# MCP Integration Guide for Hot Commands

## Overview

This guide explains how to implement your hot commands functionality as an MCP (Model Context Protocol) server, making it compatible with any MCP-enabled AI assistant like Claude, GPT, or custom AI systems.

## Why MCP for Hot Commands?

### Benefits
- **Future-Proof**: Works with any MCP-compatible AI assistant
- **Reusable**: Other teams/projects can leverage your service
- **Standardized**: Follows modern AI tool integration patterns
- **Maintainable**: Clean separation between AI interface and business logic
- **Scalable**: Can serve multiple AI assistants simultaneously

### Use Cases
- Claude can directly create and execute your hot commands
- External applications can integrate with your hot commands
- Multiple AI assistants can share the same command library
- Teams can build on top of your hot commands infrastructure

## Implementation Strategy

### Phase 1: MCP-Ready Core Service (Weeks 1-15)
Build your hot commands service with MCP compatibility in mind, but start with internal APIs.

### Phase 2: MCP Server Wrapper (Weeks 16-18)
Add the MCP protocol layer on top of your existing service.

## Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   AI Assistant  │◄──►│ MCP Hot Commands │◄──►│ Hot Commands    │
│ (Claude/GPT/etc)│    │     Server       │    │ Core Service    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │
                                ▼
                        ┌──────────────────┐
                        │   PostgreSQL +   │
                        │   Redis + Cloud  │
                        └──────────────────┘
```

## MCP Server Implementation

### Core Server Class

```python
# mcp_server/hot_commands_server.py
from mcp import Server
from mcp.errors import McpError
from typing import List, Dict, Any
import asyncio
from datetime import datetime

from services.hot_command_crud import HotCommandCRUD
from services.execution_engine import HotCommandExecutionEngine
from services.spaces_manager import SpacesManager

class HotCommandsMCPServer:
    def __init__(self):
        self.server = Server("hotcommands")
        self.crud_service = HotCommandCRUD()
        self.execution_engine = HotCommandExecutionEngine()
        self.spaces_manager = SpacesManager()
        
        # Register tools and resources
        self._register_tools()
        self._register_resources()
    
    def _register_tools(self):
        """Register all MCP tools"""
        
        @self.server.tool()
        async def list_hotcommands(
            user_id: str,
            domain: str = None,
            category: str = None,
            search: str = None,
            include_shared: bool = True
        ) -> Dict[str, Any]:
            """
            List user's hot commands with optional filtering.
            
            Args:
                user_id: User identifier
                domain: Filter by domain (e.g., 'RAN', 'CustomerExperience')
                category: Filter by category (e.g., 'Capacity', 'Mobility')
                search: Search query for command names/descriptions
                include_shared: Whether to include commands shared with user
                
            Returns:
                Dict containing list of commands and metadata
            """
            try:
                commands = await self.crud_service.list_commands(
                    user_id=user_id,
                    domain=domain,
                    category=category,
                    search_query=search,
                    include_shared=include_shared
                )
                
                return {
                    "commands": [
                        {
                            "name": cmd.command_name,
                            "display_name": cmd.display_name,
                            "description": cmd.description,
                            "query_type": cmd.query_type.value,
                            "domain": cmd.domain,
                            "category": cmd.category,
                            "parameters": {name: param.to_dict() for name, param in cmd.parameters.items()},
                            "usage_count": cmd.usage_count,
                            "last_used": cmd.last_used_at.isoformat() if cmd.last_used_at else None,
                            "is_shared": cmd.is_shared
                        }
                        for cmd in commands
                    ],
                    "total": len(commands),
                    "filtered_by": {
                        "domain": domain,
                        "category": category,
                        "search": search
                    }
                }
            except Exception as e:
                raise McpError(f"Failed to list hot commands: {str(e)}", code=-32001)
        
        @self.server.tool()
        async def create_hotcommand(
            user_id: str,
            command_name: str,
            query_text: str,
            display_name: str = None,
            description: str = "",
            query_type: str = "nl2sql",
            domain: str = None,
            category: str = None,
            parameters: Dict[str, Any] = None
        ) -> Dict[str, Any]:
            """
            Create a new hot command.
            
            Args:
                user_id: User identifier
                command_name: Unique command name (e.g., 'high_util_cells')
                query_text: The query text (natural language or SQL)
                display_name: User-friendly display name
                description: Command description
                query_type: Type of query ('nl2sql', 'direct_sql', 'tool_call')
                domain: Domain context (e.g., 'RAN')
                category: Category context (e.g., 'Capacity')
                parameters: Parameter definitions for dynamic queries
                
            Returns:
                Dict containing created command details
            """
            try:
                from models.hot_command import QueryType, Parameter, ParameterType
                
                # Convert parameters if provided
                parsed_parameters = {}
                if parameters:
                    for name, param_def in parameters.items():
                        parsed_parameters[name] = Parameter(
                            name=name,
                            type=ParameterType(param_def.get('type', 'string')),
                            required=param_def.get('required', False),
                            default=param_def.get('default'),
                            description=param_def.get('description', ''),
                            options=param_def.get('options'),
                            validation_regex=param_def.get('validation_regex')
                        )
                
                command = await self.crud_service.create_command(
                    user_id=user_id,
                    command_name=command_name,
                    query_text=query_text,
                    query_type=QueryType(query_type),
                    display_name=display_name,
                    description=description,
                    domain=domain,
                    category=category,
                    parameters=parsed_parameters
                )
                
                return {
                    "success": True,
                    "command": {
                        "id": command.id,
                        "name": command.command_name,
                        "display_name": command.display_name,
                        "description": command.description,
                        "query_text": command.query_text,
                        "created_at": command.created_at.isoformat()
                    },
                    "message": f"Hot command '{command_name}' created successfully"
                }
            except Exception as e:
                raise McpError(f"Failed to create hot command: {str(e)}", code=-32002)
        
        @self.server.tool()
        async def execute_hotcommand(
            user_id: str,
            command_name: str,
            parameters: Dict[str, Any] = None,
            output_format: str = "table"
        ) -> Dict[str, Any]:
            """
            Execute a hot command with optional parameters.
            
            Args:
                user_id: User identifier
                command_name: Name of command to execute
                parameters: Runtime parameters for the command
                output_format: Desired output format ('table', 'json', 'csv')
                
            Returns:
                Dict containing execution results
            """
            try:
                # Get command
                command = await self.crud_service.get_command(user_id, command_name)
                if not command:
                    raise McpError(f"Hot command '{command_name}' not found", code=-32003)
                
                # Execute command
                result = await self.execution_engine.execute_command(
                    command=command,
                    user_id=user_id,
                    provided_parameters=parameters or {},
                    execution_context={"output_format": output_format}
                )
                
                if result.success:
                    return {
                        "success": True,
                        "data": result.data,
                        "result_type": result.result_type,
                        "execution_time_ms": result.execution_time_ms,
                        "message": f"Command '{command_name}' executed successfully"
                    }
                else:
                    raise McpError(f"Command execution failed: {result.error}", code=-32004)
                    
            except McpError:
                raise
            except Exception as e:
                raise McpError(f"Failed to execute hot command: {str(e)}", code=-32004)
        
        @self.server.tool()
        async def update_hotcommand(
            user_id: str,
            command_name: str,
            query_text: str = None,
            display_name: str = None,
            description: str = None,
            parameters: Dict[str, Any] = None
        ) -> Dict[str, Any]:
            """
            Update an existing hot command.
            """
            try:
                updates = {}
                if query_text is not None:
                    updates['query_text'] = query_text
                if display_name is not None:
                    updates['display_name'] = display_name
                if description is not None:
                    updates['description'] = description
                if parameters is not None:
                    updates['parameters'] = parameters
                
                command = await self.crud_service.update_command(
                    user_id=user_id,
                    command_name=command_name,
                    **updates
                )
                
                return {
                    "success": True,
                    "command": command.to_dict(),
                    "message": f"Hot command '{command_name}' updated successfully"
                }
            except Exception as e:
                raise McpError(f"Failed to update hot command: {str(e)}", code=-32005)
        
        @self.server.tool()
        async def delete_hotcommand(
            user_id: str,
            command_name: str
        ) -> Dict[str, Any]:
            """
            Delete a hot command.
            """
            try:
                success = await self.crud_service.delete_command(user_id, command_name)
                
                if success:
                    return {
                        "success": True,
                        "message": f"Hot command '{command_name}' deleted successfully"
                    }
                else:
                    raise McpError(f"Hot command '{command_name}' not found", code=-32003)
                    
            except McpError:
                raise
            except Exception as e:
                raise McpError(f"Failed to delete hot command: {str(e)}", code=-32006)
        
        @self.server.tool()
        async def save_to_space(
            user_id: str,
            space_name: str,
            content: Any,
            content_type: str = "table",
            description: str = "",
            retention_days: int = 30
        ) -> Dict[str, Any]:
            """
            Save content to a space for later retrieval or sharing.
            """
            try:
                from models.space import ContentType
                
                space = await self.spaces_manager.save_space(
                    user_id=user_id,
                    space_name=space_name,
                    content=content,
                    content_type=ContentType(content_type),
                    description=description,
                    retention_days=retention_days
                )
                
                return {
                    "success": True,
                    "space": space.to_dict(),
                    "message": f"Content saved to space '{space_name}'"
                }
            except Exception as e:
                raise McpError(f"Failed to save to space: {str(e)}", code=-32007)
    
    def _register_resources(self):
        """Register MCP resources for hot command inspection"""
        
        @self.server.resource("hotcommand://{user_id}/{command_name}")
        async def get_hotcommand_resource(user_id: str, command_name: str):
            """
            Get detailed information about a specific hot command.
            Allows AI assistants to inspect command definitions.
            """
            try:
                command = await self.crud_service.get_command(user_id, command_name)
                if not command:
                    raise McpError(f"Hot command '{command_name}' not found", code=-32003)
                
                content = f"""Hot Command: {command.display_name}
Command Name: {command.command_name}
Description: {command.description}
Query Type: {command.query_type.value}
Domain: {command.domain or 'N/A'}
Category: {command.category or 'N/A'}
Usage Count: {command.usage_count}
Last Used: {command.last_used_at.isoformat() if command.last_used_at else 'Never'}

Query Text:
{command.query_text}

Parameters:
"""
                if command.parameters:
                    for name, param in command.parameters.items():
                        content += f"  - {name} ({param.type.value})"
                        if param.required:
                            content += " [REQUIRED]"
                        if param.default:
                            content += f" [DEFAULT: {param.default}]"
                        if param.description:
                            content += f" - {param.description}"
                        content += "\n"
                else:
                    content += "  No parameters\n"
                
                return {
                    "contents": [{
                        "type": "text/plain",
                        "text": content
                    }]
                }
            except Exception as e:
                raise McpError(f"Failed to get hot command resource: {str(e)}", code=-32008)
        
        @self.server.resource("hotcommands://{user_id}")
        async def list_hotcommands_resource(user_id: str):
            """
            Get a summary of all hot commands for a user.
            Useful for AI assistants to understand available commands.
            """
            try:
                commands = await self.crud_service.list_commands(user_id)
                
                content = f"Hot Commands for User: {user_id}\n"
                content += f"Total Commands: {len(commands)}\n\n"
                
                if commands:
                    for cmd in commands:
                        content += f"/{cmd.command_name}"
                        if cmd.display_name != cmd.command_name:
                            content += f" ({cmd.display_name})"
                        content += f" - {cmd.description or 'No description'}\n"
                        if cmd.parameters:
                            content += f"  Parameters: {', '.join(cmd.parameters.keys())}\n"
                        content += f"  Domain: {cmd.domain or 'N/A'}, Category: {cmd.category or 'N/A'}\n"
                        content += f"  Used {cmd.usage_count} times\n\n"
                else:
                    content += "No hot commands found.\n"
                
                return {
                    "contents": [{
                        "type": "text/plain", 
                        "text": content
                    }]
                }
            except Exception as e:
                raise McpError(f"Failed to list hot commands resource: {str(e)}", code=-32009)

# MCP Server Entry Point
async def main():
    server = HotCommandsMCPServer()
    
    # Start the MCP server
    from mcp.server.stdio import serve_stdio
    await serve_stdio(server.server)

if __name__ == "__main__":
    asyncio.run(main())
```

## Configuration

### MCP Server Configuration

```yaml
# mcp_config.yaml
name: "hotcommands"
version: "1.0.0"
description: "Hot Commands MCP Server for NL2SQL Framework"

server:
  host: "localhost"
  port: 8000
  
tools:
  - name: "list_hotcommands"
    description: "List user's hot commands with filtering"
  - name: "create_hotcommand" 
    description: "Create a new hot command"
  - name: "execute_hotcommand"
    description: "Execute a hot command with parameters"
  - name: "update_hotcommand"
    description: "Update an existing hot command"
  - name: "delete_hotcommand"
    description: "Delete a hot command"
  - name: "save_to_space"
    description: "Save content to a persistent space"

resources:
  - pattern: "hotcommand://{user_id}/{command_name}"
    description: "Individual hot command details"
  - pattern: "hotcommands://{user_id}" 
    description: "All hot commands for a user"

authentication:
  type: "api_key"
  header: "X-API-Key"
```

### Docker Configuration

```dockerfile
# Dockerfile for MCP Server
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "mcp_server/hot_commands_server.py"]
```

```yaml
# docker-compose.yml (updated)
version: '3.8'
services:
  hotcommands-mcp:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/hotcommands
      - REDIS_URL=redis://redis:6379
      - MCP_SERVER_NAME=hotcommands
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: hotcommands
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## Usage Examples

### From Claude/AI Assistant

```
User: "Show me my hot commands for the RAN domain"
Assistant: *calls list_hotcommands tool with domain="RAN"*

User: "Create a hot command called 'cell_health' that shows cells with poor signal"
Assistant: *calls create_hotcommand tool with appropriate parameters*

User: "Execute my 'top_sales' command for the last week"
Assistant: *calls execute_hotcommand with parameters={"period": "week"}*
```

### From External Applications

```python
# Other applications can use your MCP server
import mcp_client

client = mcp_client.Client("http://localhost:8000")
commands = await client.call_tool("list_hotcommands", {"user_id": "user123"})
```

### Resource Inspection

AIs can inspect command details using resources:

```python
# AI assistant can read command details
resource = await client.read_resource("hotcommand://user123/my_sales_cmd")
# Returns detailed command information for better understanding
```

## Integration Benefits

### 1. Multi-Assistant Support
- Claude can directly create and execute your hot commands
- GPT-4 with MCP support can use the same functionality
- Custom AI assistants can integrate seamlessly

### 2. Standardized Protocol
- Automatic error handling with proper MCP error codes
- Progress tracking for long-running operations
- Resource discovery for AI understanding

### 3. Resource Discovery
```python
# AIs can discover available tools and resources
tools = await client.list_tools()
resources = await client.list_resources()
```

### 4. Future-Proof
- Works with any MCP-compatible system
- Follows industry standards for AI tool integration
- Easy to extend with new functionality

## Testing MCP Integration

### Unit Tests for MCP Tools

```python
# tests/test_mcp_server.py
import pytest
from unittest.mock import AsyncMock
from mcp_server.hot_commands_server import HotCommandsMCPServer

class TestMCPServer:
    @pytest.fixture
    async def mcp_server(self):
        server = HotCommandsMCPServer()
        # Mock dependencies
        server.crud_service = AsyncMock()
        server.execution_engine = AsyncMock()
        return server
    
    @pytest.mark.asyncio
    async def test_list_hotcommands_tool(self, mcp_server):
        # Mock data
        mock_commands = [
            # ... mock command objects
        ]
        mcp_server.crud_service.list_commands.return_value = mock_commands
        
        # Test tool
        result = await mcp_server.list_hotcommands(
            user_id="test_user",
            domain="RAN"
        )
        
        assert result["total"] == len(mock_commands)
        assert result["filtered_by"]["domain"] == "RAN"
    
    @pytest.mark.asyncio
    async def test_create_hotcommand_tool(self, mcp_server):
        # Mock successful creation
        mock_command = # ... mock command object
        mcp_server.crud_service.create_command.return_value = mock_command
        
        # Test tool
        result = await mcp_server.create_hotcommand(
            user_id="test_user",
            command_name="test_cmd",
            query_text="show test data"
        )
        
        assert result["success"] is True
        assert result["command"]["name"] == "test_cmd"
```

### Integration Tests

```python
# tests/integration/test_mcp_integration.py
import pytest
from mcp.client import Client

class TestMCPIntegration:
    @pytest.mark.integration
    async def test_full_mcp_workflow(self, mcp_server_url):
        client = Client(mcp_server_url)
        
        # Create command via MCP
        create_result = await client.call_tool("create_hotcommand", {
            "user_id": "test_user",
            "command_name": "integration_test",
            "query_text": "show test data"
        })
        assert create_result["success"] is True
        
        # Execute command via MCP
        exec_result = await client.call_tool("execute_hotcommand", {
            "user_id": "test_user",
            "command_name": "integration_test"
        })
        assert exec_result["success"] is True
        
        # List commands via MCP
        list_result = await client.call_tool("list_hotcommands", {
            "user_id": "test_user"
        })
        assert len(list_result["commands"]) >= 1
```

## Deployment Checklist

- [ ] MCP server implementation complete
- [ ] Configuration files set up
- [ ] Docker containers configured
- [ ] Authentication/authorization implemented
- [ ] Error handling and logging in place
- [ ] Resource discovery implemented
- [ ] Integration tests passing
- [ ] Performance benchmarks met
- [ ] Security audit completed
- [ ] Documentation updated

## Migration Path

### Option 1: Gradual Migration
1. Build core service (Weeks 1-15)
2. Add MCP wrapper (Weeks 16-18)
3. Test dual interface (chatbot + MCP)
4. Gradually migrate AI interactions to MCP

### Option 2: MCP-First
1. Build with MCP from the start
2. Use MCP client for your chatbot interface
3. Single API layer for all interactions

## Conclusion

Implementing your hot commands as an MCP server provides:
- **Future-proof architecture** that works with any MCP-compatible AI
- **Reusable service** that other teams can leverage
- **Standardized protocol** with built-in error handling and discovery
- **Scalability** to serve multiple AI assistants

The phased approach ensures you get immediate value while building toward a more flexible, industry-standard architecture. 
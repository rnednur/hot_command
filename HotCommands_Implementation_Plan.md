# Hot Commands Implementation Plan for NL2SQL Framework

## Project Overview

### Objective
Build a robust, scalable hot commands CRUD framework within the existing NL2SQL slash command chatbot system, enabling users to create, manage, and execute personalized command shortcuts with advanced features like parameterization, sharing, and spaces.

### Scope
- Core CRUD operations for hot commands
- Advanced parameter handling and validation
- Integration with existing NL2SQL framework
- Spaces functionality for result persistence
- Security and access control
- Performance optimization and monitoring

### Success Criteria
- Users can create/manage hot commands in <2 seconds
- Support 1000+ hot commands per user
- 99.9% uptime for hot command operations
- Seamless integration with existing domain/category system
- Comprehensive audit logging and security

## Architecture Overview

### System Components
```
┌─────────────────────────────────────────────────────────────┐
│                    Chat Interface (React)                   │
├─────────────────────────────────────────────────────────────┤
│                 Command Router & Parser                     │
├─────────────────────────────────────────────────────────────┤
│  Hot Commands Framework  │  NL2SQL Engine  │  Tool Manager  │
├─────────────────────────────────────────────────────────────┤
│     PostgreSQL DB       │   Redis Cache    │  Cloud Storage │
└─────────────────────────────────────────────────────────────┘
```

### Core Modules
1. **HotCommand CRUD Service** - Core operations
2. **Parameter Engine** - Handle dynamic parameters
3. **Validation Layer** - Input/security validation
4. **Execution Engine** - Command execution routing
5. **Spaces Manager** - Result persistence
6. **Security Manager** - Auth/RBAC enforcement

## Phase 1: Foundation and Core CRUD (Weeks 1-3)

### 1.1 Database Design and Setup

#### Enhanced Database Schema
```sql
-- Main hot commands table
CREATE TABLE hot_commands (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    command_name VARCHAR(100) NOT NULL,
    display_name VARCHAR(150), -- User-friendly name
    description TEXT,
    query_text TEXT NOT NULL,
    query_type VARCHAR(20) NOT NULL CHECK (query_type IN ('nl2sql', 'direct_sql', 'tool_call')),
    domain VARCHAR(50),
    category VARCHAR(50),
    parameters JSONB DEFAULT '{}',
    metadata JSONB DEFAULT '{}',
    is_active BOOLEAN DEFAULT TRUE,
    is_shared BOOLEAN DEFAULT FALSE,
    usage_count INTEGER DEFAULT 0,
    last_used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    UNIQUE(user_id, command_name),
    
    -- Indexes
    CREATE INDEX idx_user_commands ON hot_commands (user_id, is_active),
    CREATE INDEX idx_domain_category ON hot_commands (domain, category),
    CREATE INDEX idx_shared_commands ON hot_commands (is_shared, is_active)
);

-- Command sharing table
CREATE TABLE hot_command_shares (
    id SERIAL PRIMARY KEY,
    command_id INTEGER REFERENCES hot_commands(id) ON DELETE CASCADE,
    shared_with_user_id VARCHAR(255),
    shared_with_team_id VARCHAR(255),
    permission_level VARCHAR(20) DEFAULT 'view' CHECK (permission_level IN ('view', 'execute', 'edit')),
    shared_by_user_id VARCHAR(255) NOT NULL,
    shared_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Either user or team, not both
    CHECK ((shared_with_user_id IS NOT NULL) != (shared_with_team_id IS NOT NULL))
);

-- Command execution history
CREATE TABLE hot_command_executions (
    id SERIAL PRIMARY KEY,
    command_id INTEGER REFERENCES hot_commands(id),
    user_id VARCHAR(255) NOT NULL,
    execution_params JSONB,
    execution_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    duration_ms INTEGER,
    success BOOLEAN,
    error_details TEXT,
    result_summary JSONB
);

-- Spaces table (enhanced from original requirements)
CREATE TABLE spaces (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    space_name VARCHAR(100) NOT NULL,
    display_name VARCHAR(150),
    description TEXT,
    domain VARCHAR(50),
    category VARCHAR(50),
    content_type VARCHAR(50) NOT NULL,
    content_location TEXT, -- File path for large content
    content_metadata JSONB DEFAULT '{}',
    content_preview TEXT, -- Small preview for UI
    size_bytes BIGINT DEFAULT 0,
    is_shared BOOLEAN DEFAULT FALSE,
    retention_days INTEGER DEFAULT 30,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    
    UNIQUE(user_id, space_name)
);
```

### 1.2 Core Data Models

```python
# models/hot_command.py
from dataclasses import dataclass
from typing import Dict, List, Optional, Any
from datetime import datetime
from enum import Enum

class QueryType(Enum):
    NL2SQL = "nl2sql"
    DIRECT_SQL = "direct_sql"
    TOOL_CALL = "tool_call"

class ParameterType(Enum):
    STRING = "string"
    INTEGER = "integer"
    FLOAT = "float"
    DATE = "date"
    DATETIME = "datetime"
    BOOLEAN = "boolean"
    LIST = "list"

@dataclass
class Parameter:
    name: str
    type: ParameterType
    required: bool = False
    default: Any = None
    description: str = ""
    options: List[Any] = None  # For enum-like parameters
    validation_regex: str = None

@dataclass
class HotCommand:
    id: Optional[int]
    user_id: str
    command_name: str
    display_name: str
    description: str
    query_text: str
    query_type: QueryType
    domain: Optional[str]
    category: Optional[str]
    parameters: Dict[str, Parameter]
    metadata: Dict[str, Any]
    is_active: bool = True
    is_shared: bool = False
    usage_count: int = 0
    last_used_at: Optional[datetime] = None
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None

@dataclass
class ExecutionResult:
    success: bool
    data: Any = None
    error: str = None
    execution_time_ms: int = 0
    result_type: str = "table"  # table, chart, text, json
```

### 1.3 Core CRUD Service Implementation

```python
# services/hot_command_crud.py
from typing import List, Optional, Dict, Any
from sqlalchemy.orm import Session
from models.hot_command import HotCommand, Parameter, QueryType
from utils.validation import HotCommandValidator
from utils.exceptions import *

class HotCommandCRUD:
    def __init__(self, db_session: Session, validator: HotCommandValidator):
        self.db = db_session
        self.validator = validator
    
    async def create_command(
        self,
        user_id: str,
        command_name: str,
        query_text: str,
        query_type: QueryType = QueryType.NL2SQL,
        display_name: str = None,
        description: str = "",
        domain: str = None,
        category: str = None,
        parameters: Dict[str, Parameter] = None,
        metadata: Dict[str, Any] = None
    ) -> HotCommand:
        """Create a new hot command with comprehensive validation"""
        
        # Validation pipeline
        await self._validate_creation_permissions(user_id, domain, category)
        await self._validate_command_name(user_id, command_name)
        await self._validate_query_syntax(query_text, query_type)
        await self._validate_parameters(parameters or {})
        
        # Create command
        command = HotCommand(
            id=None,
            user_id=user_id,
            command_name=command_name,
            display_name=display_name or command_name,
            description=description,
            query_text=query_text,
            query_type=query_type,
            domain=domain,
            category=category,
            parameters=parameters or {},
            metadata=metadata or {}
        )
        
        # Persist to database
        db_command = await self._save_to_db(command)
        
        # Register with command router
        await self._register_dynamic_command(command)
        
        return db_command
    
    async def get_command(self, user_id: str, command_name: str, include_shared: bool = True) -> Optional[HotCommand]:
        """Retrieve a specific hot command"""
        # Implementation details...
        pass
    
    async def list_commands(
        self,
        user_id: str,
        domain: str = None,
        category: str = None,
        include_shared: bool = True,
        search_query: str = None,
        limit: int = 100,
        offset: int = 0
    ) -> List[HotCommand]:
        """List hot commands with filtering and pagination"""
        # Implementation details...
        pass
    
    async def update_command(
        self,
        user_id: str,
        command_name: str,
        **updates
    ) -> HotCommand:
        """Update hot command with versioning"""
        # Implementation details...
        pass
    
    async def delete_command(self, user_id: str, command_name: str) -> bool:
        """Soft delete a hot command"""
        # Implementation details...
        pass
```

### Phase 1 Deliverables
- [ ] Database schema and migrations
- [ ] Core data models
- [ ] Basic CRUD service implementation
- [ ] Unit tests for CRUD operations
- [ ] API endpoints for basic operations

## Phase 2: Advanced Features (Weeks 4-6)

### 2.1 Parameter Engine

```python
# services/parameter_engine.py
import re
from typing import Dict, Any, List
from models.hot_command import Parameter, ParameterType

class ParameterEngine:
    def __init__(self):
        self.parameter_pattern = re.compile(r'\{\{(\w+)(?::(\w+))?(?::(\w+))?\}\}')
    
    def parse_query_parameters(self, query_text: str) -> Dict[str, Parameter]:
        """Extract parameter definitions from query text"""
        parameters = {}
        matches = self.parameter_pattern.findall(query_text)
        
        for name, type_str, required_str in matches:
            param_type = ParameterType(type_str) if type_str else ParameterType.STRING
            required = required_str == 'required'
            
            parameters[name] = Parameter(
                name=name,
                type=param_type,
                required=required
            )
        
        return parameters
    
    def substitute_parameters(
        self,
        query_text: str,
        command_parameters: Dict[str, Parameter],
        provided_parameters: Dict[str, Any]
    ) -> str:
        """Safely substitute parameters in query text"""
        # Implementation details...
        pass
```

### 2.2 Enhanced Validation Layer

```python
# utils/validation.py
import re
from typing import Dict, Any, List
from models.hot_command import HotCommand, Parameter, QueryType
from utils.exceptions import ValidationError

class HotCommandValidator:
    def __init__(self, nl2sql_engine, command_registry, security_manager):
        self.nl2sql_engine = nl2sql_engine
        self.command_registry = command_registry
        self.security = security_manager
        
        # Reserved command names that cannot be used
        self.reserved_commands = {
            'query', 'schema', 'help', 'set_domain', 'format',
            'export', 'chart', 'list_hotcommands', 'delete_hotcommand'
        }
    
    async def validate_command_name(self, user_id: str, command_name: str) -> bool:
        """Validate command name uniqueness and format"""
        # Check format
        if not re.match(r'^[a-zA-Z][a-zA-Z0-9_]*$', command_name):
            raise ValidationError("INVALID_NAME_FORMAT", "Command name must start with letter and contain only alphanumeric characters and underscores")
        
        # Check reserved names
        if command_name.lower() in self.reserved_commands:
            raise ValidationError("RESERVED_NAME", f"'{command_name}' is a reserved command name")
        
        # Check uniqueness
        existing = await self._check_existing_command(user_id, command_name)
        if existing:
            raise ValidationError("DUPLICATE_NAME", f"Command '{command_name}' already exists")
        
        return True
    
    async def validate_query_syntax(self, query_text: str, query_type: QueryType) -> bool:
        """Validate query syntax based on type"""
        if query_type == QueryType.NL2SQL:
            return await self._validate_nl2sql_query(query_text)
        elif query_type == QueryType.DIRECT_SQL:
            return await self._validate_sql_query(query_text)
        elif query_type == QueryType.TOOL_CALL:
            return await self._validate_tool_call(query_text)
```

### Phase 2 Deliverables
- [ ] Parameter engine implementation
- [ ] Enhanced validation layer
- [ ] Command execution engine
- [ ] Integration tests
- [ ] Performance benchmarks

## Phase 3: Spaces and Sharing (Weeks 7-9)

### 3.1 Spaces Management System

```python
# services/spaces_manager.py
from typing import List, Optional, Dict, Any
from models.space import Space, ContentType
from utils.storage import CloudStorageManager

class SpacesManager:
    def __init__(self, db_session, storage_manager: CloudStorageManager):
        self.db = db_session
        self.storage = storage_manager
        self.content_size_threshold = 1024 * 1024  # 1MB
    
    async def save_space(
        self,
        user_id: str,
        space_name: str,
        content: Any,
        content_type: ContentType,
        display_name: str = None,
        description: str = "",
        retention_days: int = 30
    ) -> Space:
        """Save content to a space with automatic storage optimization"""
        
        # Serialize content
        serialized_content, content_size = await self._serialize_content(content, content_type)
        
        # Determine storage location
        if content_size > self.content_size_threshold:
            # Store in cloud storage
            storage_key = f"spaces/{user_id}/{space_name}_{int(time.time())}"
            await self.storage.upload(storage_key, serialized_content)
            content_location = storage_key
            content_preview = await self._generate_preview(content, content_type)
        else:
            # Store in database
            content_location = None
            content_preview = serialized_content[:500]  # First 500 chars
        
        return await self._save_space_to_db(space)
```

### Phase 3 Deliverables
- [ ] Spaces management system
- [ ] Cloud storage integration
- [ ] Sharing mechanisms
- [ ] Content preview generation
- [ ] Retention and cleanup policies

## Phase 4: UI Integration (Weeks 10-12)

### 4.1 Chat Interface Enhancements

```typescript
// components/HotCommandPanel.tsx
import React, { useState, useEffect } from 'react';
import { HotCommand, ExecutionResult } from '../types/hotcommand';
import { hotCommandAPI } from '../services/api';

interface HotCommandPanelProps {
  userId: string;
  currentDomain?: string;
  currentCategory?: string;
}

export const HotCommandPanel: React.FC<HotCommandPanelProps> = ({
  userId,
  currentDomain,
  currentCategory
}) => {
  const [commands, setCommands] = useState<HotCommand[]>([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [isCreating, setIsCreating] = useState(false);

  const loadCommands = async () => {
    try {
      const response = await hotCommandAPI.listCommands({
        userId,
        domain: currentDomain,
        category: currentCategory,
        searchQuery
      });
      setCommands(response.data);
    } catch (error) {
      console.error('Failed to load commands:', error);
    }
  };

  useEffect(() => {
    loadCommands();
  }, [userId, currentDomain, currentCategory, searchQuery]);

  return (
    <div className="hot-command-panel">
      <div className="panel-header">
        <h3>Hot Commands</h3>
        <button onClick={() => setIsCreating(true)}>
          Create New
        </button>
      </div>
      
      <div className="search-section">
        <input
          type="text"
          placeholder="Search commands..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
        />
      </div>
      
      <div className="commands-list">
        {commands.map(command => (
          <CommandCard
            key={command.id}
            command={command}
            onExecute={handleExecuteCommand}
            onEdit={handleEditCommand}
            onDelete={handleDeleteCommand}
          />
        ))}
      </div>
    </div>
  );
};
```

### Phase 4 Deliverables
- [ ] Hot commands panel in chat interface
- [ ] Command creation wizard
- [ ] Parameter configuration UI
- [ ] Execution history view
- [ ] Sharing interface

## Phase 5: Security and Performance (Weeks 13-15)

### 5.1 Security Enhancements

```python
# security/rbac_manager.py
class RBACManager:
    def __init__(self, db_session):
        self.db = db_session
        self.permissions_cache = TTLCache(maxsize=1000, ttl=300)  # 5-minute cache
    
    async def check_hotcommand_permission(
        self,
        user_id: str,
        action: str,  # 'create', 'read', 'update', 'delete', 'execute', 'share'
        command: HotCommand = None,
        domain: str = None,
        category: str = None
    ) -> bool:
        """Check if user has permission for hot command operation"""
        
        cache_key = f"{user_id}:{action}:{domain}:{category}"
        if cache_key in self.permissions_cache:
            return self.permissions_cache[cache_key]
        
        # Get user roles and check permissions
        user_roles = await self._get_user_roles(user_id)
        has_permission = await self._check_domain_permissions(user_roles, action, domain, category)
        
        self.permissions_cache[cache_key] = has_permission
        return has_permission
```

### 5.2 Performance Optimization

```python
# performance/caching_layer.py
from redis import Redis
import json
from typing import Any, Optional

class HotCommandCache:
    def __init__(self, redis_client: Redis):
        self.redis = redis_client
        self.command_ttl = 3600  # 1 hour
        self.result_ttl = 300    # 5 minutes
    
    async def get_command(self, user_id: str, command_name: str) -> Optional[HotCommand]:
        """Get cached command"""
        cache_key = f"hotcmd:{user_id}:{command_name}"
        cached_data = await self.redis.get(cache_key)
        
        if cached_data:
            return HotCommand.from_dict(json.loads(cached_data))
        return None
    
    async def cache_command(self, command: HotCommand) -> None:
        """Cache command"""
        cache_key = f"hotcmd:{command.user_id}:{command.command_name}"
        await self.redis.setex(
            cache_key,
            self.command_ttl,
            json.dumps(command.to_dict())
        )
```

### Phase 5 Deliverables
- [ ] Comprehensive RBAC implementation
- [ ] Input sanitization and validation
- [ ] Caching layer with Redis
- [ ] Performance monitoring
- [ ] Security audit logging

## Phase 6: Testing and Documentation (Weeks 16-18)

### 6.1 Testing Strategy

```python
# tests/test_hotcommand_crud.py
import pytest
from unittest.mock import AsyncMock
from services.hot_command_crud import HotCommandCRUD
from models.hot_command import HotCommand, QueryType

class TestHotCommandCRUD:
    @pytest.fixture
    async def crud_service(self):
        db_session = AsyncMock()
        validator = AsyncMock()
        return HotCommandCRUD(db_session, validator)
    
    @pytest.mark.asyncio
    async def test_create_command_success(self, crud_service):
        """Test successful command creation"""
        # Arrange
        user_id = "test_user"
        command_name = "test_command"
        query_text = "show top 5 products by sales"
        
        # Act
        result = await crud_service.create_command(
            user_id=user_id,
            command_name=command_name,
            query_text=query_text,
            query_type=QueryType.NL2SQL
        )
        
        # Assert
        assert result.command_name == command_name
        assert result.user_id == user_id
        assert result.query_text == query_text
```

### 6.2 Documentation

```markdown
# Hot Commands User Guide

## Getting Started

### Creating Your First Hot Command

```
/hotcommand my_sales show top 5 products by sales this month
```

This creates a new command called `my_sales` that you can execute later with:

```
/my_sales
```

### Advanced Command Creation

For commands with parameters:

```
/hotcommand sales_by_region show sales for {{region:string:required}} in {{period:string:default=month}}
```

Execute with specific parameters:

```
/sales_by_region region=US period=week
```

### Parameter Types

| Type | Description | Example |
|------|-------------|---------|
| string | Text value | `{{name:string}}` |
| integer | Whole number | `{{limit:integer:default=10}}` |
| date | Date in YYYY-MM-DD format | `{{start_date:date:required}}` |
| boolean | true/false value | `{{active:boolean:default=true}}` |

### Managing Your Commands

- List all commands: `/list_hotcommands`
- Edit a command: `/edit_hotcommand my_sales --query "new query text"`
- Delete a command: `/delete_hotcommand my_sales`
- Share a command: `/share_hotcommand my_sales team_analytics view`
```

### Phase 6 Deliverables
- [ ] Comprehensive test suite (unit, integration, load)
- [ ] User documentation and guides
- [ ] Developer API documentation
- [ ] Performance benchmarks
- [ ] Security audit report

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| Phase 1 | 3 weeks | Foundation, CRUD operations |
| Phase 2 | 3 weeks | Parameters, validation, execution |
| Phase 3 | 3 weeks | Spaces, sharing, collaboration |
| Phase 4 | 3 weeks | UI integration, user experience |
| Phase 5 | 3 weeks | Security, performance optimization |
| Phase 6 | 3 weeks | Testing, documentation, deployment |

**Total: 18 weeks (4.5 months)**

## Success Metrics

- **Performance**: Command execution <2s, creation <1s
- **Scalability**: Support 1000+ commands per user, 100 concurrent users
- **Reliability**: 99.9% uptime, <0.1% error rate
- **Usability**: 90% user adoption within first month
- **Security**: Zero security incidents, full audit compliance

## Risk Mitigation

| Risk | Impact | Mitigation Strategy |
|------|--------|-------------------|
| Database performance degradation | High | Implement caching, optimize queries, add read replicas |
| Security vulnerabilities | High | Regular security audits, input validation, RBAC enforcement |
| Integration complexity | Medium | Modular architecture, comprehensive testing |
| User adoption resistance | Medium | Gradual rollout, training, user feedback integration |
| Scalability issues | Medium | Load testing, auto-scaling, performance monitoring |

## MCP (Model Context Protocol) Integration

### Why Choose MCP Architecture?

The hot commands functionality is an ideal candidate for MCP server implementation because:

- **Future-Proof**: Works with any MCP-compatible AI assistant (Claude, GPT, etc.)
- **Reusable**: Other teams/projects can leverage your hot commands service
- **Standardized**: Follows modern AI tool integration patterns
- **Maintainable**: Clean separation between AI interface and business logic
- **Scalable**: Can serve multiple AI assistants simultaneously

### MCP Implementation Strategy

We recommend a **phased approach** that builds the core service first, then adds MCP compatibility:

#### Phase 1-5: Core Service Development (Weeks 1-15)
Build the hot commands service with MCP-ready architecture but internal APIs first.

#### Phase 6: MCP Server Implementation (Weeks 16-18)
Add MCP wrapper and protocol compliance.

### MCP Server Architecture

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

### MCP Server Implementation

```python
# mcp_server/hot_commands_server.py
from mcp import Server, Resource, Tool
from mcp.errors import McpError
from typing import List, Dict, Any
import asyncio

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
            """List user's hot commands with optional filtering."""
            try:
                commands = await self.crud_service.list_commands(
                    user_id=user_id,
                    domain=domain,
                    category=category,
                    search_query=search,
                    include_shared=include_shared
                )
                
                return {
                    "commands": [cmd.to_dict() for cmd in commands],
                    "total": len(commands),
                    "filtered_by": {"domain": domain, "category": category, "search": search}
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
            """Create a new hot command."""
            try:
                command = await self.crud_service.create_command(
                    user_id=user_id,
                    command_name=command_name,
                    query_text=query_text,
                    query_type=QueryType(query_type),
                    display_name=display_name,
                    description=description,
                    domain=domain,
                    category=category,
                    parameters=parameters or {}
                )
                
                return {
                    "success": True,
                    "command": command.to_dict(),
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
            """Execute a hot command with optional parameters."""
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
                
                return {
                    "success": result.success,
                    "data": result.data,
                    "execution_time_ms": result.execution_time_ms,
                    "error": result.error
                }
            except Exception as e:
                raise McpError(f"Failed to execute hot command: {str(e)}", code=-32004)

# MCP Server Entry Point
async def main():
    server = HotCommandsMCPServer()
    from mcp.server.stdio import serve_stdio
    await serve_stdio(server.server)

if __name__ == "__main__":
    asyncio.run(main())
```

### MCP Configuration

```yaml
# mcp_config.yaml
name: "hotcommands"
version: "1.0.0"
description: "Hot Commands MCP Server for NL2SQL Framework"

tools:
  - name: "list_hotcommands"
    description: "List user's hot commands with filtering"
  - name: "create_hotcommand" 
    description: "Create a new hot command"
  - name: "execute_hotcommand"
    description: "Execute a hot command with parameters"

resources:
  - pattern: "hotcommand://{user_id}/{command_name}"
    description: "Individual hot command details"
```

### Updated Timeline with MCP

| Phase | Duration | Key Deliverables | MCP Integration |
|-------|----------|------------------|-----------------|
| Phase 1 | 3 weeks | Foundation, CRUD operations | MCP-ready service design |
| Phase 2 | 3 weeks | Parameters, validation, execution | Internal APIs compatible with MCP |
| Phase 3 | 3 weeks | Spaces, sharing, collaboration | Resource patterns for MCP |
| Phase 4 | 3 weeks | UI integration, user experience | Dual interface (UI + MCP prep) |
| Phase 5 | 3 weeks | Security, performance optimization | MCP security considerations |
| **Phase 6** | **3 weeks** | **Testing, documentation, MCP server** | **Full MCP implementation** |

### MCP Benefits

- **Multi-Assistant Support**: Works with Claude, GPT, and other MCP-compatible AIs
- **Standardized Protocol**: Built-in error handling, progress tracking, resource discovery
- **Future-Proof**: Follows modern AI tool integration patterns
- **Reusable**: Other teams can leverage your hot commands service

This MCP integration transforms your hot commands from a single-purpose chatbot feature into a reusable, standardized AI tool that can work with any MCP-compatible system while maintaining full backward compatibility with your existing chatbot interface.
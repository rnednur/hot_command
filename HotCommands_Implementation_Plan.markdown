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
    INDEX idx_user_commands (user_id, is_active),
    INDEX idx_domain_category (domain, category),
    INDEX idx_shared_commands (is_shared, is_active),
    INDEX idx_command_search (user_id, command_name, display_name)
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

-- Command versions (for audit trail)
CREATE TABLE hot_command_versions (
    id SERIAL PRIMARY KEY,
    command_id INTEGER REFERENCES hot_commands(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    query_text TEXT NOT NULL,
    parameters JSONB,
    metadata JSONB,
    changed_by VARCHAR(255) NOT NULL,
    change_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
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
    
    UNIQUE(user_id, space_name),
    INDEX idx_user_spaces (user_id, is_shared),
    INDEX idx_expiration (expires_at, is_shared)
);
```

#### Database Setup Tasks
- [ ] Create migration scripts
- [ ] Set up connection pooling
- [ ] Configure backup strategies
- [ ] Implement database monitoring

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
        
        # Create initial version
        await self._create_version(db_command.id, query_text, parameters, metadata, user_id, "Initial creation")
        
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
    
    async def bulk_operations(self, user_id: str, operations: List[Dict]) -> Dict[str, Any]:
        """Handle bulk create/update/delete operations"""
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
    
    def validate_execution_parameters(
        self,
        command_parameters: Dict[str, Parameter],
        provided_parameters: Dict[str, Any]
    ) -> ValidationResult:
        """Validate parameters provided during execution"""
        # Implementation details...
        pass
    
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
    
    async def validate_command_name(self, user_id: str, command_name: str) -> ValidationResult:
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
        
        return ValidationResult(valid=True)
    
    async def validate_query_syntax(self, query_text: str, query_type: QueryType) -> ValidationResult:
        """Validate query syntax based on type"""
        if query_type == QueryType.NL2SQL:
            return await self._validate_nl2sql_query(query_text)
        elif query_type == QueryType.DIRECT_SQL:
            return await self._validate_sql_query(query_text)
        elif query_type == QueryType.TOOL_CALL:
            return await self._validate_tool_call(query_text)
    
    async def validate_domain_permissions(self, user_id: str, domain: str, category: str) -> bool:
        """Check if user can create commands in domain/category"""
        return await self.security.check_domain_permissions(user_id, domain, category, 'create_hotcommand')
```

### 2.3 Command Execution Engine

```python
# services/execution_engine.py
from typing import Dict, Any
from models.hot_command import HotCommand, ExecutionResult, QueryType

class HotCommandExecutionEngine:
    def __init__(self, nl2sql_engine, sql_executor, tool_manager, parameter_engine):
        self.nl2sql_engine = nl2sql_engine
        self.sql_executor = sql_executor
        self.tool_manager = tool_manager
        self.parameter_engine = parameter_engine
    
    async def execute_command(
        self,
        command: HotCommand,
        user_id: str,
        provided_parameters: Dict[str, Any] = None,
        execution_context: Dict[str, Any] = None
    ) -> ExecutionResult:
        """Execute a hot command with full parameter substitution"""
        
        start_time = time.time()
        
        try:
            # Validate and substitute parameters
            final_query = await self._prepare_query(command, provided_parameters)
            
            # Route to appropriate execution engine
            if command.query_type == QueryType.NL2SQL:
                result = await self._execute_nl2sql(final_query, execution_context)
            elif command.query_type == QueryType.DIRECT_SQL:
                result = await self._execute_sql(final_query, execution_context)
            elif command.query_type == QueryType.TOOL_CALL:
                result = await self._execute_tool_call(final_query, execution_context)
            
            # Log execution
            await self._log_execution(command.id, user_id, provided_parameters, True, None, int((time.time() - start_time) * 1000))
            
            # Update usage statistics
            await self._update_usage_stats(command.id)
            
            return ExecutionResult(
                success=True,
                data=result,
                execution_time_ms=int((time.time() - start_time) * 1000)
            )
            
        except Exception as e:
            await self._log_execution(command.id, user_id, provided_parameters, False, str(e), int((time.time() - start_time) * 1000))
            return ExecutionResult(
                success=False,
                error=str(e),
                execution_time_ms=int((time.time() - start_time) * 1000)
            )
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
        
        # Create space record
        space = Space(
            user_id=user_id,
            space_name=space_name,
            display_name=display_name or space_name,
            description=description,
            content_type=content_type,
            content_location=content_location,
            content_preview=content_preview,
            size_bytes=content_size,
            retention_days=retention_days,
            expires_at=datetime.now() + timedelta(days=retention_days)
        )
        
        return await self._save_space_to_db(space)
```

### 3.2 Sharing System

```python
# services/sharing_service.py
class SharingService:
    def __init__(self, db_session, security_manager):
        self.db = db_session
        self.security = security_manager
    
    async def share_command(
        self,
        owner_user_id: str,
        command_name: str,
        share_with: str,  # user_id or team_id
        permission_level: str = "view",
        is_team: bool = False
    ) -> bool:
        """Share a hot command with another user or team"""
        
        # Validate permissions
        if not await self.security.can_share_command(owner_user_id, command_name):
            raise PermissionDeniedError("Cannot share this command")
        
        # Create sharing record
        share_record = HotCommandShare(
            command_id=command_id,
            shared_with_user_id=share_with if not is_team else None,
            shared_with_team_id=share_with if is_team else None,
            permission_level=permission_level,
            shared_by_user_id=owner_user_id
        )
        
        return await self._save_share_record(share_record)
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
  const [selectedCommand, setSelectedCommand] = useState<HotCommand | null>(null);

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
      
      {isCreating && (
        <CreateCommandModal
          onClose={() => setIsCreating(false)}
          onSave={handleCreateCommand}
          currentDomain={currentDomain}
          currentCategory={currentCategory}
        />
      )}
    </div>
  );
};
```

### 4.2 Command Creation Wizard

```typescript
// components/CreateCommandModal.tsx
export const CreateCommandModal: React.FC<CreateCommandModalProps> = ({
  onClose,
  onSave,
  currentDomain,
  currentCategory
}) => {
  const [formData, setFormData] = useState({
    commandName: '',
    displayName: '',
    description: '',
    queryText: '',
    queryType: 'nl2sql',
    parameters: {},
    domain: currentDomain,
    category: currentCategory
  });
  
  const [detectedParameters, setDetectedParameters] = useState([]);
  const [isValidating, setIsValidating] = useState(false);
  const [validationResult, setValidationResult] = useState(null);

  const handleQueryTextChange = async (queryText: string) => {
    setFormData(prev => ({ ...prev, queryText }));
    
    // Auto-detect parameters
    const params = await hotCommandAPI.detectParameters(queryText);
    setDetectedParameters(params);
  };

  const validateCommand = async () => {
    setIsValidating(true);
    try {
      const result = await hotCommandAPI.validateCommand(formData);
      setValidationResult(result);
    } catch (error) {
      setValidationResult({ valid: false, errors: [error.message] });
    }
    setIsValidating(false);
  };

  return (
    <Modal title="Create Hot Command" onClose={onClose}>
      <form onSubmit={handleSubmit}>
        <div className="form-section">
          <label>Command Name *</label>
          <input
            type="text"
            value={formData.commandName}
            onChange={(e) => setFormData(prev => ({ ...prev, commandName: e.target.value }))}
            placeholder="e.g., high_util_cells"
          />
        </div>
        
        <div className="form-section">
          <label>Query Text *</label>
          <textarea
            value={formData.queryText}
            onChange={(e) => handleQueryTextChange(e.target.value)}
            placeholder="Enter your natural language query or SQL..."
            rows={4}
          />
        </div>
        
        {detectedParameters.length > 0 && (
          <div className="parameters-section">
            <h4>Detected Parameters</h4>
            {detectedParameters.map(param => (
              <ParameterEditor
                key={param.name}
                parameter={param}
                onChange={handleParameterChange}
              />
            ))}
          </div>
        )}
        
        <div className="form-actions">
          <button type="button" onClick={validateCommand} disabled={isValidating}>
            {isValidating ? 'Validating...' : 'Validate'}
          </button>
          <button type="submit" disabled={!validationResult?.valid}>
            Create Command
          </button>
        </div>
      </form>
    </Modal>
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
        
        # Get user roles
        user_roles = await self._get_user_roles(user_id)
        
        # Check domain/category permissions
        has_permission = await self._check_domain_permissions(user_roles, action, domain, category)
        
        # Check command-specific permissions (for shared commands)
        if command and not has_permission:
            has_permission = await self._check_command_access(user_id, command, action)
        
        self.permissions_cache[cache_key] = has_permission
        return has_permission

# security/input_sanitizer.py
class InputSanitizer:
    def __init__(self):
        self.sql_injection_patterns = [
            r"(\bUNION\b.*\bSELECT\b)",
            r"(\bDROP\b.*\bTABLE\b)",
            r"(\bDELETE\b.*\bFROM\b)",
            r"(\bINSERT\b.*\bINTO\b)",
            r"(\bUPDATE\b.*\bSET\b)",
            # Add more patterns
        ]
    
    def sanitize_command_name(self, command_name: str) -> str:
        """Sanitize command name input"""
        # Remove special characters, limit length
        sanitized = re.sub(r'[^a-zA-Z0-9_]', '', command_name)
        return sanitized[:100]  # Limit length
    
    def validate_query_safety(self, query_text: str, query_type: str) -> bool:
        """Check if query is safe to execute"""
        if query_type == "direct_sql":
            # Check for dangerous SQL patterns
            for pattern in self.sql_injection_patterns:
                if re.search(pattern, query_text, re.IGNORECASE):
                    raise SecurityError(f"Potentially dangerous SQL detected: {pattern}")
        
        return True
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
    
    async def invalidate_user_commands(self, user_id: str) -> None:
        """Invalidate all cached commands for a user"""
        pattern = f"hotcmd:{user_id}:*"
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)

# performance/metrics_collector.py
class MetricsCollector:
    def __init__(self, monitoring_service):
        self.monitoring = monitoring_service
    
    async def record_command_execution(
        self,
        command_name: str,
        user_id: str,
        execution_time_ms: int,
        success: bool,
        error_type: str = None
    ):
        """Record command execution metrics"""
        
        tags = {
            'command_name': command_name,
            'success': str(success).lower(),
            'error_type': error_type or 'none'
        }
        
        # Record timing
        await self.monitoring.histogram(
            'hotcommand_execution_time',
            execution_time_ms,
            tags=tags
        )
        
        # Record count
        await self.monitoring.increment(
            'hotcommand_executions_total',
            tags=tags
        )
        
        # Record error rate
        if not success:
            await self.monitoring.increment(
                'hotcommand_errors_total',
                tags=tags
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
from unittest.mock import AsyncMock, MagicMock
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
    
    @pytest.mark.asyncio
    async def test_create_command_duplicate_name(self, crud_service):
        """Test command creation with duplicate name"""
        # Setup mock to return existing command
        crud_service.validator.validate_command_name.side_effect = ValidationError("DUPLICATE_NAME", "Command already exists")
        
        # Test
        with pytest.raises(ValidationError) as exc_info:
            await crud_service.create_command(
                user_id="test_user",
                command_name="existing_command",
                query_text="test query"
            )
        
        assert exc_info.value.code == "DUPLICATE_NAME"

# tests/test_parameter_engine.py
class TestParameterEngine:
    @pytest.fixture
    def parameter_engine(self):
        return ParameterEngine()
    
    def test_parse_query_parameters(self, parameter_engine):
        """Test parameter parsing from query text"""
        query = "show sales for {{date:date:required}} in region {{region:string}}"
        
        parameters = parameter_engine.parse_query_parameters(query)
        
        assert len(parameters) == 2
        assert parameters['date'].type == ParameterType.DATE
        assert parameters['date'].required == True
        assert parameters['region'].type == ParameterType.STRING
        assert parameters['region'].required == False

# tests/integration/test_full_workflow.py
class TestFullWorkflow:
    @pytest.mark.integration
    async def test_complete_hotcommand_lifecycle(self, test_client, test_db):
        """Test complete lifecycle: create -> execute -> update -> delete"""
        
        # Create command
        create_response = await test_client.post('/api/hotcommands', json={
            'command_name': 'test_workflow',
            'query_text': 'show top {{limit:integer:default=5}} products',
            'query_type': 'nl2sql'
        })
        assert create_response.status_code == 201
        
        # Execute command
        exec_response = await test_client.post('/api/hotcommands/test_workflow/execute', json={
            'parameters': {'limit': 10}
        })
        assert exec_response.status_code == 200
        
        # Update command
        update_response = await test_client.put('/api/hotcommands/test_workflow', json={
            'description': 'Updated description'
        })
        assert update_response.status_code == 200
        
        # Delete command
        delete_response = await test_client.delete('/api/hotcommands/test_workflow')
        assert delete_response.status_code == 204
```

### 6.2 Load Testing

```python
# tests/load/test_performance.py
import asyncio
import aiohttp
import time
from concurrent.futures import ThreadPoolExecutor

class LoadTestRunner:
    def __init__(self, base_url: str, concurrent_users: int = 100):
        self.base_url = base_url
        self.concurrent_users = concurrent_users
    
    async def test_command_creation_load(self):
        """Test command creation under load"""
        async def create_command(session, user_id, command_index):
            start_time = time.time()
            async with session.post(f'{self.base_url}/api/hotcommands', json={
                'command_name': f'load_test_cmd_{user_id}_{command_index}',
                'query_text': f'show data for user {user_id}',
                'query_type': 'nl2sql'
            }) as response:
                duration = time.time() - start_time
                return response.status, duration
        
        # Run concurrent requests
        async with aiohttp.ClientSession() as session:
            tasks = []
            for user_id in range(self.concurrent_users):
                for cmd_index in range(5):  # 5 commands per user
                    task = create_command(session, user_id, cmd_index)
                    tasks.append(task)
            
            results = await asyncio.gather(*tasks)
        
        # Analyze results
        success_count = sum(1 for status, _ in results if status == 201)
        avg_duration = sum(duration for _, duration in results) / len(results)
        
        print(f"Success rate: {success_count}/{len(results)} ({success_count/len(results)*100:.1f}%)")
        print(f"Average response time: {avg_duration:.3f}s")
        
        assert success_count / len(results) >= 0.95  # 95% success rate
        assert avg_duration <= 2.0  # Max 2 seconds average
```

### 6.3 Documentation

```markdown
# Hot Commands User Guide

## Getting Started

### Creating Your First Hot Command

The simplest way to create a hot command is using the `/hotcommand` slash command:

```
/hotcommand my_sales show top 5 products by sales this month
```

This creates a new command called `my_sales` that you can execute later with:

```
/my_sales
```

### Advanced Command Creation

For more complex commands with parameters:

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

### Spaces

Save query results for later use or sharing:

```
/query show monthly sales by region
/save_space monthly_sales_report table
/share_space monthly_sales_report marketing_team view
```

Access saved spaces:

```
/view_space monthly_sales_report
/list_spaces
```
```

### Phase 6 Deliverables
- [ ] Comprehensive test suite (unit, integration, load)
- [ ] User documentation and guides
- [ ] Developer API documentation
- [ ] Performance benchmarks
- [ ] Security audit report

## Deployment Plan

### Infrastructure Requirements

```yaml
# docker-compose.yml
version: '3.8'
services:
  hotcommands-api:
    build: ./api
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/hotcommands
      - REDIS_URL=redis://redis:6379
      - CLOUD_STORAGE_BUCKET=hotcommands-spaces
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

### Monitoring Setup

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'hotcommands'
    static_configs:
      - targets: ['hotcommands-api:8000']
    metrics_path: '/metrics'

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Deployment Checklist

- [ ] Database migrations executed
- [ ] Environment variables configured
- [ ] SSL certificates installed
- [ ] Monitoring and alerting configured
- [ ] Backup procedures tested
- [ ] Load balancer configured
- [ ] Auto-scaling policies set
- [ ] Security scan completed

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

This implementation plan provides a comprehensive roadmap for building a robust, scalable hot commands framework that integrates seamlessly with your existing NL2SQL chatbot system. 
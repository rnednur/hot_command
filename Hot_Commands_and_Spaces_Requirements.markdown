# Requirements for Hot Commands and Spaces in NL2SQL Slash Command Framework

## 1. Overview
This document extends the NL2SQL slash command framework to include:
- **Hot Commands**: Allow users to persist frequently used queries as named shortcuts (e.g., `/hotcommand top_sales` to save `/query show top 5 products by sales`).
- **Spaces**: Enable users to persist query outputs in shared spaces for collaboration, accessible by other users or teams.

These features integrate with the existing framework, supporting domain/category demarcation (e.g., RAN/Capacity, CustomerExperience/Mobility) and external tool integrations.

## 2. Design-Time Requirements

### 2.1. Hot Commands
- **Command Definition**:
  - Users can create hot commands using `/hotcommand <name> [query]`, which saves a query (natural language or SQL) for reuse.
  - Example: `/hotcommand top_sales show top 5 products by sales` creates `/top_sales` to execute the query.
- **Storage**:
  - Store hot commands in a database table or file per user, linked to their domain/category.
  - Schema:
    ```sql
    CREATE TABLE hot_commands (
        id SERIAL PRIMARY KEY,
        user_id VARCHAR(255) NOT NULL,
        domain VARCHAR(50),
        category VARCHAR(50),
        command_name VARCHAR(100) UNIQUE,
        query_text TEXT NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP
    );
    ```
- **Validation**:
  - Ensure `command_name` is unique per user and does not conflict with existing slash commands (e.g., `/query`, `/schema`).
  - Validate `query_text` for syntax and domain/category compatibility before saving.
- **Metadata**:
  - Store optional metadata (e.g., description, output format) with each hot command.
  - Example: `{ "command_name": "top_sales", "description": "Top 5 products by sales in RAN/Capacity", "output_format": "table" }`.
- **Extensibility**:
  - Allow hot commands to reference NL2SQL queries or direct SQL queries.
  - Support parameterization for dynamic inputs (e.g., `/hotcommand sales_by_date show sales for [date]` allows `/sales_by_date 2025-06-20`).
- **Access Control**:
  - Restrict hot command creation/editing to authenticated users.
  - Allow users to share hot commands with teams (e.g., via `/share_hotcommand`).

### 2.2. Spaces
- **Definition**:
  - Spaces are persistent storage areas where users can save query outputs (e.g., tables, charts, CSV) for personal use or sharing with others.
  - Example: `/save_space sales_report` saves the last query output to a space named "sales_report".
- **Storage**:
  - Store spaces in a database table or cloud storage (e.g., S3, GCP Cloud Storage) with metadata and content.
  - Schema:
    ```sql
    CREATE TABLE spaces (
        id SERIAL PRIMARY KEY,
        user_id VARCHAR(255) NOT NULL,
        space_name VARCHAR(100) NOT NULL,
        domain VARCHAR(50),
        category VARCHAR(50),
        content_type VARCHAR(50), -- e.g., table, csv, json, chart
        content TEXT, -- JSON or reference to file in cloud storage
        shared_with TEXT[], -- List of user_ids or team_ids
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP
    );
    ```
- **Content Types**:
  - Support multiple formats: table, CSV, JSON, chart (e.g., Chart.js JSON), or raw SQL output.
  - Store large outputs (e.g., CSVs) in cloud storage with a reference in the database.
- **Sharing Mechanism**:
  - Allow users to share spaces with individuals or teams via `/share_space <space_name> <user/team>`.
  - Support permission levels (e.g., view, edit) for shared spaces.
- **Access Control**:
  - Restrict access to spaces based on user authentication and RBAC (e.g., only `RAN_admin` can access RAN/Capacity spaces).
  - Encrypt sensitive data in spaces (e.g., customer data in CustomerExperience/Mobility).
- **Versioning**:
  - Optionally support versioning of spaces to track changes to shared outputs.
  - Example: Updating a space with new query results creates a new version.

### 2.3. Framework Integration
- **Command Registry**:
  - Extend the command registry to include hot commands dynamically.
  - Example: Register `/top_sales` as a user-defined command linked to a handler that executes the stored query.
- **Context Awareness**:
  - Ensure hot commands and spaces respect the current domain/category context (e.g., `/top_sales` in RAN/Capacity uses `bandwidth_usage` table).
- **UI Integration**:
  - Add a section in the chat interface to list and manage hot commands (e.g., `/list_hotcommands`).
  - Display spaces with previews (e.g., table snippets, chart thumbnails) and sharing options.
- **API Support**:
  - Expose REST API endpoints for external systems to create, manage, and access hot commands and spaces.
  - Example: `POST /api/hotcommand`, `GET /api/spaces/<space_name>`.

### 2.4. Security
- **Authentication**: Require user authentication for creating/editing hot commands and spaces.
- **Authorization**: Enforce RBAC for hot commands and spaces (e.g., only `CX_analyst` can save CustomerExperience/Mobility spaces).
- **Data Protection**: Encrypt sensitive query outputs in spaces (e.g., customer PII in CustomerExperience/FWA).
- **Audit Logging**: Log all hot command and space operations (create, edit, share, access) for auditing.

### 2.5. Development and Testing
- **Database Schema**: Design and test database tables for hot commands and spaces.
- **Unit Tests**: Test hot command creation, execution, and deletion; test space saving, sharing, and retrieval.
- **Mock Data**: Use mock queries and outputs to test hot commands and spaces across domains/categories.
- **Documentation**: Update developer and user guides to include hot commands and spaces usage.

## 3. Run-Time Requirements

### 3.1. Hot Commands
- **Execution**:
  - Execute hot commands (e.g., `/top_sales`) by retrieving and running the stored query.
  - Support parameterized queries (e.g., `/sales_by_date 2025-06-20` replaces `[date]` in the stored query).
  - Process hot commands in <1 second for simple queries, <5 seconds for complex NL2SQL or tool-based queries.
- **Management**:
  - `/list_hotcommands`: Display all user-defined hot commands with names and descriptions.
  - `/delete_hotcommand <name>`: Remove a hot command.
  - `/edit_hotcommand <name> [new_query]`: Update a hot command's query or metadata.
- **Concurrency**: Handle multiple users executing hot commands simultaneously without conflicts.
- **Error Handling**: Provide clear errors (e.g., "Hot command 'top_sales' not found. Use `/list_hotcommands`.").

### 3.2. Spaces
- **Saving Outputs**:
  - `/save_space <space_name>`: Save the last query output to a named space.
  - Support multiple formats (e.g., `/save_space sales_report csv` for CSV output).
  - Store outputs efficiently (e.g., large CSVs in cloud storage with metadata in the database).
- **Accessing Spaces**:
  - `/view_space <space_name>`: Display the content of a space (e.g., table, chart).
  - `/list_spaces`: List all spaces accessible to the user, including shared ones.
- **Sharing**:
  - `/share_space <space_name> <user/team> [permissions]`: Share a space with specified users or teams.
  - Example: `/share_space sales_report marketing_team view` grants view-only access.
- **Performance**:
  - Retrieve space contents in <2 seconds for small outputs, <10 seconds for large files (e.g., CSVs from cloud storage).
  - Optimize chart rendering for spaces with visualizations (e.g., <5 seconds for Chart.js charts).
- **Error Handling**: Handle invalid space names or permissions (e.g., "Space 'sales_report' not found or access denied.").

### 3.3. Integration with Existing Framework
- **Command Processing**:
  - Extend the command parser to recognize hot commands dynamically.
  - Route hot commands to the NL2SQL engine or direct SQL execution based on stored query type.
- **Tool Integration**:
  - Allow hot commands to leverage external tools (e.g., `/hotcommand capacity_alerts` calls RAN_Analytics_API).
  - Store tool outputs in spaces (e.g., `/save_space capacity_alerts` saves API response).
- **Context Management**:
  - Ensure hot commands and spaces inherit the user's current domain/category (e.g., RAN/Capacity).
  - Validate queries/outputs against the domain/category schema before saving.

### 3.4. Security
- **Access Control**: Restrict hot command execution and space access based on user roles and domain/category permissions.
- **Data Encryption**: Encrypt space contents in transit and at rest (e.g., AES-256 for cloud storage).
- **Rate Limiting**: Prevent abuse of hot commands or space operations (e.g., limit to 100 hot command executions/hour per user).

### 3.5. Monitoring and Logging
- **Usage Tracking**: Log hot command executions and space operations (create, view, share) with timestamps and user IDs.
- **Performance Metrics**: Monitor latency for hot command execution and space retrieval.
- **Error Logging**: Log errors (e.g., failed query in hot command, inaccessible space) for debugging.

## 4. Domain/Category-Specific Considerations
- **Hot Commands**:
  - RAN/Capacity: `/hotcommand high_util_cells show cells with utilization > 80%`.
  - RAN/RF: `/hotcommand rf_issues show cells with signal strength < -90 dBm`.
  - CustomerExperience/Mobility: `/hotcommand churn_risk show users with churn probability > 70%`.
  - CustomerExperience/FWA: `/hotcommand fwa_latency show FWA users with latency > 50ms`.
- **Spaces**:
  - RAN/Capacity: Save capacity trend charts (e.g., `/save_space capacity_trend chart`).
  - RAN/RF: Store interference maps (e.g., `/save_space interference_area_456 map`).
  - CustomerExperience/Mobility: Share churn risk reports (e.g., `/share_space churn_report marketing_team`).
  - CustomerExperience/FWA: Save FWA health summaries (e.g., `/save_space fwa_health_site_789 table`).

## 5. Non-Functional Requirements
- **Performance**: Hot command execution in <5 seconds, space retrieval in <10 seconds for large outputs.
- **Scalability**: Support 1000+ hot commands per user, 10,000+ spaces across all users.
- **Availability**: Ensure 99.9% uptime for hot command and space operations.
- **Usability**: Allow users to create hot commands and spaces with minimal steps (e.g., single command).
- **Security**: Comply with GDPR/CCPA for spaces containing sensitive data (e.g., customer PII).

## 6. Implementation Notes
- **Tech Stack**:
  - Backend: Extend existing Python (Flask/FastAPI) backend to handle hot commands and spaces.
  - Database: PostgreSQL for storing hot commands and space metadata.
  - Storage: AWS S3/GCP Cloud Storage for large space contents (e.g., CSVs, charts).
  - Frontend: Update React UI to display hot commands and spaces (e.g., sidebar with `/list_hotcommands`, `/list_spaces`).
- **API Endpoints**:
  - `POST /api/hotcommand`: Create/edit a hot command.
  - `GET /api/hotcommands`: List user's hot commands.
  - `POST /api/space`: Save a space.
  - `GET /api/spaces/<space_name>`: Retrieve a space.
  - `POST /api/share_space`: Share a space.
- **Testing**:
  - Unit tests for hot command creation, execution, and deletion.
  - Integration tests for space saving, retrieval, and sharing.
  - Load tests for 1000 concurrent hot command executions and space accesses.
- **Documentation**:
  - User guide: How to create/use hot commands and spaces (e.g., `/hotcommand`, `/save_space`).
  - Developer guide: Adding hot command handlers and space storage integrations.

## 7. Example Workflow
```plaintext
User: /set_domain RAN/Capacity
Bot: Context set to RAN/Capacity.

User: /hotcommand high_util_cells show cells with utilization > 80%
Bot: Hot command 'high_util_cells' created.

User: /high_util_cells
Bot: SQL: SELECT cell_id, utilization FROM bandwidth_usage WHERE utilization > 0.8;
Results:
| Cell ID | Utilization |
|---------|-------------|
| cell_123| 0.85        |
| cell_456| 0.90        |

User: /save_space high_util_report table
Bot: Output saved to space 'high_util_report'.

User: /share_space high_util_report network_team view
Bot: Space 'high_util_report' shared with network_team (view-only).

User: /list_hotcommands
Bot: Hot Commands:
- high_util_cells: Show cells with utilization > 80%

User: /list_spaces
Bot: Spaces:
- high_util_report (table, shared with network_team)
```
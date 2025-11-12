# ArangoDB Knowledge Graph - Implementation Action Plan

## Project Overview
Build a knowledge graph management system using Python, FastAPI, and ArangoDB that maintains isolated course-based graphs with knowledge nodes and prerequisite/sub-topic relationships.

---

## Phase 1: Environment Setup

### 1.1 Project Structure Setup
- [ ] Create project directory structure
  - `/app` - Main application code
  - `/app/models` - Pydantic models
  - `/app/api` - API route handlers
  - `/app/database` - Database connection and operations
  - `/app/schemas` - Request/response schemas
  - `/app/utils` - Utility functions
  - `/tests` - Test files

### 1.2 Dependencies Installation
- [ ] Create `requirements.txt` with:
  - `fastapi` - Web framework
  - `uvicorn` - ASGI server
  - `python-arango` - ArangoDB driver
  - `pydantic` - Data validation
  - `pydantic-settings` - Configuration management
  - `python-dotenv` - Environment variable management
  - `pytest` - Testing framework
  - `httpx` - HTTP client for testing

### 1.3 ArangoDB Setup
- [ ] Install ArangoDB locally or use Docker
- [ ] Configure database connection settings
- [ ] Create database for the project
- [ ] Set up authentication credentials

### 1.4 Configuration
- [ ] Create `.env` file for environment variables
  - Database URL
  - Database name
  - Username/password
  - API configuration
- [ ] Create `config.py` for application settings
- [ ] Add `.gitignore` for Python projects

---

## Phase 2: Database Layer Implementation

### 2.1 Database Connection
- [ ] Implement database connection manager
- [ ] Create connection pooling/singleton pattern
- [ ] Implement health check endpoint
- [ ] Add error handling for connection failures

### 2.2 Collection Setup
- [ ] Create initialization script for collections
  - `graphs` collection (document)
  - `knowledge_nodes` collection (document)
  - `edges` collection (edge)
- [ ] Define and create named graph in ArangoDB

### 2.3 Index Creation
- [ ] Create hash index on `graphs.course_id`
- [ ] Create unique index on `graphs.graph_id`
- [ ] Create unique index on `knowledge_nodes.node_id`
- [ ] Create hash index on `knowledge_nodes.graph_id`
- [ ] Create unique index on `edges.edge_id`
- [ ] Create hash index on `edges.graph_id`
- [ ] Create edge indexes on `_from` and `_to`

### 2.4 Database Operations Module
- [ ] Implement CRUD operations for graphs
- [ ] Implement CRUD operations for nodes
- [ ] Implement CRUD operations for edges
- [ ] Add query methods for graph isolation validation
- [ ] Implement bulk operations for graph copying

---

## Phase 3: Data Models and Schemas

### 3.1 Pydantic Models
- [ ] Create `Graph` model
  - graph_id, course_id, is_prototype, timestamps
- [ ] Create `KnowledgeNode` model
  - node_id, graph_id, mastery_points, title, description, timestamps
- [ ] Create `Edge` model
  - edge_id, graph_id, edge_type, from/to references, timestamp
- [ ] Create enums for edge_type (prerequisite, sub-topic)

### 3.2 Request/Response Schemas
- [ ] Create request schemas for each API endpoint
- [ ] Create response schemas for each API endpoint
- [ ] Add validation rules and constraints
- [ ] Create error response schemas

---

## Phase 4: Business Logic Layer

### 4.1 Graph Operations
- [ ] Implement `create_graph()` function
  - Check if course_id already exists (must be unique for new graphs)
  - Throw exception if course_id exists
  - Generate unique graph_id
  - Set is_prototype to false
  - Store in database
- [ ] Implement `delete_graph()` function
  - Delete all associated edges
  - Delete all associated nodes
  - Delete graph document
  - Return deletion statistics
- [ ] Implement `copy_graph()` function
  - Clone graph metadata with NEW graph_id
  - Keep the SAME course_id as the original (this is how multiple graphs share course_id)
  - Create node_id mapping (old -> new)
  - Clone all nodes with new IDs
  - Clone all edges with new IDs and mapped node references

### 4.2 Node Operations
- [ ] Implement `create_node()` function
  - Validate graph exists
  - Generate globally unique node_id
  - Associate with graph_id
  - Store node with properties
- [ ] Implement `get_node()` function
  - Validate graph and node exist
  - Verify node belongs to specified graph
  - Return node object
- [ ] Implement `modify_node()` function
  - Validate graph and node exist
  - Verify node belongs to graph
  - Apply partial updates
  - Protect immutable fields
  - Update timestamp
- [ ] Implement `delete_node()` function
  - Validate graph and node exist
  - Find all connected edges (incoming/outgoing)
  - Delete all edges
  - Delete node
  - Return node_id and deleted edge_ids

### 4.3 Edge Operations
- [ ] Implement `create_edge()` function
  - Validate graph exists
  - Validate both nodes exist and belong to graph
  - Validate edge_type enum
  - Check for duplicate edges
  - Generate unique edge_id
  - Create edge document
- [ ] Implement `delete_edge()` function
  - Validate graph exists
  - Validate edge exists between nodes
  - Verify edge belongs to specified graph
  - Throw exception if nodes not connected
  - Delete edge

### 4.4 Graph Isolation Enforcement
- [ ] Create validation utility for graph membership
- [ ] Implement checks in all node operations
- [ ] Implement checks in all edge operations
- [ ] Add comprehensive error messages for violations

---

## Phase 5: API Layer Implementation

### 5.1 FastAPI Application Setup
- [ ] Create main FastAPI app instance
- [ ] Configure CORS middleware
- [ ] Set up exception handlers
- [ ] Configure API documentation
- [ ] Add API versioning (/api/v1)

### 5.2 Graph Endpoints
- [ ] `POST /api/v1/graphs` - Create graph
- [ ] `DELETE /api/v1/graphs/{graph_id}` - Delete graph
- [ ] `POST /api/v1/graphs/{graph_id}/copy` - Copy graph

### 5.3 Node Endpoints
- [ ] `POST /api/v1/graphs/{graph_id}/nodes` - Create node
- [ ] `GET /api/v1/graphs/{graph_id}/nodes/{node_id}` - Get node
- [ ] `PATCH /api/v1/graphs/{graph_id}/nodes/{node_id}` - Modify node
- [ ] `DELETE /api/v1/graphs/{graph_id}/nodes/{node_id}` - Delete node

### 5.4 Edge Endpoints
- [ ] `POST /api/v1/graphs/{graph_id}/edges` - Create edge
- [ ] `DELETE /api/v1/graphs/{graph_id}/edges` - Delete edge (by nodes)
- [ ] Optional: `DELETE /api/v1/graphs/{graph_id}/edges/{edge_id}` - Delete edge (by ID)

### 5.5 Health and Info Endpoints
- [ ] `GET /health` - Health check endpoint
- [ ] `GET /api/v1/info` - API information

---

## Phase 6: Error Handling and Validation

### 6.1 Custom Exceptions
- [ ] Create `GraphNotFoundException`
- [ ] Create `NodeNotFoundException`
- [ ] Create `EdgeNotFoundException`
- [ ] Create `GraphIsolationViolationException`
- [ ] Create `DuplicateCourseIdException`
- [ ] Create `NodesNotConnectedException`

### 6.2 Exception Handlers
- [ ] Implement global exception handler
- [ ] Map exceptions to HTTP status codes
- [ ] Create consistent error response format
- [ ] Add detailed error messages

### 6.3 Input Validation
- [ ] Validate all request bodies with Pydantic
- [ ] Validate path parameters
- [ ] Validate query parameters
- [ ] Add custom validators where needed

---

## Phase 7: Testing

### 7.1 Unit Tests
- [ ] Test database connection and operations
- [ ] Test each business logic function
- [ ] Test validation logic
- [ ] Test error conditions
- [ ] Test graph isolation enforcement

### 7.2 Integration Tests
- [ ] Test complete API endpoints
- [ ] Test graph lifecycle (create, copy, delete)
- [ ] Test node lifecycle (CRUD operations)
- [ ] Test edge lifecycle (create, delete)
- [ ] Test cascade deletions
- [ ] Test error scenarios

### 7.3 Test Data
- [ ] Create fixtures for test data
- [ ] Create sample graphs with nodes and edges
- [ ] Create cleanup utilities

---

## Phase 8: Documentation

### 8.1 API Documentation
- [ ] Enhance OpenAPI/Swagger documentation
- [ ] Add example requests/responses
- [ ] Document error codes
- [ ] Add authentication information (when implemented)

### 8.2 Code Documentation
- [ ] Add docstrings to all functions
- [ ] Document complex algorithms
- [ ] Add inline comments for clarity

### 8.3 User Documentation
- [ ] Create README with setup instructions
- [ ] Document environment variables
- [ ] Add usage examples
- [ ] Create troubleshooting guide

---

## Phase 9: Deployment Preparation

### 9.1 Docker Setup
- [ ] Create Dockerfile for FastAPI application
- [ ] Create docker-compose.yml with ArangoDB and API
- [ ] Add environment variable configuration
- [ ] Test containerized deployment

### 9.2 Configuration Management
- [ ] Separate development/production configs
- [ ] Add logging configuration
- [ ] Configure CORS for production
- [ ] Set up health monitoring

### 9.3 Performance Optimization
- [ ] Review and optimize database queries
- [ ] Add connection pooling if needed
- [ ] Implement caching where appropriate
- [ ] Add rate limiting considerations

---

## Phase 10: Demo Preparation

### 10.1 Sample Data Creation
- [ ] Create script to populate sample courses
- [ ] Generate realistic knowledge graphs
- [ ] Create diverse node and edge examples
- [ ] Add various graph structures (linear, branching, complex)

### 10.2 Demo Scenarios
- [ ] Prepare demo flow document
- [ ] Create sample API call collections (Postman/Insomnia)
- [ ] Test all operations with sample data
- [ ] Prepare rollback/reset procedures

### 10.3 Presentation Materials
- [ ] Create architecture diagrams
- [ ] Prepare code walkthrough
- [ ] Document key features and design decisions
- [ ] Create performance benchmarks

---

## Dependencies and Risks

### Critical Dependencies
1. ArangoDB must be running and accessible
2. Python 3.9+ required
3. Network connectivity for database access

### Technical Risks
1. **Graph Isolation**: Ensure strict enforcement in all operations
2. **Unique ID Generation**: Implement robust UUID generation
3. **Cascade Deletions**: Ensure complete cleanup without orphans
4. **Performance**: Large graphs may require query optimization
5. **Course ID Uniqueness**: Must enforce uniqueness check on create_graph but allow duplicates through copy_graph

### Clarifications Received
- **Course ID Behavior**: When creating a NEW graph, course_id must be unique (throw exception if exists). When COPYING a graph, it copies the same course_id, which is how multiple graphs can share the same course_id. This enables versioning and multiple instances of the same course.
- **API Count**: Specification mentions 11 APIs but only 9 distinct operations identified. 

### Recommended Additions
- **Authentication**: Not specified. Recommend adding API key or JWT authentication for production.
- **Query API**: Consider adding `GET /api/v1/graphs?course_id={course_id}` to retrieve all graphs for a given course_id.

---

## Estimated Timeline

- **Phase 1**: Environment Setup - 0.5 day
- **Phase 2**: Database Layer - 1 day
- **Phase 3**: Data Models - 0.5 day
- **Phase 4**: Business Logic - 2 days
- **Phase 5**: API Layer - 1.5 days
- **Phase 6**: Error Handling - 1 day
- **Phase 7**: Testing - 1.5 days
- **Phase 8**: Documentation - 1 day
- **Phase 9**: Deployment - 1 day
- **Phase 10**: Demo Prep - 0.5 day

**Total Estimated Time**: ~10-11 days

---

## Success Criteria

✓ All 9 APIs fully functional and tested
✓ Graph isolation strictly enforced
✓ All indexes created and optimized
✓ Comprehensive error handling
✓ API documentation complete and accurate
✓ Unit and integration tests passing
✓ Sample data loads successfully
✓ Demo scenarios execute smoothly
✓ System handles edge cases gracefully
✓ Code is clean, documented, and maintainable

---

## Next Steps

1. Review and approve this action plan
2. Clarify the requirements contradictions mentioned above
3. Set up development environment
4. Begin Phase 1: Environment Setup

# ArangoDB Knowledge Graph System Design

## Overview
This system implements a graph-based knowledge management platform using ArangoDB. Each course has isolated graphs containing knowledge nodes connected by prerequisite and sub-topic relationships.

---

## Data Model

### Collections

#### 1. **Graphs Collection** (Document Collection)
Stores metadata for each knowledge graph.

```json
{
  "_key": "graph_12345",
  "_id": "graphs/graph_12345",
  "graph_id": "graph_12345",
  "course_id": "CS101",
  "is_prototype": false,
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

**Properties:**
- `graph_id` (string, unique): Unique identifier for the graph
- `course_id` (string, indexed): Course identifier (unique when creating new graphs, but multiple graphs can share the same course_id through copying)
- `is_prototype` (boolean): Whether this is a prototype graph
- `created_at` (datetime): Creation timestamp
- `updated_at` (datetime): Last modification timestamp

**Indexes:**
- Hash index on `course_id` for efficient course-based queries (important for finding all versions of a course)
- Unique index on `graph_id`

---

#### 2. **Knowledge_Nodes Collection** (Document Collection)
Stores individual knowledge nodes within graphs.

```json
{
  "_key": "node_67890",
  "_id": "knowledge_nodes/node_67890",
  "node_id": "node_67890",
  "graph_id": "graph_12345",
  "mastery_points": 100,
  "title": "Introduction to Variables",
  "description": "Learn about variables in programming",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

**Properties:**
- `node_id` (string, globally unique): Unique identifier for the node
- `graph_id` (string, indexed): Reference to parent graph
- `mastery_points` (integer): Points associated with mastering this knowledge
- `title` (string, optional): Node title
- `description` (string, optional): Node description
- Additional custom properties can be added as needed

**Indexes:**
- Unique index on `node_id` (global uniqueness)
- Hash index on `graph_id` for efficient graph-based queries

---

#### 3. **Edges Collection** (Edge Collection)
Stores relationships between knowledge nodes.

```json
{
  "_key": "edge_11111",
  "_id": "edges/edge_11111",
  "_from": "knowledge_nodes/node_67890",
  "_to": "knowledge_nodes/node_67891",
  "edge_id": "edge_11111",
  "graph_id": "graph_12345",
  "edge_type": "prerequisite",
  "created_at": "2025-01-01T00:00:00Z"
}
```

**Properties:**
- `edge_id` (string, unique): Unique identifier for the edge
- `graph_id` (string, indexed): Reference to parent graph
- `edge_type` (enum): Either "prerequisite" or "sub-topic"
  - **prerequisite**: Node A must be completed before Node B
  - **sub-topic**: Node B is a sub-topic of Node A
- `_from` (string): Source node reference
- `_to` (string): Target node reference

**Indexes:**
- Unique index on `edge_id`
- Hash index on `graph_id`
- Edge index on `_from` and `_to`

---

## Graph Isolation Strategy

To ensure strict isolation between graphs:

1. **Application-Level Enforcement**: All queries must include `graph_id` filter
2. **Validation**: Before any node/edge operation, verify that the entity belongs to the specified graph
3. **No Cross-Graph References**: Edges can only connect nodes within the same graph
4. **Cascade Deletion**: When a graph is deleted, all associated nodes and edges are removed

---

## API Specifications

### Base URL
```
http://localhost:8000/api/v1
```

### Authentication
*(To be implemented based on requirements)*

---

### 1. Create Graph

**Endpoint:** `POST /graphs`

**Description:** Creates a new knowledge graph for a course.

**Request Body:**
```json
{
  "course_id": "CS101"
}
```

**Response (201 Created):**
```json
{
  "graph_id": "graph_12345",
  "course_id": "CS101",
  "is_prototype": false,
  "created_at": "2025-01-01T00:00:00Z"
}
```

**Error Responses:**
- `400 Bad Request`: Invalid course_id format
- `409 Conflict`: Graph with this course_id already exists (cannot create new graph with existing course_id)

**Business Logic:**
- Check if any graph with this course_id already exists; if yes, throw 409 Conflict exception
- Generate unique graph_id
- Set is_prototype to false
- Create new graph document
- Return the newly created graph_id

**Note:** This API enforces course_id uniqueness. Multiple graphs with the same course_id can only exist through the Copy Graph API.

---

### 2. Delete Graph

**Endpoint:** `DELETE /graphs/{graph_id}`

**Description:** Deletes a graph and all its associated nodes and edges.

**Path Parameters:**
- `graph_id` (string): The unique identifier of the graph

**Response (200 OK):**
```json
{
  "graph_id": "graph_12345",
  "deleted_nodes_count": 25,
  "deleted_edges_count": 40
}
```

**Error Responses:**
- `404 Not Found`: Graph does not exist

**Business Logic:**
- Delete all edges associated with graph_id
- Delete all nodes associated with graph_id
- Delete the graph document
- Return graph_id and counts of deleted entities

---

### 3. Copy Graph

**Endpoint:** `POST /graphs/{graph_id}/copy`

**Description:** Clones an existing graph with all its nodes and edges.

**Path Parameters:**
- `graph_id` (string): The graph to be copied

**Response (201 Created):**
```json
{
  "graph_id": "graph_67890",
  "original_graph_id": "graph_12345",
  "course_id": "CS101",
  "is_prototype": false,
  "nodes_copied": 25,
  "edges_copied": 40
}
```

**Error Responses:**
- `404 Not Found`: Source graph does not exist

**Business Logic:**
- Generate new unique graph_id
- Copy graph metadata with new graph_id, keeping the SAME course_id as the original
- Set is_prototype to false (or copy from original)
- Clone all nodes with new node_ids, mapping old to new
- Clone all edges with new edge_ids, using the node_id mapping
- Return new graph_id

**Note:** This is the ONLY way to create multiple graphs with the same course_id. The copied graph inherits the course_id from the source graph, allowing versioning or multiple instances of the same course.

---

### 4. Create Node

**Endpoint:** `POST /graphs/{graph_id}/nodes`

**Description:** Creates a new knowledge node within a specific graph.

**Path Parameters:**
- `graph_id` (string): The graph to add the node to

**Request Body:**
```json
{
  "mastery_points": 100,
  "title": "Introduction to Variables",
  "description": "Learn about variables"
}
```

**Response (201 Created):**
```json
{
  "node_id": "node_67890",
  "graph_id": "graph_12345",
  "mastery_points": 100,
  "title": "Introduction to Variables",
  "description": "Learn about variables",
  "created_at": "2025-01-01T00:00:00Z"
}
```

**Error Responses:**
- `404 Not Found`: Graph does not exist

**Business Logic:**
- Verify graph exists
- Generate globally unique node_id
- Create node with graph_id reference
- Return node_id and full node object

---

### 5. Get Node

**Endpoint:** `GET /graphs/{graph_id}/nodes/{node_id}`

**Description:** Retrieves a specific node from a graph.

**Path Parameters:**
- `graph_id` (string): The graph containing the node
- `node_id` (string): The node to retrieve

**Response (200 OK):**
```json
{
  "node_id": "node_67890",
  "graph_id": "graph_12345",
  "mastery_points": 100,
  "title": "Introduction to Variables",
  "description": "Learn about variables",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

**Error Responses:**
- `404 Not Found`: Graph or node does not exist
- `403 Forbidden`: Node does not belong to specified graph

**Business Logic:**
- Verify node exists and belongs to the specified graph
- Return complete node object

---

### 6. Delete Node

**Endpoint:** `DELETE /graphs/{graph_id}/nodes/{node_id}`

**Description:** Deletes a node and all its connected edges.

**Path Parameters:**
- `graph_id` (string): The graph containing the node
- `node_id` (string): The node to delete

**Response (200 OK):**
```json
{
  "node_id": "node_67890",
  "deleted_edge_ids": ["edge_11111", "edge_22222", "edge_33333"]
}
```

**Error Responses:**
- `404 Not Found`: Graph or node does not exist
- `403 Forbidden`: Node does not belong to specified graph

**Business Logic:**
- Verify node exists and belongs to graph
- Find all edges connected to this node (both incoming and outgoing)
- Delete all connected edges
- Delete the node
- Return node_id and collection of deleted edge_ids

---

### 7. Modify Node

**Endpoint:** `PATCH /graphs/{graph_id}/nodes/{node_id}`

**Description:** Updates specific properties of a node.

**Path Parameters:**
- `graph_id` (string): The graph containing the node
- `node_id` (string): The node to modify

**Request Body:**
```json
{
  "mastery_points": 150,
  "title": "Advanced Variables"
}
```

**Response (200 OK):**
```json
{
  "node_id": "node_67890",
  "graph_id": "graph_12345",
  "mastery_points": 150,
  "title": "Advanced Variables",
  "description": "Learn about variables",
  "updated_at": "2025-01-02T00:00:00Z"
}
```

**Error Responses:**
- `404 Not Found`: Graph or node does not exist
- `403 Forbidden`: Node does not belong to specified graph
- `400 Bad Request`: Invalid property values

**Business Logic:**
- Verify node exists and belongs to graph
- Apply partial update to node properties
- Cannot modify node_id or graph_id
- Update updated_at timestamp
- Return complete updated node object

---

### 8. Create Edge

**Endpoint:** `POST /graphs/{graph_id}/edges`

**Description:** Creates an edge between two nodes in a graph.

**Path Parameters:**
- `graph_id` (string): The graph containing the nodes

**Request Body:**
```json
{
  "from_node_id": "node_67890",
  "to_node_id": "node_67891",
  "edge_type": "prerequisite"
}
```

**Response (201 Created):**
```json
{
  "edge_id": "edge_11111",
  "graph_id": "graph_12345",
  "from_node_id": "node_67890",
  "to_node_id": "node_67891",
  "edge_type": "prerequisite",
  "created_at": "2025-01-01T00:00:00Z"
}
```

**Error Responses:**
- `404 Not Found`: Graph or nodes do not exist
- `403 Forbidden`: Nodes do not belong to specified graph
- `400 Bad Request`: Invalid edge_type or nodes are the same
- `409 Conflict`: Edge already exists between these nodes

**Business Logic:**
- Verify both nodes exist and belong to the specified graph
- Validate edge_type is either "prerequisite" or "sub-topic"
- Generate unique edge_id
- Create edge connecting the two nodes
- Return edge_id and full edge object

---

### 9. Delete Edge

**Endpoint:** `DELETE /graphs/{graph_id}/edges`

**Description:** Deletes an edge between two nodes.

**Path Parameters:**
- `graph_id` (string): The graph containing the edge

**Request Body:**
```json
{
  "from_node_id": "node_67890",
  "to_node_id": "node_67891"
}
```

**Alternative:** `DELETE /graphs/{graph_id}/edges/{edge_id}`

**Response (200 OK):**
```json
{
  "edge_id": "edge_11111",
  "from_node_id": "node_67890",
  "to_node_id": "node_67891"
}
```

**Error Responses:**
- `404 Not Found`: Graph or edge does not exist
- `403 Forbidden`: Edge does not belong to specified graph
- `400 Bad Request`: Nodes are not connected

**Business Logic:**
- Verify edge exists between the two nodes in the specified graph
- If nodes are not connected, throw exception
- Delete the edge
- Return edge_id and node references

---

## Technology Stack

- **Backend Framework**: FastAPI (Python)
- **Database**: ArangoDB
- **Python Driver**: python-arango
- **API Documentation**: OpenAPI/Swagger (auto-generated by FastAPI)
- **Validation**: Pydantic models

---

## Error Handling

All errors follow this format:

```json
{
  "error": "error_code",
  "message": "Human-readable error message",
  "details": {}
}
```

**Standard HTTP Status Codes:**
- `200 OK`: Successful operation
- `201 Created`: Resource created successfully
- `400 Bad Request`: Invalid input
- `403 Forbidden`: Operation not allowed (e.g., cross-graph access)
- `404 Not Found`: Resource does not exist
- `409 Conflict`: Resource conflict (e.g., duplicate course_id)
- `500 Internal Server Error`: Server error

---

## Original Requirements

### Data Model Requirements

1. Construct a simple BE using Python, FastAPI and ArangoDB
2. The data model for the ArangoDB is described as follow
3. There is only one type of nodes for now: the knowledge node
4. Each of the graphs would have a property called course id. The course id is not unique. And we are able to efficiently query all the graphs that have the same course id. (index this property)
5. Each graph would have a boolean property named is_prototype
6. Each of the graphs are associated with a unique id number, and the graph can be retrieved by the id
7. Each of the graphs are separated from each other. A node associated with one graph is strictly prohibited from any interaction with a node associated with another graph
8. Each node would contain the property of mastery points, and a globally unique node id
9. In each graph, the nodes are connected with edges. There are two types of edges: the first one is prerequisite, and the second one is sub-topic. And each edge would have a unique id

### API Requirements

10. There are 11 types of API: create node, delete node, modify node, get node, create edge, delete edge, copy graph, delete graph, create graph *(Note: 9 unique APIs identified)*
11. create graph API would create a new graph, given the course id as input, set the is_prototype as false and return the unique graph id. Also check if the course id already exist, if so throw an exception
12. delete graph API would delete the graph, given the graph id as input, and return the graph id
13. copy graph API would clone an existing graph, given the existing graph id as input, and return id of the new graph
14. create node API would create a node in a specific graph, input is the graph id, and output is the node id
15. get node API takes graph id and node id as input, and returns the node object
16. delete node API takes graph id and node id as input. It deletes the input node, and all the edges connected to that node. It returns an object. The first property of that object is node id, and the second property is a collection of edge id
17. modify node API takes graph id and node id as input and a snap of node property that we want to change
18. create edge API takes graph id and node ids as input, returns the edge id
19. delete edge API takes graph id and two connected nodes as input, if the two nodes are not already connected, we throw an exception

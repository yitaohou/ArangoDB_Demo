# PostgreSQL Database Schema Design

## Overview
This document describes the PostgreSQL database schema for the knowledge graph management system. It provides **true database-level graph isolation** using row-level security policies.

---

## Database Schema

### **1. graphs Table**

Stores metadata for each knowledge graph.

```sql
CREATE TABLE graphs (
    id BIGSERIAL PRIMARY KEY,
    graph_id VARCHAR(255) UNIQUE NOT NULL,
    course_id VARCHAR(255) NOT NULL,
    is_prototype BOOLEAN DEFAULT FALSE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Indexes
CREATE UNIQUE INDEX idx_graphs_graph_id ON graphs(graph_id);
CREATE INDEX idx_graphs_course_id ON graphs(course_id);

-- Trigger to update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_graphs_updated_at 
    BEFORE UPDATE ON graphs
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Comments
COMMENT ON TABLE graphs IS 'Stores metadata for each knowledge graph';
COMMENT ON COLUMN graphs.graph_id IS 'Unique identifier for the graph (business key)';
COMMENT ON COLUMN graphs.course_id IS 'Course identifier (unique for new graphs, shared when copied)';
COMMENT ON COLUMN graphs.is_prototype IS 'Whether this is a prototype graph';
```

**Properties:**
- `id` (BIGSERIAL): Internal database primary key
- `graph_id` (VARCHAR, UNIQUE, INDEXED): Unique identifier for the graph
- `course_id` (VARCHAR, INDEXED): Course identifier (enforced unique on create, shared on copy)
- `is_prototype` (BOOLEAN): Whether this is a prototype graph
- `created_at` (TIMESTAMP): Creation timestamp
- `updated_at` (TIMESTAMP): Last modification timestamp (auto-updated)

---

### **2. knowledge_nodes Table**

Stores individual knowledge nodes within graphs.

```sql
CREATE TABLE knowledge_nodes (
    id BIGSERIAL PRIMARY KEY,
    node_id VARCHAR(255) UNIQUE NOT NULL,
    graph_id VARCHAR(255) NOT NULL,
    mastery_points INTEGER DEFAULT 0 NOT NULL,
    title VARCHAR(500),
    description TEXT,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    
    -- Foreign key constraint
    CONSTRAINT fk_nodes_graph FOREIGN KEY (graph_id) 
        REFERENCES graphs(graph_id) 
        ON DELETE CASCADE
);

-- Indexes
CREATE UNIQUE INDEX idx_nodes_node_id ON knowledge_nodes(node_id);
CREATE INDEX idx_nodes_graph_id ON knowledge_nodes(graph_id);
CREATE INDEX idx_nodes_mastery_points ON knowledge_nodes(mastery_points);

-- GIN index for JSONB metadata (for flexible querying)
CREATE INDEX idx_nodes_metadata ON knowledge_nodes USING GIN(metadata);

-- Trigger to update updated_at
CREATE TRIGGER update_nodes_updated_at 
    BEFORE UPDATE ON knowledge_nodes
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Comments
COMMENT ON TABLE knowledge_nodes IS 'Stores individual knowledge nodes within graphs';
COMMENT ON COLUMN knowledge_nodes.node_id IS 'Globally unique identifier for the node';
COMMENT ON COLUMN knowledge_nodes.graph_id IS 'Reference to parent graph (enforces isolation)';
COMMENT ON COLUMN knowledge_nodes.mastery_points IS 'Points associated with mastering this knowledge';
COMMENT ON COLUMN knowledge_nodes.metadata IS 'Flexible JSONB field for additional properties';
```

**Properties:**
- `id` (BIGSERIAL): Internal database primary key
- `node_id` (VARCHAR, UNIQUE, INDEXED): Globally unique identifier for the node
- `graph_id` (VARCHAR, INDEXED, FK): Reference to parent graph
- `mastery_points` (INTEGER): Points associated with mastering this knowledge
- `title` (VARCHAR): Node title (optional)
- `description` (TEXT): Node description (optional)
- `metadata` (JSONB): Flexible field for additional custom properties
- `created_at` (TIMESTAMP): Creation timestamp
- `updated_at` (TIMESTAMP): Last modification timestamp (auto-updated)

---

### **3. edges Table**

Stores relationships between knowledge nodes.

```sql
CREATE TYPE edge_type_enum AS ENUM ('prerequisite', 'sub-topic');

CREATE TABLE edges (
    id BIGSERIAL PRIMARY KEY,
    edge_id VARCHAR(255) UNIQUE NOT NULL,
    graph_id VARCHAR(255) NOT NULL,
    from_node_id VARCHAR(255) NOT NULL,
    to_node_id VARCHAR(255) NOT NULL,
    edge_type edge_type_enum NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    
    -- Foreign key constraints
    CONSTRAINT fk_edges_graph FOREIGN KEY (graph_id) 
        REFERENCES graphs(graph_id) 
        ON DELETE CASCADE,
    CONSTRAINT fk_edges_from_node FOREIGN KEY (from_node_id) 
        REFERENCES knowledge_nodes(node_id) 
        ON DELETE CASCADE,
    CONSTRAINT fk_edges_to_node FOREIGN KEY (to_node_id) 
        REFERENCES knowledge_nodes(node_id) 
        ON DELETE CASCADE,
    
    -- Prevent self-loops
    CONSTRAINT chk_no_self_loop CHECK (from_node_id != to_node_id),
    
    -- Ensure nodes belong to same graph (enforced at application level too)
    CONSTRAINT chk_same_graph CHECK (
        graph_id = (SELECT graph_id FROM knowledge_nodes WHERE node_id = from_node_id)
        AND graph_id = (SELECT graph_id FROM knowledge_nodes WHERE node_id = to_node_id)
    )
);

-- Indexes
CREATE UNIQUE INDEX idx_edges_edge_id ON edges(edge_id);
CREATE INDEX idx_edges_graph_id ON edges(graph_id);
CREATE INDEX idx_edges_from_node ON edges(from_node_id);
CREATE INDEX idx_edges_to_node ON edges(to_node_id);
CREATE INDEX idx_edges_type ON edges(edge_type);

-- Composite index for edge lookups
CREATE UNIQUE INDEX idx_edges_unique_direction ON edges(graph_id, from_node_id, to_node_id, edge_type);

-- GIN index for JSONB metadata
CREATE INDEX idx_edges_metadata ON edges USING GIN(metadata);

-- Comments
COMMENT ON TABLE edges IS 'Stores relationships between knowledge nodes';
COMMENT ON COLUMN edges.edge_id IS 'Unique identifier for the edge';
COMMENT ON COLUMN edges.graph_id IS 'Reference to parent graph (enforces isolation)';
COMMENT ON COLUMN edges.edge_type IS 'Type of relationship: prerequisite or sub-topic';
COMMENT ON COLUMN edges.from_node_id IS 'Source node of the edge';
COMMENT ON COLUMN edges.to_node_id IS 'Target node of the edge';
```

**Properties:**
- `id` (BIGSERIAL): Internal database primary key
- `edge_id` (VARCHAR, UNIQUE, INDEXED): Unique identifier for the edge
- `graph_id` (VARCHAR, INDEXED, FK): Reference to parent graph
- `from_node_id` (VARCHAR, INDEXED, FK): Source node reference
- `to_node_id` (VARCHAR, INDEXED, FK): Target node reference
- `edge_type` (ENUM): Either "prerequisite" or "sub-topic"
- `metadata` (JSONB): Flexible field for additional properties
- `created_at` (TIMESTAMP): Creation timestamp

**Constraints:**
- No self-loops (node can't connect to itself)
- Both nodes must belong to the same graph
- Unique combination of (graph_id, from_node_id, to_node_id, edge_type)

---

## Graph Isolation Implementation

### **Row-Level Security (RLS)**

PostgreSQL's row-level security provides **true database-level isolation**.

```sql
-- Enable row-level security
ALTER TABLE knowledge_nodes ENABLE ROW LEVEL SECURITY;
ALTER TABLE edges ENABLE ROW LEVEL SECURITY;

-- Create policy for nodes (example - adjust based on your auth)
CREATE POLICY nodes_isolation_policy ON knowledge_nodes
    USING (
        graph_id = current_setting('app.current_graph_id', true)
        OR current_setting('app.bypass_rls', true) = 'true'
    );

-- Create policy for edges
CREATE POLICY edges_isolation_policy ON edges
    USING (
        graph_id = current_setting('app.current_graph_id', true)
        OR current_setting('app.bypass_rls', true) = 'true'
    );

-- Grant permissions (adjust based on your needs)
GRANT SELECT, INSERT, UPDATE, DELETE ON knowledge_nodes TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON edges TO app_user;
```

**Usage in Application:**

```python
# Set graph context before queries
await conn.execute("SET app.current_graph_id = $1", graph_id)

# Now all queries automatically filter by graph_id
nodes = await conn.fetch("SELECT * FROM knowledge_nodes")
# This only returns nodes from the current graph!

# For operations that need cross-graph access (e.g., copy graph)
await conn.execute("SET app.bypass_rls = 'true'")
# Perform cross-graph operation
await conn.execute("SET app.bypass_rls = 'false'")
```

**Benefits:**
- ✅ Impossible to accidentally query wrong graph
- ✅ Database enforces isolation, not application code
- ✅ Can still bypass for admin operations
- ✅ No changes needed to application queries

---

## Graph Traversal Queries

### **Example 1: Find Direct Prerequisites**

```sql
SELECT 
    n.node_id,
    n.title,
    n.mastery_points,
    e.edge_type
FROM knowledge_nodes n
INNER JOIN edges e ON e.to_node_id = n.node_id
WHERE e.from_node_id = 'start_node'
  AND e.graph_id = 'graph_123'
  AND e.edge_type = 'prerequisite';
```

---

### **Example 2: Find All Prerequisites (Multi-level)**

```sql
WITH RECURSIVE prerequisites AS (
    -- Base case: start node
    SELECT 
        node_id,
        title,
        mastery_points,
        0 as depth,
        ARRAY[node_id] as path
    FROM knowledge_nodes
    WHERE node_id = 'start_node'
      AND graph_id = 'graph_123'
    
    UNION ALL
    
    -- Recursive case: follow prerequisite edges
    SELECT 
        n.node_id,
        n.title,
        n.mastery_points,
        p.depth + 1,
        p.path || n.node_id
    FROM knowledge_nodes n
    INNER JOIN edges e ON e.to_node_id = n.node_id
    INNER JOIN prerequisites p ON e.from_node_id = p.node_id
    WHERE e.graph_id = 'graph_123'
      AND e.edge_type = 'prerequisite'
      AND p.depth < 10  -- Prevent infinite loops
      AND NOT n.node_id = ANY(p.path)  -- Detect cycles
)
SELECT * FROM prerequisites
ORDER BY depth, node_id;
```

---

### **Example 3: Find All Sub-topics**

```sql
WITH RECURSIVE subtopics AS (
    -- Base case
    SELECT 
        node_id,
        title,
        0 as level,
        CAST(title AS VARCHAR(1000)) as path_string
    FROM knowledge_nodes
    WHERE node_id = 'root_topic'
      AND graph_id = 'graph_123'
    
    UNION ALL
    
    -- Recursive case: follow sub-topic edges
    SELECT 
        n.node_id,
        n.title,
        s.level + 1,
        s.path_string || ' > ' || n.title
    FROM knowledge_nodes n
    INNER JOIN edges e ON e.to_node_id = n.node_id
    INNER JOIN subtopics s ON e.from_node_id = s.node_id
    WHERE e.graph_id = 'graph_123'
      AND e.edge_type = 'sub-topic'
      AND s.level < 10
)
SELECT * FROM subtopics
ORDER BY level, path_string;
```

---

### **Example 4: Find All Connected Nodes (Both Directions)**

```sql
WITH RECURSIVE connected_nodes AS (
    -- Base case
    SELECT 
        node_id,
        title,
        0 as distance
    FROM knowledge_nodes
    WHERE node_id = 'start_node'
      AND graph_id = 'graph_123'
    
    UNION
    
    -- Follow edges in both directions
    SELECT 
        n.node_id,
        n.title,
        c.distance + 1
    FROM knowledge_nodes n
    INNER JOIN edges e ON (
        (e.from_node_id = n.node_id OR e.to_node_id = n.node_id)
    )
    INNER JOIN connected_nodes c ON (
        e.from_node_id = c.node_id OR e.to_node_id = c.node_id
    )
    WHERE e.graph_id = 'graph_123'
      AND n.node_id != c.node_id
      AND c.distance < 10
)
SELECT DISTINCT * FROM connected_nodes
ORDER BY distance, node_id;
```

---

### **Example 5: Check for Cycles**

```sql
WITH RECURSIVE cycle_check AS (
    SELECT 
        node_id,
        ARRAY[node_id] as path,
        false as has_cycle
    FROM knowledge_nodes
    WHERE graph_id = 'graph_123'
    
    UNION ALL
    
    SELECT 
        n.node_id,
        c.path || n.node_id,
        n.node_id = ANY(c.path) as has_cycle
    FROM knowledge_nodes n
    INNER JOIN edges e ON e.to_node_id = n.node_id
    INNER JOIN cycle_check c ON e.from_node_id = c.node_id
    WHERE e.graph_id = 'graph_123'
      AND NOT c.has_cycle
      AND array_length(c.path, 1) < 100
)
SELECT DISTINCT 
    path,
    has_cycle
FROM cycle_check
WHERE has_cycle = true;
```

---

## Helper Functions

### **Function: Copy Graph**

```sql
CREATE OR REPLACE FUNCTION copy_graph(
    source_graph_id VARCHAR,
    new_graph_id VARCHAR
) RETURNS TABLE (
    new_graph_id VARCHAR,
    nodes_copied INTEGER,
    edges_copied INTEGER
) AS $$
DECLARE
    nodes_count INTEGER;
    edges_count INTEGER;
BEGIN
    -- Copy graph metadata (keeping same course_id)
    INSERT INTO graphs (graph_id, course_id, is_prototype)
    SELECT new_graph_id, course_id, is_prototype
    FROM graphs
    WHERE graph_id = source_graph_id;
    
    -- Copy nodes with new node_ids
    WITH copied_nodes AS (
        INSERT INTO knowledge_nodes (
            node_id, 
            graph_id, 
            mastery_points, 
            title, 
            description, 
            metadata
        )
        SELECT 
            new_graph_id || '_' || node_id,  -- Generate new node_id
            new_graph_id,
            mastery_points,
            title,
            description,
            metadata
        FROM knowledge_nodes
        WHERE graph_id = source_graph_id
        RETURNING 1
    )
    SELECT COUNT(*) INTO nodes_count FROM copied_nodes;
    
    -- Copy edges with new edge_ids and mapped node references
    WITH copied_edges AS (
        INSERT INTO edges (
            edge_id,
            graph_id,
            from_node_id,
            to_node_id,
            edge_type,
            metadata
        )
        SELECT 
            new_graph_id || '_' || edge_id,  -- Generate new edge_id
            new_graph_id,
            new_graph_id || '_' || from_node_id,  -- Map to new node_id
            new_graph_id || '_' || to_node_id,    -- Map to new node_id
            edge_type,
            metadata
        FROM edges
        WHERE graph_id = source_graph_id
        RETURNING 1
    )
    SELECT COUNT(*) INTO edges_count FROM copied_edges;
    
    -- Return results
    RETURN QUERY 
    SELECT 
        new_graph_id AS new_graph_id,
        nodes_count AS nodes_copied,
        edges_count AS edges_copied;
END;
$$ LANGUAGE plpgsql;

-- Usage:
-- SELECT * FROM copy_graph('graph_123', 'graph_456');
```

---

### **Function: Delete Graph (Cascade)**

```sql
CREATE OR REPLACE FUNCTION delete_graph(
    target_graph_id VARCHAR
) RETURNS TABLE (
    graph_id VARCHAR,
    nodes_deleted INTEGER,
    edges_deleted INTEGER
) AS $$
DECLARE
    nodes_count INTEGER;
    edges_count INTEGER;
BEGIN
    -- Count before deletion
    SELECT COUNT(*) INTO edges_count 
    FROM edges WHERE edges.graph_id = target_graph_id;
    
    SELECT COUNT(*) INTO nodes_count 
    FROM knowledge_nodes WHERE knowledge_nodes.graph_id = target_graph_id;
    
    -- Delete edges (cascade will handle this, but explicit for clarity)
    DELETE FROM edges WHERE edges.graph_id = target_graph_id;
    
    -- Delete nodes (cascade will handle edges too)
    DELETE FROM knowledge_nodes WHERE knowledge_nodes.graph_id = target_graph_id;
    
    -- Delete graph
    DELETE FROM graphs WHERE graphs.graph_id = target_graph_id;
    
    -- Return results
    RETURN QUERY 
    SELECT 
        target_graph_id AS graph_id,
        nodes_count AS nodes_deleted,
        edges_count AS edges_deleted;
END;
$$ LANGUAGE plpgsql;

-- Usage:
-- SELECT * FROM delete_graph('graph_123');
```

---

### **Function: Get Node with Connections**

```sql
CREATE OR REPLACE FUNCTION get_node_with_connections(
    target_node_id VARCHAR,
    target_graph_id VARCHAR
) RETURNS TABLE (
    node JSONB,
    incoming_edges JSONB,
    outgoing_edges JSONB
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        to_jsonb(n.*) as node,
        COALESCE(
            (SELECT jsonb_agg(to_jsonb(e.*)) 
             FROM edges e 
             WHERE e.to_node_id = target_node_id 
               AND e.graph_id = target_graph_id),
            '[]'::jsonb
        ) as incoming_edges,
        COALESCE(
            (SELECT jsonb_agg(to_jsonb(e.*)) 
             FROM edges e 
             WHERE e.from_node_id = target_node_id 
               AND e.graph_id = target_graph_id),
            '[]'::jsonb
        ) as outgoing_edges
    FROM knowledge_nodes n
    WHERE n.node_id = target_node_id
      AND n.graph_id = target_graph_id;
END;
$$ LANGUAGE plpgsql;

-- Usage:
-- SELECT * FROM get_node_with_connections('node_123', 'graph_456');
```

---

## Views for Common Queries

### **View: Graph Statistics**

```sql
CREATE OR REPLACE VIEW graph_statistics AS
SELECT 
    g.graph_id,
    g.course_id,
    g.is_prototype,
    COUNT(DISTINCT n.node_id) as node_count,
    COUNT(DISTINCT e.edge_id) as edge_count,
    COUNT(DISTINCT CASE WHEN e.edge_type = 'prerequisite' THEN e.edge_id END) as prerequisite_count,
    COUNT(DISTINCT CASE WHEN e.edge_type = 'sub-topic' THEN e.edge_id END) as subtopic_count,
    AVG(n.mastery_points) as avg_mastery_points,
    g.created_at,
    g.updated_at
FROM graphs g
LEFT JOIN knowledge_nodes n ON n.graph_id = g.graph_id
LEFT JOIN edges e ON e.graph_id = g.graph_id
GROUP BY g.graph_id, g.course_id, g.is_prototype, g.created_at, g.updated_at;

-- Usage:
-- SELECT * FROM graph_statistics WHERE course_id = 'CS101';
```

---

### **View: Node Degree (In/Out connections)**

```sql
CREATE OR REPLACE VIEW node_degree AS
SELECT 
    n.graph_id,
    n.node_id,
    n.title,
    COUNT(DISTINCT e_in.edge_id) as in_degree,
    COUNT(DISTINCT e_out.edge_id) as out_degree,
    COUNT(DISTINCT e_in.edge_id) + COUNT(DISTINCT e_out.edge_id) as total_degree
FROM knowledge_nodes n
LEFT JOIN edges e_in ON e_in.to_node_id = n.node_id AND e_in.graph_id = n.graph_id
LEFT JOIN edges e_out ON e_out.from_node_id = n.node_id AND e_out.graph_id = n.graph_id
GROUP BY n.graph_id, n.node_id, n.title;

-- Usage:
-- SELECT * FROM node_degree WHERE graph_id = 'graph_123' ORDER BY total_degree DESC;
```

---

## Indexes Summary

| Table | Index Type | Columns | Purpose |
|-------|-----------|---------|---------|
| graphs | UNIQUE | graph_id | Primary lookup key |
| graphs | B-tree | course_id | Query all graphs for a course |
| knowledge_nodes | UNIQUE | node_id | Primary lookup key |
| knowledge_nodes | B-tree | graph_id | Isolation and filtering |
| knowledge_nodes | B-tree | mastery_points | Range queries |
| knowledge_nodes | GIN | metadata | JSONB queries |
| edges | UNIQUE | edge_id | Primary lookup key |
| edges | B-tree | graph_id | Isolation and filtering |
| edges | B-tree | from_node_id | Traversal queries |
| edges | B-tree | to_node_id | Traversal queries |
| edges | B-tree | edge_type | Filter by relationship type |
| edges | UNIQUE | (graph_id, from_node_id, to_node_id, edge_type) | Prevent duplicates |
| edges | GIN | metadata | JSONB queries |

---

## Performance Considerations

### **Query Optimization Tips:**

1. **Always include graph_id in WHERE clause** (even with RLS, explicit helps planner)
2. **Limit recursion depth** in CTEs to prevent runaway queries
3. **Use path arrays** to detect cycles in recursive queries
4. **Add indexes** on frequently queried metadata fields
5. **Use EXPLAIN ANALYZE** to check query plans

### **Expected Performance:**

| Query Type | Nodes | Edges | Expected Time |
|------------|-------|-------|---------------|
| Single node lookup | Any | Any | < 5ms |
| 1-hop traversal | 10K | 50K | 10-50ms |
| 2-3 hop traversal | 10K | 50K | 50-200ms |
| 5+ hop traversal | 10K | 50K | 200-500ms |
| Graph copy | 1K | 5K | 100-300ms |

---

## Migration from Neo4j (If Applicable)

### **Export from Neo4j:**

```cypher
// Export graphs
MATCH (g:Graph)
RETURN g.graph_id, g.course_id, g.is_prototype

// Export nodes
MATCH (n:KnowledgeNode)
RETURN n.node_id, n.graph_id, n.mastery_points, n.title, n.description

// Export edges
MATCH (n1)-[e:PREREQUISITE|SUB_TOPIC]->(n2)
RETURN e.edge_id, e.graph_id, n1.node_id as from_node_id, 
       n2.node_id as to_node_id, type(e) as edge_type
```

### **Import to PostgreSQL:**

```python
import psycopg
import csv

# Connect to PostgreSQL
conn = psycopg.connect("dbname=knowledge_graph user=postgres")

# Import graphs
with open('graphs.csv', 'r') as f:
    with conn.cursor() as cur:
        cur.copy_expert("""
            COPY graphs (graph_id, course_id, is_prototype) 
            FROM STDIN WITH CSV HEADER
        """, f)

# Import nodes
with open('nodes.csv', 'r') as f:
    with conn.cursor() as cur:
        cur.copy_expert("""
            COPY knowledge_nodes (node_id, graph_id, mastery_points, title, description) 
            FROM STDIN WITH CSV HEADER
        """, f)

# Import edges
with open('edges.csv', 'r') as f:
    with conn.cursor() as cur:
        cur.copy_expert("""
            COPY edges (edge_id, graph_id, from_node_id, to_node_id, edge_type) 
            FROM STDIN WITH CSV HEADER
        """, f)

conn.commit()
```

---

## Original Requirements Mapping

| Requirement | PostgreSQL Implementation |
|------------|---------------------------|
| 1. Python, FastAPI, Database | ✅ FastAPI + asyncpg/psycopg |
| 3. Knowledge nodes only | ✅ knowledge_nodes table |
| 4. course_id indexed, non-unique | ✅ Indexed, allows copies |
| 5. is_prototype boolean | ✅ In graphs table |
| 6. Unique graph_id | ✅ Unique constraint + index |
| 7. Graph isolation | ✅ Row-level security + FK constraints |
| 8. Mastery points + unique node_id | ✅ In knowledge_nodes table |
| 9. Two edge types with unique edge_id | ✅ ENUM type + unique constraint |
| 11. Create graph (check course_id) | ✅ Application logic + unique check |
| 12. Delete graph | ✅ delete_graph() function |
| 13. Copy graph | ✅ copy_graph() function |
| 14-19. CRUD APIs | ✅ Standard SQL operations |

---

## Next Steps

1. **Create database schema** - Run all CREATE TABLE statements
2. **Set up row-level security** - Configure RLS policies
3. **Create helper functions** - Install copy_graph, delete_graph, etc.
4. **Test graph queries** - Verify recursive CTEs work as expected
5. **Benchmark performance** - Test with realistic data volumes
6. **Implement FastAPI endpoints** - Connect Python to PostgreSQL

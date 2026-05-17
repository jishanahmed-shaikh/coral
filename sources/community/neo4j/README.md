# Neo4j

Query nodes, relationships, indexes, and constraints from
[Neo4j](https://neo4j.com/) graph databases using SQL via the
[HTTP transactional API](https://neo4j.com/docs/http-api/current/).

## How it works

Neo4j stores data as a property graph — nodes connected by typed relationships.
This source sends Cypher queries to the Neo4j HTTP transactional API
(`/db/neo4j/tx/commit`) and maps the results into flat SQL-queryable tables.
Each node label gets its own table. Relationships are exposed as a single
`relationships` table with `from_label`, `rel_type`, and `to_label` columns.
Schema metadata (indexes, constraints, labels, relationship types) is exposed
as dedicated tables.

## Authentication

Neo4j uses HTTP Basic Auth. Set `NEO4J_USERNAME` and `NEO4J_PASSWORD` to match
your instance credentials. The default credentials for a fresh Docker instance
are `neo4j` / `password` (set via `NEO4J_AUTH=neo4j/password`).

## Local Setup

```bash
docker run -d \
  --name neo4j \
  -p 7474:7474 \
  -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password \
  neo4j:latest
```

Neo4j Browser will be available at `http://localhost:7474`.
Log in with username `neo4j` and password `password`.

### 1. Create mock data

Run the following in the Neo4j Browser query editor to create sample nodes
and relationships for testing:

```cypher
CREATE
  (u1:User {id: 1, name: 'JARS', email: 'jars@example.com'}),
  (u2:User {id: 2, name: 'Alex', email: 'alex@example.com'}),
  (u3:User {id: 3, name: 'Sarah', email: 'sarah@example.com'}),
  (o1:Organization {id: 101, name: 'OpenAI'}),
  (o2:Organization {id: 102, name: 'NeoTech'}),
  (p1:Product {id: 201, name: 'RTX 5090', category: 'GPU'}),
  (p2:Product {id: 202, name: 'MacBook Pro', category: 'Laptop'}),
  (p3:Product {id: 203, name: 'Quest 4', category: 'VR'}),
  (ord1:Order {id: 301, total: 4500}),
  (ord2:Order {id: 302, total: 2200}),
  (t1:Technology {name: 'AI'}),
  (t2:Technology {name: 'Cloud'}),
  (t3:Technology {name: 'GraphDB'}),
  (u1)-[:WORKS_AT]->(o1),
  (u2)-[:WORKS_AT]->(o2),
  (u3)-[:WORKS_AT]->(o1),
  (u1)-[:INTERESTED_IN]->(t1),
  (u1)-[:INTERESTED_IN]->(t2),
  (u2)-[:INTERESTED_IN]->(t3),
  (u1)-[:PLACED]->(ord1),
  (u2)-[:PLACED]->(ord2),
  (ord1)-[:CONTAINS]->(p1),
  (ord1)-[:CONTAINS]->(p3),
  (ord2)-[:CONTAINS]->(p2),
  (o1)-[:USES]->(t1),
  (o1)-[:USES]->(t2),
  (o2)-[:USES]->(t3);
```

### 2. Add indexes

```cypher
CREATE INDEX user_email_index FOR (u:User) ON (u.email);
CREATE INDEX product_name_index FOR (p:Product) ON (p.name);
CREATE INDEX organization_name_index FOR (o:Organization) ON (o.name);
```

### 3. Add constraints

```cypher
CREATE CONSTRAINT user_id_unique IF NOT EXISTS
FOR (u:User)
REQUIRE u.id IS UNIQUE;

CREATE CONSTRAINT product_id_unique IF NOT EXISTS
FOR (p:Product)
REQUIRE p.id IS UNIQUE;

CREATE CONSTRAINT organization_id_unique IF NOT EXISTS
FOR (o:Organization)
REQUIRE o.id IS UNIQUE;
```

### 4. Verify metadata

Run these in the Neo4j Browser to confirm everything was created correctly:

```cypher
-- Labels
CALL db.labels();

-- Relationship types
CALL db.relationshipTypes();

-- Indexes
SHOW INDEXES;

-- Constraints
SHOW CONSTRAINTS;
```

### 5. Visual graph query (Neo4j Browser only)

```cypher
MATCH (n)-[r]->(m)
RETURN n, r, m
LIMIT 100
```

## Configuration

| Input            | Kind     | Required | Description                                                              |
|------------------|----------|----------|--------------------------------------------------------------------------|
| `NEO4J_URL`      | variable | yes      | Base URL of the Neo4j HTTP interface, e.g. `http://localhost:7474`       |
| `NEO4J_USERNAME` | variable | yes      | Neo4j username (default: `neo4j`)                                        |
| `NEO4J_PASSWORD` | secret   | yes      | Neo4j password (set via `NEO4J_AUTH=neo4j/<password>` in Docker)         |

## Schema

### `node_labels`

One row per node label defined in the database. Start here to discover what
node types exist before querying individual node tables.

### `relationship_types`

One row per relationship type. Use these values to filter the `relationships`
table by `rel_type`.

### `users`

All nodes with the `User` label. Fields: `id`, `name`, `email`.

### `organizations`

All nodes with the `Organization` label. Fields: `id`, `name`.

### `products`

All nodes with the `Product` label. Fields: `id`, `name`, `category`.

### `orders`

All nodes with the `Order` label. Fields: `id`, `total`.

### `technologies`

All nodes with the `Technology` label. Fields: `name`.

### `relationships`

All relationships in the graph with source and target node info. Fields:
`from_label`, `from_id`, `rel_type`, `to_label`, `to_id`. Use this table to
traverse the graph topology in SQL. Filter by `rel_type` in a WHERE clause
to scope results (e.g. `WHERE rel_type = 'WORKS_AT'`).

### `indexes`

All indexes defined in the database. Fields: `name`, `type`, `state`,
`entity_type`, `labels_or_types`, `properties`.

### `constraints`

All constraints defined in the database. Fields: `name`, `type`,
`entity_type`, `labels_or_types`, `properties`.

## Example Queries

```sql
-- Discover all node labels in the graph
SELECT label FROM neo4j.node_labels;

-- Discover all relationship types
SELECT relationship_type FROM neo4j.relationship_types;

-- List all users
SELECT id, name, email FROM neo4j.users;

-- List all products by category
SELECT name, category FROM neo4j.products ORDER BY category, name;

-- See which users work at which organizations
SELECT r.from_id AS user_id, u.name AS user_name, o.name AS org_name
FROM neo4j.relationships r
JOIN neo4j.users u ON r.from_id = CAST(u.id AS VARCHAR)
JOIN neo4j.organizations o ON r.to_id = CAST(o.id AS VARCHAR)
WHERE r.rel_type = 'WORKS_AT';

-- See which orders contain which products
SELECT r.from_id AS order_id, p.name AS product_name, p.category
FROM neo4j.relationships r
JOIN neo4j.products p ON r.to_id = CAST(p.id AS VARCHAR)
WHERE r.rel_type = 'CONTAINS';

-- Count relationships by type
SELECT rel_type, COUNT(*) AS count
FROM neo4j.relationships
GROUP BY rel_type
ORDER BY count DESC;

-- List all ONLINE indexes
SELECT name, type, entity_type, labels_or_types, properties
FROM neo4j.indexes
WHERE state = 'ONLINE';

-- List all uniqueness constraints
SELECT name, labels_or_types, properties
FROM neo4j.constraints
WHERE type = 'NODE_PROPERTY_UNIQUENESS';
```

## Notes

- The `relationships` table returns all relationships in the graph. For very
  large graphs, add a `WHERE rel_type = '...'` clause to scope results.
- The `from_id` and `to_id` columns in `relationships` are coerced to strings
  using `coalesce(toString(node.id), node.name)`. Numeric IDs are cast to
  string for uniform comparison.
- The `labels_or_types` and `properties` columns in `indexes` and `constraints`
  are JSON arrays (e.g. `["User"]`, `["id"]`).
- This source targets the `neo4j` database. For a different database, update
  the path in the manifest from `/db/neo4j/tx/commit` to
  `/db/<your-database>/tx/commit`.

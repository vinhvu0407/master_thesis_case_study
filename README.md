# Tutorial: Building a Domain aware Event Knowledge graph with Neo4j and Cypher

This tutorial shows you to the basics of building domain aware event knowledge graphs, consisting of event tables and knowledge graphs, using the database system Neo4j and the Cypher query language. 

## 1. Create an event knowledge graph with the constructed event table T.

The foundation of building the event knowledge graph is taken from the following GitHub page: https://github.com/multi-dimensional-process-mining/eventgraph_tutorial/blob/main/order_process/tutorial-your-first-event-knowledge-graph.md

### 1.1 Importing events

By opening the Neo4j browser or console we entered the following Cypher query in order to load our prepared event table T into Neo4j:

```
LOAD CSV WITH HEADERS FROM "file:///eventgraph_tutorial/order_process/prepared_logs/order_management_log_event_table_prepared.csv" as line CREATE (e:Event {Log: "order_process",EventID: line.EventID, Activity: line.Activity, timestamp: datetime(line.timestamp), Customer: line.Customer, Order: line.Order, Package: line.Package, Product: line.Product, Item: line.Item, Price: toFloat(line.Price), Weight: toFloat(line.Weight)})
```
We can also perform property-specific queries rather than querying entire nodes to verify, if the data has been accurately loaded into Neo4j.
The following Cypher query accomplishes this:

```
MATCH (e:Event) RETURN e.Customer,e.Order,e.Package,e.Product,e.Item,e.Price,e.Weight
```

In a next step we have to split the strings stored in the Item property at the delimiter (’,’), creating a list of strings. This list will then replace the current value of e.Item and e.Product. The
following queries accomplishes this task for the property values of Item and Products:

```
MATCH (e:Event) WHERE e.Item <> "null" WITH e,split(e.Item, ',') AS items SET e.Item=items

MATCH (e:Event) WHERE e.Product <> "null" WITH e,split(e.Product, ',') AS vals SET e.Product=vals
```

### 1.2 Inferring entities

After the previous steps, our graph contains isolated :Event nodes with various properties. To connect these events, we identify their relationships with entities. We will create new nodes and rela-
tionships to represent these connections, utilizing certain properties as identifiers for specific entity types.

Create entity nodes - We now want to explicitly represent the events, that pertain to the same entity. To achieve this, we create a distinct :Entity
node for each unique entity identifier value. Using the subsequent query, we generate a unique :Entity node for every identifier value associated with Order, each possessing a specific ID:

```
MATCH (e:Event) UNWIND e.Order AS id_val
WITH DISTINCT id_val
CREATE (:Entity {ID:id_val, EntityType:"Order"})
```

This creates three new :Entity nodes, one for id val=O1, one for id val=O2 and one for id val=O3. We can now also apply the query above to deduce entity nodes for all the
other entity types, including Customer, Package and Item.

Correlate events to entity nodes - To explicitly represent the relationships between events and entities, we introduce correlation relationships (label :CORR). The subsequent query establishes a :CORR relationship from any :Event, that references an Order value with an id to the :Entity with an EntityType of Order and the same id:


```
MATCH (e:Event) UNWIND e.Order AS id_val WITH e,id_val
MATCH (n:Entity {EntityType: "Order"}) WHERE id_val = n.ID
MERGE (e)-[:CORR]->(n)
```

This query is expected to generate 30 :CORR relationships. With this, we have established the initial graph structure that offers valuable querying capabilities.

Next, we extend this analysis by adding :CORR relationships for the remaining entity types: Customer, Package and Item:

```
MATCH (e:Event) UNWIND e.Customer AS id_val WITH e,id_val
MATCH (n:Entity {EntityType: "Customer"}) WHERE id_val = n.ID
MERGE (e)-[:CORR]->(n)

MATCH (e:Event) UNWIND e.Package AS id_val WITH e,id_val
MATCH (n:Entity {EntityType: "Package"}) WHERE id_val = n.ID
MERGE (e)-[:CORR]->(n)

MATCH (e:Event) UNWIND e.Item AS id_val WITH e,id_val
MATCH (n:Entity {EntityType: "Item"}) WHERE id_val = n.ID
MERGE (e)-[:CORR]->(n)
```

### 1.3 Inferring temporal relations

Up to this point, the graph has outlined the basic structure, showing event-entity correlations. Now, we aim to incorporate chronological information explicitly. To achieve this, we
follow two steps: 1) Arrange entity-specific events chronologically based on timestamps and 2) introduce directly-follows relationships (:DF) between
consecutive events. We can infer temporal relations for all events, using the following query:

```
MATCH (n:Entity)
MATCH (n)<-[:CORR]-(e)
WITH n, e AS nodes ORDER BY e.timestamp, ID(e)
WITH n, collect(nodes) AS event_node_list
UNWIND range(0, size(event_node_list)-2) AS i
WITH n, event_node_list[i] AS e1, event_node_list[i+1] AS e2
MERGE (e1)-[df:DF {EntityType:n.EntityType, ID:n.ID}]->(e2)
```

Now we have constructed our Event Knowledge Graph.


## 2. Construct a knowledge graph representing domain knowledge

The queries below demonstrate the creation of our knowledge graph through two separate CREATE statements. The first statement establishes nodes and
their node labels, while the second statement defines relationship types and their respective labels.

```
CREATE (customer:Customers {EntityType: 'Customers'}),
       (custtype:CustomerType {EntityType: 'Customer Type'}),
       (marco:Customer {id2: 'C1', name: 'Marco Pegoraro', age: 22, gender: 'male'}),
       (christine:Customer {id2: 'C2', name: 'Christine Dobbert', age: 36, gender: 'female'}),
       (luis:Customer {id2: 'C3', name: 'Luis Santos', age: 18, gender: 'male'}),
       (germany:Country {Region: 'Europe', Country: 'Germany'}),
       (belgium:Country {Region: 'Europe', Country: 'Belgium'}),
       (shipping:Shipping {EntityType: 'Shipping'}),
       (vehicle:Vehicle {EntityType: 'Vehicle'}),
       (truck1:Truck {id2: 'Truck1', operating_days: ['monday', 'wednesday', 'friday'], operating_time: '9-21'}),
       (truck2:Truck {id2: 'Truck2', operating_days: ['monday', 'tuesday', 'wednesday', 'thursday', 'friday'], operating_time: '9-21'}),
       (packages:Packages {EntityType: 'Packages'}),
       (spack:Package {id2: 'Size: S', weightrange: [0.00, 1.00], quantity: 3}),
       (mpack:Package {id2: 'Size: M', weightrange: [1.01, 2.00], quantity: 2}),
       (lpack:Package {id2: 'Size: L', weightrange: [2.01, 5.00], quantity: 5})

CREATE (customer)-[:HAS_CATEGORY]->(custtype),
       (marco)-[:HAS_TYPE {customertype: 'basic'}]->(custtype),
       (christine)-[:HAS_TYPE {customertype: 'premium'}]->(custtype),
       (luis)-[:HAS_TYPE {customertype: 'regular'}]->(custtype),
       (marco)-[:LIVES_IN]->(germany),
       (christine)-[:LIVES_IN]->(germany),
       (luis)-[:LIVES_IN]->(belgium),
       (truck1)-[:OPERATES_IN]->(germany),
       (truck2)-[:OPERATES_IN]->(belgium),
       (vehicle)-[:CONTAINS]->(truck1),
       (vehicle)-[:CONTAINS]->(truck2),
       (vehicle)-[:PART_OF]->(shipping),
       (packages)-[:PART_OF]->(shipping),
       (packages)-[:HAS_SIZE]->(spack),
       (packages)-[:HAS_SIZE]->(mpack),
       (packages)-[:HAS_SIZE]->(lpack);
```

The graph captures relationships such as customer-type associations and customer residency locations containing attribute-value pairs for nodes and
relationships. Furthermore, it tracks vehicle operations in different countries and includes package details, such as size and quantity. This knowledge graph
offers a structured and interconnected representation of data related to these nodes, facilitating queries and analysis of their relationships and attributes.


## 3. Connecting the event knowledge graph and the knowledge graph forming a domain aware event knowledge graph (DAEKG).

We can now connect these two graphs to form a unified data model. This model combines the event knowledge graph, representing data from an event table with multiple entities, with a knowledge graph, enriched with domain-specific information. We have implemented the following Cypher queries to generate these new relationships:

```
MATCH (n1:Entity {ID: 'C1'}), (n2:Customer {id2: 'C1'}),(n3:Entity {ID: 'C2'}), (n4:Customer {id2: 'C2'}),(n5:Entity {ID: 'C3'}), (n6:Customer {id2: 'C3'})
CREATE (n1)-[:REL]->(n2),(n3)-[:REL]->(n4),(n5)-[:REL]->(n6)

MATCH (node1:`Packages` {EntityType: 'Packages'}), (node2:Entity {ID: 'P1'}), (node3:Entity {ID: 'P2'}), (node4:Entity {ID: 'P3'}), (node5:Entity {ID: 'P4'}), (node6:Entity {ID: 'P5'}), (node7:Entity {ID: 'P6'}), (node8:Entity {ID: 'P7'})
CREATE (node1)-[:REL]->(node2), (node1)-[:REL]->(node3), (node1)-[:REL]->(node4), (node1)-[:REL]->(node5), (node1)-[:REL]->(node6), (node1)-[:REL]->(node7), (node1)-[:REL]->(node8);
```

## 4. Perform process discovery using the newly constructed domain aware event knowledge graph (DAEKG).

In the following the questions (Q) are listed with the corresponding Cypher queries in order to answer the questions. To explore research question RQ2 comprehensively, we
break it down into smaller, more detailed sub-questions, specifically Q2.1 to Q2.9. These initial sub-questions can be answered exclusively using the
event knowledge graph:

* Q2.1: Which customer placed the most expensive order?

```
MATCH (n1)<-[:CORR]-(n2{Activity: 'pay order'})
RETURN n1, n2;
```

* Q2.2: What items are included in order O2 and how many packages were created for its delivery?

```
MATCH (n1{ID: 'O2'})<-[:CORR]-(n2{Activity: 'create package'})-[:CORR]->(n3)
RETURN n1, n2, n3;
```

* Q2.3: Is payment made for orders prior to the creation of their first package? It is a requirement that each order must be paid for before the initial package is created for shipment. It is also possible for orders to be shipped in multiple packages.

```
MATCH (n1{EntityType:'Order'})<-[:CORR]-(n2)
WHERE n2.Activity IN ['pay order', 'create package']
RETURN n1, n2;
```

* Q2.4: Are orders delivered within 21 days? After an order is placed, all packages should be delivered within 21 days.

```
MATCH (n1{EntityType:'Order'})<-[:CORR]-(n2)
WHERE n2.Activity IN ['place order', 'package delivered']
RETURN n1, n2;
```


The event knowledge graph, when complemented by the constructed knowledge graph, forming the domain-aware event knowledge graph, is capable of addressing more advanced research questions, RQ2.5 to RQ2.9, which seek to provide more detailed answers:

* Q2.5: Where do our customers live?

```
MATCH (n1)-[:REL]-(n2)-[:LIVES_IN]->(n3)
RETURN n1, n2, n3;
```

* Q2.6: What is the customer type of our customers?

```
MATCH (n1)-[:REL]-(n2)-[:HAS_TYPE]->(n3) 
RETURN n1, n2, n3;
```

* Q2.7: What is the quantity of packages being sent in small (S), medium (M) and large (L) sizes and are sufficient package sizes still available?

```
MATCH (n1)-[:CORR]->(n2{EntityType:'Package'})-[:REL]-(n3)-[:HAS_SIZE]->(n4)
WHERE n1.Activity IN ['create package']
RETURN n1, n2, n3, n4;
```

* Q2.8: Depending on the customer type, were different delivery times adhered to (21 days for basic and regular customers and 14 days for premium customers)?

```
MATCH (n1{EntityType:'Order'})-[:CORR]-(n2)-[:CORR]-(n3)-[:REL]->(n4)-[:HAS_TYPE]->(n5)
WHERE n2.Activity IN ['place order', 'package delivered']
RETURN n1, n2, n3, n4, n5;
```

* Q2.9: Could orders O1 and O2 have been delivered on time? If so, what would be the action for achieving this?

```
MATCH (n1)-[:CORR]-(n2)-[:CORR]-(n3)-[:CORR]-(n4{EntityType:'Customer'})
-[:REL]-(n5)-[:LIVES_IN]-(n6)-[:OPERATES_IN]-(n7)
WHERE n1.Activity = 'place order' AND n2.ID IN ['O1', 'O2'] 
AND n3.Activity = '2package delivered'
WITH n1, n3, n4, n5, n6, n2, n7, n3.timestamp AS 
PackageDeliveredTimestamp
ORDER BY PackageDeliveredTimestamp DESC
WITH n1, n4, n5, n6, n2, n7, COLLECT(n3)[0] AS latestn3
RETURN n1, n2, latestn3 AS n3, n4, n5, n6, n7;
```

End of tutorial!






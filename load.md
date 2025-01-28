# Loading Data into AvantGraph
Documents how to load graph data (vertices and edges) into AvantGraph.
There are three steps:
1. Convert your data to the Neo4J-compatible JSON format.
2. Define the graph schema using `ag-schema`.
3. Load the data into the graph with `ag-load-graph`.

## Input Format
AvantGraph follows the [Neo4J JSON format for graph data](https://neo4j.com/labs/apoc/4.4/export/json/).
Graph data must be converted into this format before it can be loaded.

Input files must be UTF-8 encoded text with a single JSON object per line.
Each line/object defines either a vertex or an edge.

Vertex objects have the following properties:

* `type`: the value is `"node"`, indicating that the object is a vertex.
* `labels`: this contains a list of strings representing the labels of the vertex in the loaded graph.
   The set of labels must match those of a vertex table defined in the schema.
* `id`: a unique identifier for the vertex. 
  In the loaded graph this value becomes the primary key.
  It is also used by edge objects to refer to this vertex.
* `properties`: a JSON object containing the property values for this vertex. 
  Supported property types include strings, integers (signed 64-bit), floating point (also 64-bit) and booleans.
  *NOTE: Nested data types and lists are not yet supported.*

Edge objects have the following properties:

* `type`: the value is `"relationship"`, indicating that the object is an edge.
* `label`: a string representing the label of the edge in the loaded graph
* `properties`: a JSON object containing the property values for this edge, similar to the properties object on vertices.
* `start`/`end`: a JSON object pointing to a vertex (source and target vertex, respectively). 
  It has properties `id` and `labels` to identify the pointed-to vertex, which must match with `id` and `labels` of a **previously loaded** vertex.

An example of a valid graph definition is given below.

```json
{ "type": "node", "labels": ["MyVertexType"], "id": "000", "properties": {"myint": 42, "mystring": "hello"} }
{ "type": "node", "labels": ["MyVertexType"], "id": "001", "properties": {"myint": 43, "mystring": "world"} }
{ "type": "relationship", "start": { "id": "000", "labels": ["MyVertexType"] }, "end": { "id": "001", "labels": ["MyVertexType"] }, "label": "MyEdgeType", "properties": {"mybool": true} }
```

## Defining The Graph Schema
AvantGraph is statically typed. 
Before loading any data, you must define the types of vertices and edges that exist in the graph, and what properties they have.

Start by initializing a new graph directory:

```bash
ag-schema create-graph /tmp/my-graph
```

Then create a vertex table for every type of vertex in the graph, e.g:

```bash
# NOTE: The first property is the primary key, and must be of type CHAR_STRING.
ag-schema create-vertex-table /tmp/my-graph/ --vertex-label=MyVertexType \
    --vertex-property=id=CHAR_STRING \
    --vertex-property=myint=S64 \
    --vertex-property=mystring=CHAR_STRING
```

A vertex table must have a set of one or more labels.
To define a vertex table with more than one label, repeat the `--vertex-label` flag.

Lastly, create edge tables for the different relationships in the graph, e.g:

```bash
ag-schema create-edge-table /tmp/my-graph/ --edge-label=MyEdgeType \
    --src-labels=MyVertexType \
    --trg-labels=MyVertexType \
    --edge-property=mybool=BOOLEAN
```

Edges always have a single label.

AvantGraph automatically defines a reverse edge table, so there is no need to create such tables manually. 
For example, if you define a table `Cites`, adding another table `IsCitedBy` is redundant.

## Loading the Data
With the data and schema ready, populate the graph using `ag-load-graph`:

```bash
ag-load-graph --graph-format=json in.jsonnl /tmp/my-graph/
```

If the inputs for different tables are stored in separate files, you can load them one after the other:

```bash
ag-load-graph --graph-format=json MyVertexType.jsonnl /tmp/my-graph/
ag-load-graph --graph-format=json MyEdgeType.jsonnl /tmp/my-graph/
```
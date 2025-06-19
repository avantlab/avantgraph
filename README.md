# AvantGraph - The next-generation graph analytics engine
> AvantGraph will be released under an open license soon.
>
> Until then, you can try out some of the features with the provided docker image.

## Getting Started

### A note on compatibility
AvantGraph currently requires the Linux-only `io_uring` API.
We recommend running it on an x86-64 linux host with a kernel version 5.15 or newer.

### Pull the docker container
Make sure you have [installed docker](https://docs.docker.com/engine/install/) and that the daemon is [running](https://docs.docker.com/config/daemon/start/).

AvantGraph is published on our GitHub container registry.
You can fetch it using the following command:

```bash
docker pull ghcr.io/avantlab/avantgraph:release-2025-06-19
```

### Start the docker container
Start an instance of the image you have just pulled:

```bash
docker run -it --rm \
    -p 127.0.0.1:7687:7687/tcp \
    --privileged \
    ghcr.io/avantlab/avantgraph:release-2025-06-19
```

This opens an interactive shell inside the AvantGraph container.
The `--privileged` flag enables `io_uring` support on Linux hosts.
**Important**: Keep this terminal open - you'll run all subsequent commands inside this container.

### Load a graph
Start by creating a graph and loading vertex and edge data.
Below we provide sample instructions for loading a small toy graph.
Detailed instructions are available [here](./load.md)

```bash
export GRAPH_DIR=/tmp/toots

# Initialize the graph directory
ag-schema create-graph "${GRAPH_DIR}"

# Define the graph schema
ag-schema create-vertex-table "${GRAPH_DIR}" --vertex-label=Person --vertex-property=id=CHAR_STRING --vertex-property=name=CHAR_STRING --vertex-property=age=S64
ag-schema create-vertex-table "${GRAPH_DIR}" --vertex-label=Person --vertex-label=Moderator --vertex-property=id=CHAR_STRING --vertex-property=name=CHAR_STRING --vertex-property=age=S64
ag-schema create-vertex-table "${GRAPH_DIR}" --vertex-label=Toot --vertex-property=id=CHAR_STRING --vertex-property=likes=S64 --vertex-property=retoots=S64 --vertex-property=content=CHAR_STRING

ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=Follows --src-labels=Person --trg-labels=Person --edge-property=days=S64
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=Tooted --src-labels=Person --trg-labels=Toot --edge-property=days=S64
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=Likes --src-labels=Person --trg-labels=Toot --edge-property=days=S64
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=Retoots --src-labels=Person --trg-labels=Toot --edge-property=days=S64
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=Mentions --src-labels=Toot --trg-labels=Person
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=RepliesTo --src-labels=Toot --trg-labels=Toot
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=Removed --src-labels=Person --src-labels=Moderator --trg-labels=Toot --edge-property=reason=CHAR_STRING

# JSON in Neo4J dump format
cat <<EOF > /tmp/toots.json
{"type": "node", "id": "0", "labels": ["Person"], "properties": {"name": "Ana", "age": 50}}
{"type": "node", "id": "1", "labels": ["Person"], "properties": {"name": "Jim"}}
{"type": "node", "id": "2", "labels": ["Person"], "properties": {"name": "David", "age": 25}}
{"type": "node", "id": "3", "labels": ["Toot"], "properties": {"content": "Happy birthday @Jim!", "likes": 15, "retoots": 4}}
{"type": "node", "id": "4", "labels": ["Toot"], "properties": {"content": "It is not his birthday!", "likes": 10, "retoots": 2}}
{"type": "node", "id": "5", "labels": ["Person", "Moderator"], "properties": {"name": "Bruno"}}
{"type": "relationship", "id": "0", "start": {"id": "0", "labels": ["Person"]}, "end": {"id": "1", "labels": ["Person"]}, "label": "Follows", "properties": {"days": 50}}
{"type": "relationship", "id": "1", "start": {"id": "0", "labels": ["Person"]}, "end": {"id": "2", "labels": ["Person"]}, "label": "Follows", "properties": {"days": 80}}
{"type": "relationship", "id": "2", "start": {"id": "1", "labels": ["Person"]}, "end": {"id": "2", "labels": ["Person"]}, "label": "Follows", "properties": {"days": 20}}
{"type": "relationship", "id": "3", "start": {"id": "0", "labels": ["Person"]}, "end": {"id": "4", "labels": ["Toot"]}, "label": "Likes", "properties": {}}
{"type": "relationship", "id": "6", "start": {"id": "0", "labels": ["Person"]}, "end": {"id": "3", "labels": ["Toot"]}, "label": "Tooted", "properties": {}}
{"type": "relationship", "id": "4", "start": {"id": "1", "labels": ["Person"]}, "end": {"id": "4", "labels": ["Toot"]}, "label": "Tooted", "properties": {}}
{"type": "relationship", "id": "5", "start": {"id": "2", "labels": ["Person"]}, "end": {"id": "3", "labels": ["Toot"]}, "label": "Retoots", "properties": {"days": 200}}
{"type": "relationship", "id": "7", "start": {"id": "4", "labels": ["Toot"]}, "end": {"id": "3", "labels": ["Toot"]}, "label": "RepliesTo", "properties": {}}
{"type": "relationship", "id": "8", "start": {"id": "5", "labels": ["Person", "Moderator"]}, "end": {"id": "4", "labels": ["Toot"]}, "label": "Removed", "properties": {"reason": "Offensive content"}}
EOF

# Load JSON into AvantGraph
ag-load-graph --graph-format=json /tmp/toots.json "$GRAPH_DIR"
```

### Run a query using the AvantGraph CLI
The `avantgraph` binary is the command line interface to AvantGraph.
Define the query to run in a file, and use the `avantgraph` binary to execute it against a loaded graph:

```bash
cat <<EOF > /tmp/query.cypher
MATCH (a:Person)-[:Tooted]->(t:Toot)
RETURN a.name AS person, t.content AS message;
EOF

avantgraph /tmp/toots --query-type=cypher /tmp/query.cypher
```

The query returns two result tuples:

```
tuple:  %person=CHAR_STRING:"Ana" %message=CHAR_STRING:"Happy birthday @Jim!"
tuple:  %person=CHAR_STRING:"Jim" %message=CHAR_STRING:"It is not his birthday!"
```

### Use AvantGraph in server mode
You can also start AvantGraph in server mode and interact with it through the Neo4J BOLT protocol.
The tool to start the local server is `ag-server`:

```bash
ag-server --listen 0.0.0.0:7687 /tmp/toots
```

Now **open a new terminal on your host machine** (not inside the Docker container).
Install the Neo4J client library for python:

```
pip3 install --user neo4j
```

Create a new file `bolt_client.py`.
Paste in the following script:

```python
#!/usr/bin/env python3

# AvantGraph is wire-compatible with Neo4J.
# We can use the standard neo4j python library to interact with it.
import neo4j
from neo4j import GraphDatabase, Result

# Connect to the AvantGraph server running in the docker container.
driver = GraphDatabase.driver("bolt://localhost:7687")

# The Cypher query to execute
query = """
MATCH (a:Person)-[:Tooted]->(t:Toot)
RETURN a.name AS person, t.content AS message;
"""

# Execute the query and read the results into a dataframe.
df = driver.execute_query(query, result_transformer_=Result.to_df)
print(df)
```

Run the script:

```bash
python3 bolt_client.py
```

We get an output identical to the command line version, but in a pandas dataframe:

```
  person                  message
0    Ana     Happy birthday @Jim!
1    Jim  It is not his birthday!
```

### Run a graph algorithm
> Graph Algorithm support is currently included as a preview, it is not stable yet.
> You may encounter performance issues with some algorithms as a result.
> Documentation on the algorithm language will come in a future release.
> In the meantime, if you would like to run custom algorithms in AvantGraph, contact us :)

AvantGraph includes support for user-provided graph algorithms written in the *GraphAlg* language.
To run a custom algorithm, include it in a Cypher query using the `WITH ALGORITHM` clause:

```cypher
WITH ALGORITHM "
func MyAlgorithm(graph: Matrix<s, s, bool>) -> Vector<s, real> {
    // Implementation goes here
    // ...
}
"
CALL MyAlgorithm()
```

An example script to load a graph and compute PageRank scores for the nodes is given below:

```bash
export GRAPH_DIR=/tmp/pr

# Initialize the graph directory
ag-schema create-graph "${GRAPH_DIR}"

# Define the graph schema
ag-schema create-vertex-table "${GRAPH_DIR}" --vertex-label=evlp_vertex --vertex-property=orig_id=U64
ag-schema create-edge-table "${GRAPH_DIR}" --edge-label=evlp_edge --src-labels=evlp_vertex --trg-labels=evlp_vertex

# JSON in Neo4J dump format
cat <<EOF > /tmp/pr.json
{"type": "node","id": "1","labels": ["evlp_vertex"],"properties": {"orig_id": 1}}
{"type": "node","id": "2","labels": ["evlp_vertex"],"properties": {"orig_id": 2}}
{"type": "node","id": "3","labels": ["evlp_vertex"],"properties": {"orig_id": 3}}
{"type": "node","id": "4","labels": ["evlp_vertex"],"properties": {"orig_id": 4}}
{"type": "node","id": "5","labels": ["evlp_vertex"],"properties": {"orig_id": 5}}
{"type": "node","id": "6","labels": ["evlp_vertex"],"properties": {"orig_id": 6}}
{"type": "node","id": "7","labels": ["evlp_vertex"],"properties": {"orig_id": 7}}
{"type": "node","id": "8","labels": ["evlp_vertex"],"properties": {"orig_id": 8}}
{"type": "node","id": "9","labels": ["evlp_vertex"],"properties": {"orig_id": 9}}
{"type": "node","id": "10","labels": ["evlp_vertex"],"properties": {"orig_id": 10}}
{"type": "node","id": "11","labels": ["evlp_vertex"],"properties": {"orig_id": 11}}
{"type": "node","id": "12","labels": ["evlp_vertex"],"properties": {"orig_id": 12}}
{"type": "node","id": "13","labels": ["evlp_vertex"],"properties": {"orig_id": 13}}
{"type": "node","id": "14","labels": ["evlp_vertex"],"properties": {"orig_id": 14}}
{"type": "node","id": "15","labels": ["evlp_vertex"],"properties": {"orig_id": 15}}
{"type": "node","id": "16","labels": ["evlp_vertex"],"properties": {"orig_id": 16}}
{"type": "node","id": "17","labels": ["evlp_vertex"],"properties": {"orig_id": 17}}
{"type": "node","id": "18","labels": ["evlp_vertex"],"properties": {"orig_id": 18}}
{"type": "node","id": "19","labels": ["evlp_vertex"],"properties": {"orig_id": 19}}
{"type": "node","id": "20","labels": ["evlp_vertex"],"properties": {"orig_id": 20}}
{"type": "node","id": "21","labels": ["evlp_vertex"],"properties": {"orig_id": 21}}
{"type": "node","id": "22","labels": ["evlp_vertex"],"properties": {"orig_id": 22}}
{"type": "node","id": "23","labels": ["evlp_vertex"],"properties": {"orig_id": 23}}
{"type": "node","id": "24","labels": ["evlp_vertex"],"properties": {"orig_id": 24}}
{"type": "node","id": "25","labels": ["evlp_vertex"],"properties": {"orig_id": 25}}
{"type": "node","id": "26","labels": ["evlp_vertex"],"properties": {"orig_id": 26}}
{"type": "node","id": "27","labels": ["evlp_vertex"],"properties": {"orig_id": 27}}
{"type": "node","id": "28","labels": ["evlp_vertex"],"properties": {"orig_id": 28}}
{"type": "node","id": "29","labels": ["evlp_vertex"],"properties": {"orig_id": 29}}
{"type": "node","id": "30","labels": ["evlp_vertex"],"properties": {"orig_id": 30}}
{"type": "node","id": "31","labels": ["evlp_vertex"],"properties": {"orig_id": 31}}
{"type": "node","id": "32","labels": ["evlp_vertex"],"properties": {"orig_id": 32}}
{"type": "node","id": "33","labels": ["evlp_vertex"],"properties": {"orig_id": 33}}
{"type": "node","id": "34","labels": ["evlp_vertex"],"properties": {"orig_id": 34}}
{"type": "node","id": "35","labels": ["evlp_vertex"],"properties": {"orig_id": 35}}
{"type": "node","id": "36","labels": ["evlp_vertex"],"properties": {"orig_id": 36}}
{"type": "node","id": "37","labels": ["evlp_vertex"],"properties": {"orig_id": 37}}
{"type": "node","id": "38","labels": ["evlp_vertex"],"properties": {"orig_id": 38}}
{"type": "node","id": "39","labels": ["evlp_vertex"],"properties": {"orig_id": 39}}
{"type": "node","id": "40","labels": ["evlp_vertex"],"properties": {"orig_id": 40}}
{"type": "node","id": "41","labels": ["evlp_vertex"],"properties": {"orig_id": 41}}
{"type": "node","id": "42","labels": ["evlp_vertex"],"properties": {"orig_id": 42}}
{"type": "node","id": "43","labels": ["evlp_vertex"],"properties": {"orig_id": 43}}
{"type": "node","id": "44","labels": ["evlp_vertex"],"properties": {"orig_id": 44}}
{"type": "node","id": "45","labels": ["evlp_vertex"],"properties": {"orig_id": 45}}
{"type": "node","id": "46","labels": ["evlp_vertex"],"properties": {"orig_id": 46}}
{"type": "node","id": "47","labels": ["evlp_vertex"],"properties": {"orig_id": 47}}
{"type": "node","id": "48","labels": ["evlp_vertex"],"properties": {"orig_id": 48}}
{"type": "node","id": "49","labels": ["evlp_vertex"],"properties": {"orig_id": 49}}
{"type": "node","id": "50","labels": ["evlp_vertex"],"properties": {"orig_id": 50}}
{"type":"relationship","id":"0","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"19","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"1","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"21","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"2","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"22","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"3","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"27","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"4","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"5","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"37","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"6","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"45","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"7","label":"evlp_edge","start":{"id":"1","labels":["evlp_vertex"]},"end":{"id":"48","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"8","label":"evlp_edge","start":{"id":"2","labels":["evlp_vertex"]},"end":{"id":"3","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"9","label":"evlp_edge","start":{"id":"2","labels":["evlp_vertex"]},"end":{"id":"20","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"10","label":"evlp_edge","start":{"id":"2","labels":["evlp_vertex"]},"end":{"id":"39","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"11","label":"evlp_edge","start":{"id":"2","labels":["evlp_vertex"]},"end":{"id":"46","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"12","label":"evlp_edge","start":{"id":"3","labels":["evlp_vertex"]},"end":{"id":"6","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"13","label":"evlp_edge","start":{"id":"3","labels":["evlp_vertex"]},"end":{"id":"10","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"14","label":"evlp_edge","start":{"id":"3","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"15","label":"evlp_edge","start":{"id":"3","labels":["evlp_vertex"]},"end":{"id":"41","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"16","label":"evlp_edge","start":{"id":"3","labels":["evlp_vertex"]},"end":{"id":"45","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"17","label":"evlp_edge","start":{"id":"4","labels":["evlp_vertex"]},"end":{"id":"15","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"18","label":"evlp_edge","start":{"id":"5","labels":["evlp_vertex"]},"end":{"id":"15","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"19","label":"evlp_edge","start":{"id":"5","labels":["evlp_vertex"]},"end":{"id":"16","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"20","label":"evlp_edge","start":{"id":"5","labels":["evlp_vertex"]},"end":{"id":"18","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"21","label":"evlp_edge","start":{"id":"5","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"22","label":"evlp_edge","start":{"id":"5","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"23","label":"evlp_edge","start":{"id":"6","labels":["evlp_vertex"]},"end":{"id":"49","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"24","label":"evlp_edge","start":{"id":"7","labels":["evlp_vertex"]},"end":{"id":"6","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"25","label":"evlp_edge","start":{"id":"7","labels":["evlp_vertex"]},"end":{"id":"27","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"26","label":"evlp_edge","start":{"id":"7","labels":["evlp_vertex"]},"end":{"id":"43","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"27","label":"evlp_edge","start":{"id":"7","labels":["evlp_vertex"]},"end":{"id":"46","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"28","label":"evlp_edge","start":{"id":"8","labels":["evlp_vertex"]},"end":{"id":"5","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"29","label":"evlp_edge","start":{"id":"8","labels":["evlp_vertex"]},"end":{"id":"21","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"30","label":"evlp_edge","start":{"id":"8","labels":["evlp_vertex"]},"end":{"id":"29","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"31","label":"evlp_edge","start":{"id":"8","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"32","label":"evlp_edge","start":{"id":"8","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"33","label":"evlp_edge","start":{"id":"8","labels":["evlp_vertex"]},"end":{"id":"43","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"34","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"16","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"35","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"18","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"36","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"21","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"37","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"38","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"39","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"35","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"40","label":"evlp_edge","start":{"id":"9","labels":["evlp_vertex"]},"end":{"id":"40","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"41","label":"evlp_edge","start":{"id":"10","labels":["evlp_vertex"]},"end":{"id":"9","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"42","label":"evlp_edge","start":{"id":"10","labels":["evlp_vertex"]},"end":{"id":"13","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"43","label":"evlp_edge","start":{"id":"10","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"44","label":"evlp_edge","start":{"id":"10","labels":["evlp_vertex"]},"end":{"id":"29","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"45","label":"evlp_edge","start":{"id":"10","labels":["evlp_vertex"]},"end":{"id":"33","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"46","label":"evlp_edge","start":{"id":"11","labels":["evlp_vertex"]},"end":{"id":"3","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"47","label":"evlp_edge","start":{"id":"11","labels":["evlp_vertex"]},"end":{"id":"39","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"48","label":"evlp_edge","start":{"id":"12","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"49","label":"evlp_edge","start":{"id":"12","labels":["evlp_vertex"]},"end":{"id":"50","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"50","label":"evlp_edge","start":{"id":"13","labels":["evlp_vertex"]},"end":{"id":"7","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"51","label":"evlp_edge","start":{"id":"13","labels":["evlp_vertex"]},"end":{"id":"12","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"52","label":"evlp_edge","start":{"id":"13","labels":["evlp_vertex"]},"end":{"id":"17","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"53","label":"evlp_edge","start":{"id":"13","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"54","label":"evlp_edge","start":{"id":"13","labels":["evlp_vertex"]},"end":{"id":"48","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"55","label":"evlp_edge","start":{"id":"14","labels":["evlp_vertex"]},"end":{"id":"4","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"56","label":"evlp_edge","start":{"id":"14","labels":["evlp_vertex"]},"end":{"id":"20","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"57","label":"evlp_edge","start":{"id":"14","labels":["evlp_vertex"]},"end":{"id":"21","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"58","label":"evlp_edge","start":{"id":"14","labels":["evlp_vertex"]},"end":{"id":"35","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"59","label":"evlp_edge","start":{"id":"14","labels":["evlp_vertex"]},"end":{"id":"38","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"60","label":"evlp_edge","start":{"id":"14","labels":["evlp_vertex"]},"end":{"id":"40","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"61","label":"evlp_edge","start":{"id":"15","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"62","label":"evlp_edge","start":{"id":"15","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"63","label":"evlp_edge","start":{"id":"15","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"64","label":"evlp_edge","start":{"id":"15","labels":["evlp_vertex"]},"end":{"id":"35","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"65","label":"evlp_edge","start":{"id":"15","labels":["evlp_vertex"]},"end":{"id":"44","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"66","label":"evlp_edge","start":{"id":"17","labels":["evlp_vertex"]},"end":{"id":"5","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"67","label":"evlp_edge","start":{"id":"17","labels":["evlp_vertex"]},"end":{"id":"9","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"68","label":"evlp_edge","start":{"id":"17","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"69","label":"evlp_edge","start":{"id":"17","labels":["evlp_vertex"]},"end":{"id":"16","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"70","label":"evlp_edge","start":{"id":"17","labels":["evlp_vertex"]},"end":{"id":"26","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"71","label":"evlp_edge","start":{"id":"17","labels":["evlp_vertex"]},"end":{"id":"37","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"72","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"1","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"73","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"12","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"74","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"75","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"76","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"44","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"77","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"45","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"78","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"79","label":"evlp_edge","start":{"id":"18","labels":["evlp_vertex"]},"end":{"id":"50","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"80","label":"evlp_edge","start":{"id":"19","labels":["evlp_vertex"]},"end":{"id":"10","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"81","label":"evlp_edge","start":{"id":"19","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"82","label":"evlp_edge","start":{"id":"19","labels":["evlp_vertex"]},"end":{"id":"13","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"83","label":"evlp_edge","start":{"id":"19","labels":["evlp_vertex"]},"end":{"id":"27","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"84","label":"evlp_edge","start":{"id":"19","labels":["evlp_vertex"]},"end":{"id":"38","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"85","label":"evlp_edge","start":{"id":"20","labels":["evlp_vertex"]},"end":{"id":"15","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"86","label":"evlp_edge","start":{"id":"20","labels":["evlp_vertex"]},"end":{"id":"25","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"87","label":"evlp_edge","start":{"id":"21","labels":["evlp_vertex"]},"end":{"id":"22","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"88","label":"evlp_edge","start":{"id":"21","labels":["evlp_vertex"]},"end":{"id":"27","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"89","label":"evlp_edge","start":{"id":"21","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"90","label":"evlp_edge","start":{"id":"21","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"91","label":"evlp_edge","start":{"id":"21","labels":["evlp_vertex"]},"end":{"id":"40","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"92","label":"evlp_edge","start":{"id":"22","labels":["evlp_vertex"]},"end":{"id":"19","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"93","label":"evlp_edge","start":{"id":"22","labels":["evlp_vertex"]},"end":{"id":"26","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"94","label":"evlp_edge","start":{"id":"22","labels":["evlp_vertex"]},"end":{"id":"27","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"95","label":"evlp_edge","start":{"id":"22","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"96","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"22","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"97","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"35","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"98","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"36","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"99","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"38","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"100","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"40","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"101","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"46","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"102","label":"evlp_edge","start":{"id":"23","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"103","label":"evlp_edge","start":{"id":"24","labels":["evlp_vertex"]},"end":{"id":"9","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"104","label":"evlp_edge","start":{"id":"24","labels":["evlp_vertex"]},"end":{"id":"13","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"105","label":"evlp_edge","start":{"id":"24","labels":["evlp_vertex"]},"end":{"id":"15","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"106","label":"evlp_edge","start":{"id":"24","labels":["evlp_vertex"]},"end":{"id":"34","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"107","label":"evlp_edge","start":{"id":"24","labels":["evlp_vertex"]},"end":{"id":"36","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"108","label":"evlp_edge","start":{"id":"24","labels":["evlp_vertex"]},"end":{"id":"50","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"109","label":"evlp_edge","start":{"id":"25","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"110","label":"evlp_edge","start":{"id":"25","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"111","label":"evlp_edge","start":{"id":"25","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"112","label":"evlp_edge","start":{"id":"25","labels":["evlp_vertex"]},"end":{"id":"34","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"113","label":"evlp_edge","start":{"id":"25","labels":["evlp_vertex"]},"end":{"id":"41","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"114","label":"evlp_edge","start":{"id":"25","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"115","label":"evlp_edge","start":{"id":"26","labels":["evlp_vertex"]},"end":{"id":"7","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"116","label":"evlp_edge","start":{"id":"26","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"117","label":"evlp_edge","start":{"id":"26","labels":["evlp_vertex"]},"end":{"id":"37","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"118","label":"evlp_edge","start":{"id":"26","labels":["evlp_vertex"]},"end":{"id":"40","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"119","label":"evlp_edge","start":{"id":"26","labels":["evlp_vertex"]},"end":{"id":"44","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"120","label":"evlp_edge","start":{"id":"26","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"121","label":"evlp_edge","start":{"id":"27","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"122","label":"evlp_edge","start":{"id":"27","labels":["evlp_vertex"]},"end":{"id":"33","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"123","label":"evlp_edge","start":{"id":"27","labels":["evlp_vertex"]},"end":{"id":"43","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"124","label":"evlp_edge","start":{"id":"28","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"125","label":"evlp_edge","start":{"id":"28","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"126","label":"evlp_edge","start":{"id":"28","labels":["evlp_vertex"]},"end":{"id":"42","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"127","label":"evlp_edge","start":{"id":"28","labels":["evlp_vertex"]},"end":{"id":"45","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"128","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"1","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"129","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"2","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"130","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"12","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"131","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"14","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"132","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"16","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"133","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"19","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"134","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"20","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"135","label":"evlp_edge","start":{"id":"29","labels":["evlp_vertex"]},"end":{"id":"36","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"136","label":"evlp_edge","start":{"id":"30","labels":["evlp_vertex"]},"end":{"id":"9","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"137","label":"evlp_edge","start":{"id":"30","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"138","label":"evlp_edge","start":{"id":"30","labels":["evlp_vertex"]},"end":{"id":"34","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"139","label":"evlp_edge","start":{"id":"30","labels":["evlp_vertex"]},"end":{"id":"44","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"140","label":"evlp_edge","start":{"id":"31","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"141","label":"evlp_edge","start":{"id":"31","labels":["evlp_vertex"]},"end":{"id":"17","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"142","label":"evlp_edge","start":{"id":"31","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"143","label":"evlp_edge","start":{"id":"31","labels":["evlp_vertex"]},"end":{"id":"39","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"144","label":"evlp_edge","start":{"id":"31","labels":["evlp_vertex"]},"end":{"id":"46","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"145","label":"evlp_edge","start":{"id":"31","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"146","label":"evlp_edge","start":{"id":"32","labels":["evlp_vertex"]},"end":{"id":"2","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"147","label":"evlp_edge","start":{"id":"32","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"148","label":"evlp_edge","start":{"id":"32","labels":["evlp_vertex"]},"end":{"id":"29","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"149","label":"evlp_edge","start":{"id":"32","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"150","label":"evlp_edge","start":{"id":"32","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"151","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"7","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"152","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"153","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"9","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"154","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"10","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"155","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"156","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"34","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"157","label":"evlp_edge","start":{"id":"33","labels":["evlp_vertex"]},"end":{"id":"37","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"158","label":"evlp_edge","start":{"id":"34","labels":["evlp_vertex"]},"end":{"id":"26","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"159","label":"evlp_edge","start":{"id":"34","labels":["evlp_vertex"]},"end":{"id":"48","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"160","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"3","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"161","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"10","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"162","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"17","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"163","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"164","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"26","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"165","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"166","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"33","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"167","label":"evlp_edge","start":{"id":"35","labels":["evlp_vertex"]},"end":{"id":"41","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"168","label":"evlp_edge","start":{"id":"36","labels":["evlp_vertex"]},"end":{"id":"20","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"169","label":"evlp_edge","start":{"id":"36","labels":["evlp_vertex"]},"end":{"id":"21","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"170","label":"evlp_edge","start":{"id":"36","labels":["evlp_vertex"]},"end":{"id":"29","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"171","label":"evlp_edge","start":{"id":"36","labels":["evlp_vertex"]},"end":{"id":"32","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"172","label":"evlp_edge","start":{"id":"36","labels":["evlp_vertex"]},"end":{"id":"46","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"173","label":"evlp_edge","start":{"id":"37","labels":["evlp_vertex"]},"end":{"id":"1","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"174","label":"evlp_edge","start":{"id":"37","labels":["evlp_vertex"]},"end":{"id":"5","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"175","label":"evlp_edge","start":{"id":"37","labels":["evlp_vertex"]},"end":{"id":"9","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"176","label":"evlp_edge","start":{"id":"37","labels":["evlp_vertex"]},"end":{"id":"13","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"177","label":"evlp_edge","start":{"id":"37","labels":["evlp_vertex"]},"end":{"id":"23","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"178","label":"evlp_edge","start":{"id":"37","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"179","label":"evlp_edge","start":{"id":"38","labels":["evlp_vertex"]},"end":{"id":"2","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"180","label":"evlp_edge","start":{"id":"38","labels":["evlp_vertex"]},"end":{"id":"22","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"181","label":"evlp_edge","start":{"id":"38","labels":["evlp_vertex"]},"end":{"id":"50","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"182","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"6","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"183","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"184","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"20","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"185","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"186","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"187","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"188","label":"evlp_edge","start":{"id":"39","labels":["evlp_vertex"]},"end":{"id":"48","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"189","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"5","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"190","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"7","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"191","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"192","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"193","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"33","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"194","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"34","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"195","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"37","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"196","label":"evlp_edge","start":{"id":"40","labels":["evlp_vertex"]},"end":{"id":"49","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"197","label":"evlp_edge","start":{"id":"41","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"198","label":"evlp_edge","start":{"id":"41","labels":["evlp_vertex"]},"end":{"id":"43","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"199","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"1","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"200","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"2","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"201","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"202","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"15","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"203","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"17","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"204","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"29","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"205","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"38","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"206","label":"evlp_edge","start":{"id":"43","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"207","label":"evlp_edge","start":{"id":"44","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"208","label":"evlp_edge","start":{"id":"44","labels":["evlp_vertex"]},"end":{"id":"13","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"209","label":"evlp_edge","start":{"id":"44","labels":["evlp_vertex"]},"end":{"id":"15","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"210","label":"evlp_edge","start":{"id":"45","labels":["evlp_vertex"]},"end":{"id":"5","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"211","label":"evlp_edge","start":{"id":"45","labels":["evlp_vertex"]},"end":{"id":"11","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"212","label":"evlp_edge","start":{"id":"45","labels":["evlp_vertex"]},"end":{"id":"12","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"213","label":"evlp_edge","start":{"id":"45","labels":["evlp_vertex"]},"end":{"id":"21","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"214","label":"evlp_edge","start":{"id":"45","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"215","label":"evlp_edge","start":{"id":"46","labels":["evlp_vertex"]},"end":{"id":"23","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"216","label":"evlp_edge","start":{"id":"46","labels":["evlp_vertex"]},"end":{"id":"24","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"217","label":"evlp_edge","start":{"id":"46","labels":["evlp_vertex"]},"end":{"id":"26","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"218","label":"evlp_edge","start":{"id":"46","labels":["evlp_vertex"]},"end":{"id":"31","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"219","label":"evlp_edge","start":{"id":"46","labels":["evlp_vertex"]},"end":{"id":"36","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"220","label":"evlp_edge","start":{"id":"46","labels":["evlp_vertex"]},"end":{"id":"41","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"221","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"222","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"14","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"223","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"16","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"224","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"225","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"29","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"226","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"34","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"227","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"35","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"228","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"40","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"229","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"42","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"230","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"46","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"231","label":"evlp_edge","start":{"id":"47","labels":["evlp_vertex"]},"end":{"id":"50","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"232","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"233","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"19","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"234","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"30","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"235","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"35","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"236","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"38","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"237","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"43","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"238","label":"evlp_edge","start":{"id":"48","labels":["evlp_vertex"]},"end":{"id":"50","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"239","label":"evlp_edge","start":{"id":"49","labels":["evlp_vertex"]},"end":{"id":"7","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"240","label":"evlp_edge","start":{"id":"49","labels":["evlp_vertex"]},"end":{"id":"8","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"241","label":"evlp_edge","start":{"id":"49","labels":["evlp_vertex"]},"end":{"id":"17","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"242","label":"evlp_edge","start":{"id":"49","labels":["evlp_vertex"]},"end":{"id":"18","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"243","label":"evlp_edge","start":{"id":"50","labels":["evlp_vertex"]},"end":{"id":"4","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"244","label":"evlp_edge","start":{"id":"50","labels":["evlp_vertex"]},"end":{"id":"28","labels":["evlp_vertex"]},"properties":{"weight":0}}
{"type":"relationship","id":"245","label":"evlp_edge","start":{"id":"50","labels":["evlp_vertex"]},"end":{"id":"47","labels":["evlp_vertex"]},"properties":{"weight":0}}
EOF

# Load JSON into AvantGraph
ag-load-graph --graph-format=json /tmp/pr.json "$GRAPH_DIR"

# Write the query to a file
cat <<EOF > /tmp/pr.cypher
WITH ALGORITHM "
func withDamping(degree:int, damping:real) -> real {
    return cast<real>(degree) / damping;
}

func PageRank(
        graph: Matrix<s1, s1, bool>) -> Vector<s1, real> {
    damping = real(0.85);
    iterations = int(14);

    n = graph.nrows;
    teleport = (real(1.0) - damping) / cast<real>(n);
    rdiff = real(1.0);

    d_out = reduceRows(cast<int>(graph));

    d = apply(withDamping, d_out, damping);

    connected = reduceRows(graph);
    sinks = Vector<bool>(n);
    sinks<!connected>[:] = bool(true);

    pr = Vector<real>(n);
    pr[:] = real(1.0) / cast<real>(n);

    for i in int(0):iterations {
        sink_pr = Vector<real>(n);
        sink_pr<sinks> = pr;
        redist = (damping / cast<real>(n)) * reduce(sink_pr);

        w = pr (./) d;

        pr[:] = teleport + redist;
        pr += cast<real>(graph).T * w;
    }

    return pr;
}
"
CALL PageRank()
RETURN row.orig_id AS vid, val;

EOF

# Run the query with embedded algorithm.
avantgraph /tmp/pr --query-type=cypher /tmp/pr.cypher
```

GraphAlg is available anywhere Cypher is allowed, including in queries sent over the Neo4J BOLT protocol to an AvantGraph server.
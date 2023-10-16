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
docker pull ghcr.io/avantlab/avantgraph:release-2023-10-16
```

### Start the docker container
Start an instance of the image you have just pulled:

```bash
docker run -it --rm \
    -p 127.0.0.1:7687:7687/tcp \
    ghcr.io/avantlab/avantgraph:release-2023-10-16
```

This will open a shell inside the container. 

### Load a graph

```bash
ag-load-graph --graph-format=json fan-in-out.json output_graph/

# Check that the output_graph directory contains a loaded graph
ls -lh output_graph
```

### Run a query using the AvantGraph CLI

```bash
avantgraph output_graph/ --query-format=cypher query.cypher
```

### Start AvantGraph in server mode

```bash
ag-server --listen 0.0.0.0:7687 output_graph/
```

### Run a query using the BOLT protocol

#### From inside the container
Form another terminal window, attach to the same container:

```bash
CONTAINER=$(docker ps -f publish=7687 -q)
docker exec -it $CONTAINER bash -s
```

Then, run the provided script:

```bash
python3 bolt_client.py
```

#### From your local machine
Install the Neo4J client library for python on your local machine:

```
pip3 install --user neo4j
```

Create a new file `bolt_client.py`.
Paste in the following script:

```python
#!/usr/bin/env python3
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687")

session = driver.session()

query = """
MATCH ()-[r:IS_PART_OF]->()
RETURN *
"""

if True:
    result = session.run(query)
    print('result:')
    print (result)
    print('result value:')
    for value in result.value():
        print(value)
```

Run the script:

```bash
python3 bolt_client.py
```

### Run a graph algorithm
AvantGraph supports user-provided graph algorithms written in the [Graphalg language](https://repository.tudelft.nl/islandora/object/uuid%3A1b5f0236-87f2-41f0-8af8-0911cb3b4d54?collection=education).

Below we show how to run a simple triangle counting algorith embedded in a Cypher query.

Move to your terminal window with the running server, and stop it with `Ctrl-C`.
Start it again, but this time with the `karate` graph.
This graph comes preloaded inside of the container, so you do not need to use `ag-load-graph`.

```bash
ag-server --listen 0.0.0.0:7687 karate/
```

Follow the same steps as above but run `bolt_triangle_count.py` instead.

```bash
CONTAINER=$(docker ps -f publish=7687 -q)
docker exec -it $CONTAINER bash -s

python3 bolt_triangle_count.py
```

The script prints the number of triangles in the graph (45).

If you want to run locally, create `bolt_triangle_count.py` containing:

```python
#!/usr/bin/env python3
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://127.0.0.1:7687")

session = driver.session()

query = """
WITH ALGORITHM "
    func TriangleCount(graph: Matrix<int>) -> int {
        L = select(tril, graph, -1);
        U = select(triu, graph, 1);
        C<L, struct> = L (+.one) U.T;
        return reduce(+, C);
    }"
CALL TriangleCount()
RETURN count
"""

result = session.run(query)
for record in result:
    print(record.values())

```

And run as `python3 bolt_triangle_count.py`.

In the current release, only a subset of graphalg is supported.
Important features such as loops are not yet available, and will be added in the coming releases.

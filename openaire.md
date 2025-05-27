# OpenAIRE in AvantGraph
Getting started with the OpenAIRE graph in AvantGraph.

## A note on compatibility
AvantGraph currently requires the Linux-only `io_uring` API.
We recommend running it on an x86-64 linux host with a kernel version 5.15 or newer.

## Pull the docker container
Make sure you have [installed docker](https://docs.docker.com/engine/install/)
and that the daemon is [running](https://docs.docker.com/config/daemon/start/).

AvantGraph is published on our GitHub container registry.
You can fetch an image preloaded with an OpenAIRE subgraph using the following 
command:

```bash
# NOTE: This guide assumes the 'transport' version.
docker pull ghcr.io/avantlab/avantgraph:openaire-transport
docker pull ghcr.io/avantlab/avantgraph:openaire-cancer
docker pull ghcr.io/avantlab/avantgraph:openaire-neuro
docker pull ghcr.io/avantlab/avantgraph:openaire-energy
```

## Start the docker container
Start an instance of the image you have just pulled:

```bash
# Replace 'transport' with a subgraph of your choice.
docker run -it --rm --privileged ghcr.io/avantlab/avantgraph:openaire-transport
```

This will open a shell inside the container.
The `--privileged` flag is necessary for `io_uring` support.

## Extract the Graph
The graph is included in the image in compressed form.
Before running any queries, extract it using the following command:

```bash
tar -I zstd -xvf /ag.tar.zst
```

This should report creation of the files `ag/strings` and `ag/main`.
The graph is now available at `~/ag` inside the container, and ready to be queried.

## Query the Graph
Write your cypher query and save it to a file inside the container.
We include a few sample queries under `/home/ubuntu`.

Use the `avantgraph` binary to run the query.
For example:

```bash
avantgraph ag/ --query-type=cypher communities.cypher
```

## Jupyter Notebook
You can also interact with AvantGraph through a Jupyter Notebook instead of the command-line interface.
When you start the container, add an extra flag `-p 8888:8888` to expose the notebook server port to the host:

```bash
docker run -it --rm --privileged -p 8888:8888 ghcr.io/avantlab/avantgraph:openaire-transport
```

Then in the shell on the container, start AvantGraph in server mode together with Jupyter Notebook:

```bash
# Start the AvantGraph server in the background.
# We will connect to this from the notebook.
ag-server ag/ --listen :7687 &

jupyter notebook --ip 0.0.0.0
```

When you start the jupyter notebook server it will print a URL printed to the console of the form `http://127.0.0.1:8888/?token=<some token>`.
Open this URL in a browser (on the host) to access the notebook.

AvantGraph is wire-compatible with Neo4J, so you can use the `neo4j` python library to interact with AvantGraph inside the notebook.
In the initial cell of the notebook, load the required libraries:

```python
import neo4j
from neo4j import GraphDatabase, Result
driver = GraphDatabase.driver("bolt://localhost:7687")
```

Then you can use `driver` in following cells to runs queries and perform your analysis:

```python
# All publications etc. citing a specific paper
query = """
    MATCH (p:result)-[:Cites]->(cited:result)
    // NOTE: Exists only in the transport dataset.
    WHERE p.id = "doi_________::1a8de608ebacf2f4732134ce9f9580f1"
    RETURN cited.id, cited.mainTitle;
"""

# Produces a pandas dataframe with the results, which the notebook turns into a nice table.
driver.execute_query(query, result_transformer_=Result.to_df)
```

## Querying DOIs
DOIs are modeled as special nodes in the graph, with connections to the associated research products.
OpenAIRE research products can have multiple associated DOIs, and in rare cases two products can have the same DOI.
DOI lookup works in both directions: You can search for research products by DOI, or look up the DOI for a known entity.
We can modify the example above to find the DOI of the paper rather than its neighbours in the citation graph:

```python
query = """
    MATCH (p:result)<-[:doi]-(d:doi)
    // NOTE: Exists only in the transport dataset.
    WHERE p.id = "doi_________::1a8de608ebacf2f4732134ce9f9580f1"
    RETURN d.doi;
"""

# Format results as DOI
records, _, _ = driver.execute_query(query)
for record in records:
    print("https://doi.org/" + record[0])
```

Or, if you know the DOI but don't know the canonical ID in OpenAIRE:

```python
# Look up paper with DOI https://doi.org/10.3141/1780-10
query = """
    MATCH (d:doi)-[:doi]->(p:result)
    WHERE d.doi = "10.3141/1780-10"
    RETURN p.id, p.mainTitle;
"""

# Produces a pandas dataframe with the results, which the notebook turns into a nice table.
driver.execute_query(query, result_transformer_=Result.to_df)
```

## Run a Graph Algorithm
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

Algorithm support is also available through the Jupyter Notebook interface.
The example python script above can be modified as follows to run PageRank on the citation graph.

```python
# Run PageRank on the citation graph and extract the 10 highest scoring results.
query = """
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
CALL PageRank("result", "Cites")
RETURN row.id AS id, val AS score
ORDER BY val DESC
LIMIT 10;
"""

# Produces a pandas dataframe with the results, which the notebook turns into a nice table.
driver.execute_query(query, result_transformer_=Result.to_df)
```

## Advanced Use Cases
You can also start AvantGraph in server mode and send queries to it from the
host machine.
Instructions are available
[here](https://github.com/avantlab/avantgraph#start-avantgraph-in-server-mode).
# AvantGraph ❤️ Clinical Knowledge Graph
Getting started with the Clinical Knowledge Graph (CKG).

## A note on compatibility
AvantGraph currently requires the Linux-only `io_uring` API.
We recommend running it on an x86-64 linux host with a kernel version 5.15 or newer.

## Pull the docker container
Make sure you have [installed docker](https://docs.docker.com/engine/install/)
and that the daemon is [running](https://docs.docker.com/config/daemon/start/).

AvantGraph is published on our GitHub container registry.
You can fetch an image preloaded with the CKG graph using the following command:

```bash
docker pull ghcr.io/avantlab/avantgraph:ckg
```

## Start the docker container
Start an instance of the image you have just pulled:

```bash
docker run -it --rm --privileged ghcr.io/avantlab/avantgraph:ckg
```

This will open a shell inside the container.
The `--privileged` flag is necessary for `io_uring` support.

## Extract the Graph
The graph is included in the image in compressed form.
Before running any queries, extract it using the following command:

```bash
tar xvf ag-ckg.tar.gz
```

This should report creation of the files `/tmp/ckg/string` and `/tmp/ckg/main`.
The graph is now available at `/tmp/ckg` inside the container, and ready to be queried.

> The graph currently contains vertices with the labels *Disease* and
> *Publication*, and edges with the label *MENTIONED_IN_PUBLICATION*.
> Please contact us if you need additional parts of the CKG.

## Query the CKG
Write your cypher query and save it to a file inside the container.
You may use the following script to create some sample queries.

```bash
cat > /home/ubuntu/01-disease-per-doi.cypher <<EOF
// Given a set of DOIs, retrieve a list with (disease) annotations for each one
MATCH (d:Disease)-[:MENTIONED_IN_PUBLICATION]->(p:Publication)
WHERE p.DOI IN ["10.1016/j.ajoc.2018.02.026", "10.1371/journal.pone.0009879"]
RETURN
    p.DOI,
    d.name,
    d.id,
    d.description,
    d.synonyms
EOF

cat > /home/ubuntu/02-disease-properties.cypher <<EOF
// Given an entity id, retrieve some (or all) properties of that entity
MATCH (d:Disease)
WHERE d.id = "DOID:0014667"
RETURN d.name, d.id, d.description, d.synonyms
EOF

cat > /home/ubuntu/04-publications-per-disease.cypher <<EOF
// Count the publications associated with a given disease
MATCH (d:Disease { id: "DOID:0014667" })-[:MENTIONED_IN_PUBLICATION]->(p:Publication)
WHERE p.DOI is not null
RETURN count(*)
EOF
```

Use the `avantgraph` binary to run the query.
For example:

```bash
avantgraph --query-type=cypher /tmp/ckg /home/ubuntu/02-disease-properties.cypher
```

## Advanced Use Cases
You can also start AvantGraph in server mode and send queries to it from the
host machine.
Instructions are available
[here](https://github.com/avantlab/avantgraph#start-avantgraph-in-server-mode).

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

## Advanced Use Cases
You can also start AvantGraph in server mode and send queries to it from the
host machine.
Instructions are available
[here](https://github.com/avantlab/avantgraph#start-avantgraph-in-server-mode).
# 1\. Graph Data Description

The graph should be a file in json format which is based on a neo4j dump format (https://neo4j.com/labs/apoc/4.1/export/json/). The file is divided into lines by `\n`. On each line there is a json object `{…}` for one vertex or one edge. In this json object, there are key-value pairs. But key-value pairs are different between vertices and edges. Before any edge is declared, all vertices are declared.

For vertices:

- Every line contains an object with the following fields (in no particular order):
  * type : the value is ‘node’, determining the type of this object is a vertex.
  * id: an unique number describing this vertex. To be used in cross referencing within the file, does not need to be stored in the loaded graph. The value is a quoted string, but should be a valid unsigned decimal integer and can thus be parsed to a number before being used.
  * labels: this contains a list of strings, to be used as a special `labels` property.
  * properties: a json object containing arbitrary key-value pairs. The values can be any type of json object. A loader may choose to skip loading one or more property keys / value types.

For edges:

* Every line contains an object with the following fields (in no particular order):
  * type : the value is ‘relationship’, determining the type of this object is an edge.
  * id: an unique number describing this edge. To be used in cross referencing within the file, does not need to be stored in the loaded graph. The value is a quoted string, but should be a valid unsigned decimal integer and can thus be parsed to a number before being used.
  * label: the string to use as the edge label.
  * properties: A json object containing arbitrary key-value pairs. The values can be any type of json object. A loader may choose to skip loading one or more property keys / value types.
  * start: json objects with two keys of id and labels describing the from_vertex of edge.
    * id: the reference to a previously defined vertex, loaded with this id.
    * labels: the labels of the associated vertex which can be used to implement load-time filtering.
  * end: json objects with two keys of id and labels describing the to_vertex of edge.
    * id: the reference to a previously defined vertex, loaded with this id.
    * labels: the labels of the associated vertex whch can be used to implement load-time filtering.

## Example Graph Data:

Please refer to the following example graph data:

```json
{"type": "node", "id": "0", "labels": [], "properties": {}} 
{"type": "node", "id": "1", "labels": [], "properties": {}} 
{"type": "node", "id": "2", "labels": [], "properties": {}} 
{"type": "relationship", "id": "0", "start": {"id": "0", "labels": []}, "end": {"id": "1", "labels": []}, "label": "https://testdata.avantgraph.io/edge-label/0", "properties": {}} 
{"type": "relationship", "id": "1", "start": {"id": "2", "labels": []}, "end": {"id": "1", "labels": []}, "label": "https://testdata.avantgraph.io/edge-label/0", "properties": {}}
```

# 2\. Command for Loading Graph

There are a number of tools in AvantGraph:

* `ag-load-graph` loads a serialized graph into AvantGraph's internal format.
* `ag-query` runs query canonicalization / rewrite rules on queries.
* `avantgraph` is the main entrypoint of AvantGraph for running a query on a graph.
* `ag-index` lists, modifies and creates indexes on a graph.
* `ag-plan` canonicalizes and plans a query for a particular graph
* `ag-queryminer` extracts non-empty queries from a graph.
* `ag-schema` lists, modifies and creates property definitions.
* `ag-extract-schema` extracts schema information from a graph, (labels, properties, etc).
* `ag-export-graph` exports a graph in various serialization formats.
* `ag-exec` executes serialized execution plans against a given graph.

To load a specific graph, we can use the following example command:

`build/src/tools/load-graph/ag-load-graph --graph-format=json test/lit/data/graphs/issue-81.json output_graph`

where `build/src/tools/load-graph/ag-load-graph` is the specified `ag-load-graph` binary; `--graph-format=json` is optional specifying the format of graph; `test/lit/data/graphs/issue-81.json` is the input graph file; and `output_graph` is the directory created to store the loading graph, including internal graph representations for edges, vertices, indexes, properties, schema and strings.

To see other available options when loading a graph, we could run the following command `build/src/tools/load-graph/ag-load-graph -help`, which will show the generic options and `ag-load-graph` options, as shown in the following table.

| header | header |
|--------|--------|
| \--help | Display available options (--help-hidden for more) |
| \--help-list | Display list of available options (--help-list-hidden for more) |
| \--version | Display the version of this program |
| \--graph-format=json | The graph format to import |
| \--load-properties | Whether to load properties |
| \--report-progress | Whether to report progress |
| \--skip-invalid-lines | Whether to skip lines that could not be parsed |
| \--trace-filters=\<filters...\> | Filters to apply to the trace recording |
| \--trace-output= | Path to the profiling output file |
| \--use-antlr | Use antlr-based parser if available |
| \--validate-iris | Whether to validate the IRI format in N-Triple values |

# 3\. Format Convert

Suppose the original dataset is not shown as expected, we need to convert it to the format as shown in `section 1` before we load to AvantGraph.

Taking OpenAIRE Graph (https://zenodo.org/record/7490192) as an example, we convert the compressed json files into a format more suitable for AvantGraph. We implement `jq` as the converers, which is a lightweight and flexible command-line JSON processor (https://jqlang.github.io/jq/).

For instance, to convert the `communities_infrastructures.tar` to the suitable format,which includes json.gz files of research communities and research infrastructures, we create the `jq converter` as shown below

```json
{
    type: "node",
    id: .id,
    labels: [ "community" ],
    properties: {
        acronym: .acronym,
        description: .description,
        name: .name,
        subject: .subject,
        type: .type,
        zenodo_community: .zenodo_community,
   }
}
```

and apply it to convert the original json format

```json
{"acronym":"eut",
"description":"<p>EUt+ is an alliance of 8 universities: Technological University Dublin, Riga Technical University, Cyprus University of Technology, Technical University of Cluj-Napoca, Polytechnic University of Cartagena, University of Technology of Troyes, Technical University of Sofia and Hochschule Darmstadt.</p>",
"id":"00|context_____::808e822405561c05252ba37d71455548",
"name":"EUt+",
"type":"Research Community”}
```

to the following format.

```json
{"type":"node",
"id":"00|context_____::808e822405561c05252ba37d71455548",
"labels":["community"],
"properties":{"acronym":"eut","description":"<p>EUt+ is an alliance of 8 universities: Technological University Dublin, Riga Technical University, Cyprus University of Technology, Technical University of Cluj-Napoca, Polytechnic University of Cartagena, University of Technology of Troyes, Technical University of Sofia and Hochschule Darmstadt.</p>","name":"EUt+","subject":null,"type":"Research Community”,"zenodo_community":null}}
```
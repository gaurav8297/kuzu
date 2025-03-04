# Kùzu Navix

A Native Vector Index Design for Graph DBMSs With Robust and Fast Predicate-Agnostic Search Performance.

- Disk Based HNSW Index backed by Buffer Manager
- A novel Prefiltering-based predicate agnostic vector search which is fast and robust acorss various selectivities and correlation scenarios.
- Multi-Threaded Index Building
- Zero Copy Fast Distance Computations through buffer manager
- Easy to use as implemented in an embedded database

Our Paper for more info: https://cs.uwaterloo.ca/~ssalihog/papers/navix-tr.pdf

## Abstract

There is an increasing demand for extending existing DBMSs with
vector indices to become unified systems that can support modern predictive applications, which require joint querying of 
vector embeddings and structured properties and connections of objects. We present NaviX, a Native vector indeX for graph DBMSs
(GDBMSs) that has two main design goals. 

First, we aim to implement a disk-based vector index that leverages the core storage and
query processing capabilities of the underlying GDBMS. To this
end, NaviX is a hiearchical navigable small world (HNSW) index,
which is itself a graph-based structure. 

Second, we aim to evaluate predicate-agnostic vector search queries, where the k nearest neighbors (kNNs) of a 
query vector 𝑣𝑄 is searched across an arbitrary subset 𝑆 of vectors that is specified by an ad-hoc selection sub-query
𝑄𝑆 . We adopt a prefiltering-based approach that evaluates 𝑄𝑆 first and passes the full information about 𝑆 to the kNN 
search operator. We study how to design a pre-filtering-based search algorithm that
is robust under different selectivities as well as correlations of 𝑆
with 𝑣𝑄 . We propose an adaptive algorithm that utilizes local selectivity of each vector in the HNSW graph to pick a 
suitable heuristic at each iteration of the kNN search algorithm. We demonstrate
NaviX’s robustness and efficiency through extensive experiments against both existing prefiltering- and 
postfiltering-based baselines that include specialized vector databases (Weaviate and Milvus) as well as DBMSs 
(PGVectorScale and VBase).


Example Predicate Agnostic Search Query:
```sql
-- Simple Filtering Query
MATCH (p:PersonChunk)
p.birth_date < date('1975-01-01')
CALL ANN_SEARCH(e.embedding, [0.1, ...], <K>, <efS>, <bool_enable_brute_force_knn>)
RETURN c.name;

-- Join Query for use case such as graph rag (https://blog.langchain.dev/enhancing-rag-based-applications-accuracy-by-constructing-and-leveraging-knowledge-graphs/)
MATCH (p:Person)-[e:PersonChunk]->(c:Chunk)
WHERE p.birth_date >= date('1971-01-01') AND p.birth_date < date('1975-01-01')
CALL ANN_SEARCH(c.embedding, [0.1, ...], <K>, <efS>, <bool_enable_brute_force_knn>)
RETURN c.id;
```

## Build from Source

```bash
$ git clone https://github.com/gaurav8297/kuzu.git
$ cd kuzu
$ make release NUM_THREADS=32
```

Build on MacOS: Use this gcc https://formulae.brew.sh/formula/gcc
```bash
$ export CXX=/opt/homebrew/bin/g++-14
$ export CC=/opt/homebrew/bin/gcc-14
$ make release NUM_THREADS=32
```

[//]: # ()
[//]: # (_Note: We evaluated our system on 32 CPU cores with an Intel&#40;R&#41; Xeon&#40;R&#41; Platinum 8260 CPU and a memory bandwidth)

[//]: # (of 110GB/s, a configuration commonly available on cloud providers such as AWS. High memory bandwidth is crucial for )

[//]: # (the prefiltering phase performance, as KuzuDB is backed by columnar storage and performs extensive scanning operations )

[//]: # (like any other analytical database._)

## Ingest Data & Build Index

Kuzu support multiple ways to ingest data into the database. Take a look at the [Import Data](https://docs.kuzudb.com/import/) for more information.
<br>
For this example, we will import the SIFT10K using the parquet file format.
<br>
```bash
# Download Data
$ wget https://huggingface.co/datasets/gaurav8297/navix_dataset

# Start Kùzu Shell and Build Index
$ ./build/release/tools/shell/kuzu /path/to/kuzu/data

> CREATE NODE TABLE sift (id INT64, embedding FLOAT[128], PRIMARY KEY (id));
> CREATE VECTOR INDEX ON sift.embedding (efConstruction=200, maxNbrsAtUpperLevel=32, maxNbrsAtLowerLevel=64, distanceFunc='L2');
> call threads=32; # Set the number of threads for index building (Optional)
> UPDATE VECTOR INDEX ON sift.embedding;
```

## Query Data

Once the index is built, you can query the data using the following query template:
```sql
MATCH (s:dataset)
WHERE <filtering_condition>
CALL ANN_SEARCH(s.<embedding_property>, [0.1, ...], <K>, <efS>, <bool_enable_brute_force_knn>)
RETURN s.<property>;
```
Another example query with one hop graph traversal:
```sql
MATCH (p:Person)-[e:PersonChunk]->(c:Chunk)
WHERE <filtering_condition>
CALL ANN_SEARCH(c.<embedding_property>, [0.1, ...], <K>, <efS>, <bool_enable_brute_force_knn>)
RETURN c.<embedding_property>;
```

Now coming back to our sift dataset, let's query the data:
```bash
# Simple vector search query
> MATCH (s:sift)
> CALL ANN_SEARCH(s.embedding, [0.1, 0.1, ...], 10, 64, false)
> RETURN s.id;

# Vector search query with filtering
> MATCH (s:sift)
> WHERE s.id < 300000
> CALL ANN_SEARCH(s.embedding, [0.1, 0.1, ...], 10, 64, false)
> RETURN s.id;
```

[//]: # (Note: The first query might be a bit slow compared to subsequent queries as the data is not cached in buffer manager.)

## Contact Us
You can contact us at [g3sehgal@uwaterloo.ca](mailto:g3sehgal@uwaterloo.ca).

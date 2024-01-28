# AeonG
## Description
AeonG is a temporal graph database that efficiently supports
temporal features based on [Memgraph](https://memgraph.com). We provide a formally defined temporal property graph model,
based on which we fundamentally design AeonG with a temporal-enhanced storage engine and query
engine. AeonG utilizes a hybrid storage engine, in which we introduce a current store for maintaining
current graphs and a historical store for storing historical graphs migrated from the current storage under
MVCC management. Furthermore, AeonG equips a native temporal query engine to process temporal
queries with efficient capability and data consistent guarantee.

## Contributions

- Fast querying capabilities over subgraphs at a past time point or range
- Small storage overhead of historical data
- Native support of transaction time
- ACID compliance


## Getting Started

### Build System Dependencies
You can follow the detailed information provided by [Memgraph](https://memgraph.notion.site/Quick-Start-82a99a85e62a4e3d89f6a9fb6d35626d) to build system dependencies. We also provide a docker image to simplify this step. We recommend using docker to build AeonG.

    docker pull hououou/aeong:v1

    docker run -it -p 7687:7687 -p 7444:7444 --mount type=bind source=$pwd,target=/home/ --entrypoint bash aeong

### Install libraries
Before compiling AeonG, you should activate the toolchain, which is  used our own custom toolchain.

    source /opt/toolchain-vXYZ/activate

Apart from the system-wide installed dependencies, AeonG needs some libraries to be built locally. The proper setup of these libraries should be checked by running the init script.

    cd aeong
     ./init

### Compile
With all of the dependencies installed and the build environment set-up, you need to configure the build system. To do that, execute the following:

    mkdir -p build
    cd build
    cmake ..

If everything went OK, you can now, finally, run build AeonG binary and client binary.

    make -j$(nproc) memgraph
    cd tests/mgbench
    make 

### Run

After the compliation, you can run AeonG

    ./memgraph


## Benchmarks
We support three temporal benchmarks to evaluate the performances. More details can be found in [tests/benchmarks/README.md](/tests/benchmarks/README.md)

* We can automatically generate graph operation query statements for generating temporal data.

        cd tests/benchmarks/$workloadname
        python create_graph_op_quries.py --arg $arg_value

* We can generate temporal query statements for evaluating temporal performance.

        cd tests/benchmarks/$workloadname
        python create_temporal_query.py --arg $arg_value

## Tools
We provide tools to create temporal database and evaluate temporal database. Those tools can be found in [srcipt directory](/tests/scripts/).

### Create temporal database
It will report the average graph operation query latency and the storage consumptions of generated temproal database.

    cd tests/scripts/
    python3 create_temporal_database.py

You can specify optional arguments to generate the desirable temporal database.  You can also check specific arguments in each workload.

    python create_temporal_database.py --help

| Flag | Description | 
|----------|----------|
|--aeong-binary|AeonG binary|
|--client-binary|client binary|
|--num-workers|number of workers |
|--data-directory|directory path where temporal data should be stored|
|--original-dataset-cypher-path|directory path where temporal data should be stored|
|--index-cypher-path|index query path|
|--graph-operation-cypher-path|directory path where temporal data should be stored|
|--no-properties-on-edges|disable properties on edges|

### Evaluate temporal query performance
It will report the average temproal query latency.

    cd tests/scripts/
    python3 evaluate_temporal_q.py

The arguments are almost same to the above create_temporal_database.p, except "--temporal-query-cypher-path" indicating temporal query path. You can specify optional arguments to generate the desirable temporal database.  You can also check specific arguments in each workload.

    python evaluate_temporal_q.py --help

## Run experiments
We also provide scripts to evaluate the performance of AeonG on T-mgBench benchmark. You can also edit following scripts to test other benchmarks.

### 1. Storage consumption & graph operation performance
* Download datasets of temporal workloads. Downloaded datasets will be shown in the [dataset directory](/tests/datasets/). The datasets will be downloaded in the /tests/datasets/T-mgBench directory.

        cd tests/scripts
        ./download_datasets.sh
* Generate graph operation query statements. The generated results will be stored in /tests/results/graph_op directory.

        cd tests/scripts
        ./generate_graph_op.sh 

* Create temporal graph database. The temporal databse will be generated in /tests/results/database directory. Additionally, it will report the average graph operation query latency and the storage consumptions of generated temproal database.

        cd tests/scripts
        python3 create_temporal_database.py


### 2. Temporal query performance
* Generate temporal graph database as above steps.

        cd tests/scripts
        ./download_datasets.sh
        ./generate_graph_op.sh 
        python3 create_temporal_database.py

* Generate temporal query statements. The generated results will be stored in /tests/results/temporal_query directory.

        ./generate_temporal_q.sh 

* Evaluate temproal query performance using the follwing command. It will report the average temproal query latency.

        python3 evaluate_temporal_q.py

## AeonG Implementation
AeonG is extended based on Memgraph. The details of our idea can be found in our [paper](https://hououou.github.io/AeonG/docs/aeong-extended-version-vldb24.pdf). We also refer Memgraph [internal docs](https://memgraph.notion.site/Memgraph-Internals-12b69132d67a417898972927d6870bd2) to better understand our codes. We made following major changes to support temporal features.

* storage engine:
  * timestamps: import timestamps into [Vertex](./src/storage/v2/vertex.hpp), [Edge](./src/storage/v2/edge.hpp), and [Delta](./src/storage/v2/delta.hpp) strutures.
  * data migaration: add data migration in the [Storage::CollectGarbage()](.src/storage/v2/storage.cpp) to migrate unreclaiemd data to RocksDB
  * retain historical data in RocksDB: [historical_delta.cpp](./src/storage/v2/history_delta.cpp) is used to transfer deltas to key-value formats and store them to RocksDB.
* query engine:
  * add temporal syntax in [Cypher.g4](./src/query/frontend/opencypher/grammar/Cypher.g4).
  * extend scan operator:   In the
    [ScanAllCursor.Pull()](./src/query/plan/operator.cpp) function, we introduce a function AddHistoricalVertices() to capture both unreclaimed and reclaimed historical versions
  * extend scan operator:   In the
    [ ExpandCursor.Pull()](./src/query/plan/operator.cpp) function, we introduce a function AddHistoricalEdges() to capture both unreclaimed and reclaimed historical versions



## Configuration settings
We inherit the configuration from Memgraph. Thus we support all configuration described in Memgraph. We refer the details in this [link](https://memgraph.com/docs/configuration/configuration-settings). Additionally, AeonG supports two more configurations to provide temporal features.

### General Settings

| Flag | Description | 
|----------|----------|
| --bolt-port       | Port on which the Bolt server should listen.       |
|--data-directory|Path to directory in which to save all permanent data.|
| --data-recovery-on-startup | Facilitates recovery of one or more individual databases and their contents during startup. Replaces --storage-recover-on-startup|
| --storage-gc-cycle-sec       | Storage garbage collector interval (in seconds).       | 
|--storage-recover-on-startup| Deprecated and replaced with the data_recovery_on_startup flag. Controls whether the storage recovers persisted data on startup. |
|--storage-properties-on-edges| Controls whether edges have properties. |
|--storage-snapshot-interval-sec| Storage snapshot creation interval (in seconds). Set to 0 to disable periodic snapshot creation. |
|--storage-snapshot-retention-count| The number of snapshots that should always be kept.| 

### AeonG specification

| Flag | Description | Default |
|----------|----------|----------|
| --retention-period-sec       | Reclaim history period (in seconds). Set to 0 to disable reclaiming history from historical storage.       | 0       |
| --retention-cycle-sec       | Reclaim history interval (in seconds).       | 60       |
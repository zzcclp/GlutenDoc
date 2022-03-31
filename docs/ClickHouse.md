## ClickHouse Backend

ClickHouse is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP), which supports best in the industry query performance, while significantly reducing storage requirements through its innovative use of columnar storage and compression.
We port ClickHouse ( based on version **21.9.1.1** ) as a library, called 'libch.so', and Gluten loads this library through JNI as the native engine. In this way, we don't need to  deploy a standalone ClickHouse Cluster, Spark uses Gluten as SparkPlugin to read and write ClickHouse MergeTree data.

### Architecture

The architecture of the ClickHouse backend is shown below:

![ClickHouse-Backend-Architecture](./image/ClickHouse/ClickHouse-Backend-Architecture.png)

1. On Spark driver, Spark uses Gluten SparkPlugin to transform the physical plan to the Substrait plan, and then pass the Substrait plan to ClickHouse backend through JNI call on executors.
2. Based on Spark DataSource V2 interface, implementing a ClickHouse Catalog to support operating the ClickHouse tables, and then using Delta to save some metadata about ClickHouse like the MergeTree parts information, and also provide ACID transactions.
3. When querying from a ClickHouse table, it will fetch MergeTree parts information from Delta metadata and assign these parts into Spark partitions according to some strategies.
4. When writing data into a ClickHouse table, it will use ClickHouse library to write MergeTree parts data and collect these MergeTree parts information after writing successfully, and then save these MergeTree parts information into Delta metadata. ( **The feature of writing MergeTree parts is coming soon.** )
5. On Spark executors, each executor will load the 'libch.so' through JNI when starting, and then call the operators according to the Substrait plan which is passed from Spark Driver, like reading data from the MergeTree parts, writing the MergeTree parts, filtering data, aggregating data and so on.
6. Currently, the ClickHouse backend only supports reading the MergeTree parts from local storage, it needs to use a high-performance shared file system to share a root bucket on every node of the cluster from the object storage, like JuiceFS.


### Development environment setup

In general, we use IDEA for Gluten development and CLion for ClickHouse backend development on **Ubuntu 20**.

#### Prerequisites

- GCC 9.0 or higher version
```
    sudo apt install gcc-9 g++-9 gcc-10 g++-10 gcc-11 g++-11  

    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 110 --slave /usr/bin/g++ g++ /usr/bin/g++-11 --slave /usr/bin/gcov gcov /usr/bin/gcov-11
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 100 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 90 --slave /usr/bin/g++ g++ /usr/bin/g++-9 --slave /usr/bin/gcov gcov /usr/bin/gcov-9

    sudo update-alternatives --config gcc  # then choose the right version
    gcc --version  # check the version of the gcc

```

- Clang 12.0 or higher version ( Please refer to [How-to-Build-ClickHouse-on-Linux](https://clickhouse.com/docs/en/development/build/) )

    Install Clang 12.0 by apt manually.
```
    wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key|sudo apt-key add -

    sudo vi /etc/apt/sources.list

    ### add these sources into sources.list
    # for 12
    deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic-12 main
    deb-src http://apt.llvm.org/bionic/ llvm-toolchain-bionic-12 main

    sudo apt update
    sudo apt install -y clang-12 lldb-12 lld-12 clang-12-doc llvm-12-doc llvm-12-examples clang-tools-12 libclang-12-dev clang-format-12 libfuzzer-12-dev libc++-12-dev libc++abi-12-dev libllvm-12-ocaml-dev

    sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-12 100 --slave /usr/bin/clang++ clang++ /usr/bin/clang++-12

    sudo update-alternatives --config clang  # choose the clang-12
    clang --version  # check the version of the clang
```
- cmake 3.20 or higher version ( Please refer to [How-to-Build-ClickHouse-on-Linux](https://clickhouse.com/docs/en/development/build/) )
- Java 8
- Maven 3.6.3 or higher version
- Spark 3.1.1
- Intel Optimized Arrow 7.0.0 ( Please refer to [Intel-Optimized-Arrow-Installation](./ArrowInstallation.md) )


#### Setuping Gluten development environment

- Clone Gluten code
```
    git clone https://github.com/oap-project/gluten
```
- Open Gluten code in IDEA

#### Setuping ClickHouse backend development environment

- Clone ClickHouse backend code
```
    git clone -b local_engine_with_columnar_shuffle https://github.com/liuneng1994/ClickHouse.git
```
- Open ClickHouse backend code in CLion
- Configure the ClickHouse backend project
    - Choose File -> Settings -> Build, Execution, Deployment -> Toolchains, and then choose Bundled CMake, clang-12 as C Compiler, clang++-12 as C++ Compiler:

        ![ClickHouse-CLion-Toolchains](./image/ClickHouse/CLion-Configuration-1.png)

    - Choose File -> Settings -> Build, Execution, Deployment -> CMake:

        ![ClickHouse-CLion-Toolchains](./image/ClickHouse/CLion-Configuration-2.png)

        And then add these options into CMake options:
```
            -G "Unix Makefiles" -D WERROR=OFF -D ENABLE_PROTOBUF=1 -D ENABLE_JEMALLOC=0
```
- Build 'ch' target with Debug mode or Release mode:

    ![ClickHouse-CLion-Toolchains](./image/ClickHouse/CLion-Configuration-3.png)

    - If it builds with Debug mode successfully, there is a library file called 'libchd.so' in path 'cmake-build-debug/utils/local-engine/'.
    - If it builds with Release mode successfully, there is a library file called 'libch.so' in path 'cmake-build-release/utils/local-engine/'.


### Compiling Gluten with ClickHouse backend

The prerequisites are the same as the one above mentioned. Compile Gluten with ClickHouse backend through maven:
```
    git clone https://github.com/oap-project/gluten
    cd gluten/
    export MAVEN_OPTS="-Xmx8g -XX:ReservedCodeCacheSize=2g"
    mvn clean install -Phadoop-2.7.4 -Pspark-3.1.1 -Dhadoop.version=2.8.5 -Pclickhouse-lib -DskipTests -Dbuild_cpp=ON -Dcpp_tests=OFF -Dbuild_clickhouse_lib=ON -Dbuild_arrow=ON -Dbuild_protobuf=ON -Dbuild_jemalloc=ON -Dcheckstyle.skip
    ls -al jvm/target/gazelle-jni-jvm-XXXXX-jar-with-dependencies.jar
```


### Testing on local

#### Deploying Spark 3.1.1
```
tar -zxvf spark-3.1.1-bin-2.8.5.tgz
cd spark-3.1.1-bin-2.8.5
rm -f jars/arrow-* ./
rm -f jars/protobuf-java-2.5.0.jar ./
rm -f jars/flatbuffers-java-1.9.0.jar ./
#download protobuf-java-3.13.0.jar and flatbuffers-java-1.12.0.jar from maven
cp protobuf-java-3.13.0.jar jars/
cp flatbuffers-java-1.12.0.jar jars/
```

#### Data preparation
Currently, the feature of writing ClickHouse MergeTree parts by Spark is developing, so it needs to use command 'clickhouse-local' to generate MergeTree parts data manually. We provide a python script to call the command 'clickhouse-local' to convert parquet data to MergeTree parts:
```
#install ClickHouse community version
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client

#generate MergeTree parts
mkdir -p /path_clickhouse_database/table_path/
python3 ./parquet_to_mergetree.py --path=/tmp --source=/path_to_parquet_data/tpch-data-sf100/lineitem --dst=/path_clickhouse_database/table_path/lineitem
```
##### **This python script will convert one parquet data file to one MergeTree parts.**


#### Running Spark Thriftserver on local
```
cd spark-3.1.1-bin-2.8.5
./sbin/start-thriftserver.sh \
  --master local[3] \
  --driver-memory 10g \
  --conf spark.driver.extraJavaOptions="-XX:+UseG1GC -XX:G1HeapRegionSize=32M -XX:+ExplicitGCInvokesConcurrent -XX:+ExitOnOutOfMemoryError -XX:InitiatingHeapOccupancyPercent=45" \
  --conf spark.driver.memoryOverhead=6G \
  --conf spark.serializer=org.apache.spark.serializer.JavaSerializer \
  --conf spark.memory.fraction=0.3 \
  --conf spark.memory.storageFraction=0.3 \
  --conf spark.sql.sources.ignoreDataLocality=true \
  --conf spark.default.parallelism=1 \
  --conf spark.sql.shuffle.partitions=1 \
  --conf spark.sql.files.minPartitionNum=1 \
  --conf spark.sql.files.maxPartitionBytes=1073741824 \
  --conf spark.sql.adaptive.enabled=false \
  --conf spark.sql.parquet.filterPushdown=true \
  --conf spark.sql.parquet.enableVectorizedReader=true \
  --conf spark.locality.wait=0 \
  --conf spark.locality.wait.node=0 \
  --conf spark.locality.wait.process=0 \
  --conf spark.sql.columnVector.offheap.enabled=true \
  --conf spark.memory.offHeap.enabled=true \
  --conf spark.memory.offHeap.size=6442450944 \
  --conf spark.plugins=com.intel.oap.GazellePlugin \
  --conf spark.oap.sql.columnar.columnartorow=false \
  --conf spark.oap.sql.columnar.loadnative=true \
  --conf spark.oap.sql.columnar.libpath=/path_to_clickhouse_library/libch.so \
  --conf spark.oap.sql.columnar.iterator=false \
  --conf spark.oap.sql.columnar.loadarrow=false \
  --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.execution.datasources.v2.clickhouse.ClickHouseSparkCatalog \
  --conf spark.databricks.delta.maxSnapshotLineageLength=20 \
  --conf spark.databricks.delta.snapshotPartitions=1 \
  --conf spark.databricks.delta.properties.defaults.checkpointInterval=5 \
  --conf spark.databricks.delta.stalenessLimit=3600000

#connect to Spark Thriftserver by beeline
bin/beeline -u jdbc:hive2://localhost:10000/ -n root
```

#### Testing

- Create a TPC-H lineitem table using ClickHouse DataSource

```
    DROP TABLE IF EXISTS lineitem;
    CREATE TABLE IF NOT EXISTS lineitem (
     l_orderkey      bigint,
     l_partkey       bigint,
     l_suppkey       bigint,
     l_linenumber    bigint,
     l_quantity      double,
     l_extendedprice double,
     l_discount      double,
     l_tax           double,
     l_returnflag    string,
     l_linestatus    string,
     l_shipdate      date,
     l_commitdate    date,
     l_receiptdate   date,
     l_shipinstruct  string,
     l_shipmode      string,
     l_comment       string)
     USING clickhouse
     TBLPROPERTIES (engine='MergeTree'
                    )
     LOCATION '/path_clickhouse_database/table_path/lineitem';
```

- TPC-H Q6 test

```
    SELECT
        sum(l_extendedprice * l_discount) AS revenue
    FROM
        lineitem_ch
    WHERE
        l_shipdate >= date'1994-01-01'
        AND l_shipdate < date'1994-01-01' + interval 1 year
        AND l_discount BETWEEN 0.06 - 0.01 AND 0.06 + 0.01
        AND l_quantity < 24;
```

- Result

    The DAG is shown on Spark UI as below:
    
    ![ClickHouse-CLion-Toolchains](./image/ClickHouse/Gluten-ClickHouse-Backend-Q6-DAG.png)


### Benchmark with TPC-H Q6 on Gluten with ClickHouse backend




#### Deploying on Cloud





#### Performance


Below table shows the TPC-H Q6 Performance in a multiple-thread test (--num-executors 6 --executor-cores 6) for Velox and vanilla Spark.
Both Parquet and ORC datasets are sf1024.

| TPC-H Q6 Performance | Velox (ORC) | Vanilla Spark (Parquet) | Vanilla Spark (ORC) |
| ---------- | ----------- | ------------- | ------------- |
| Time(s) | 13.6 | 21.6  | 34.9 |












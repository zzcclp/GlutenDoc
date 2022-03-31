## ClickHouse Backend

ClickHouse is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP), which supports best in the industry query performance, while significantly reducing storage requirements through its innovative use of columnar storage and compression.
We port ClickHouse ( based on version **21.9.1.1** ) as a library, called 'libch.so', and Gluten loads this library through JNI as the native engine. In this way, we don't need to  deploy a standalone ClickHouse Cluster, Spark uses Gluten as SparkPlugin to read and write ClickHouse MergeTree data.

### Architecture

The architecture of the ClickHouse backend is shown below:

![ClickHouse-Backend-Architecture](./image/ClickHouse/ClickHouse-Backend-Architecture.png)

1. On Spark driver, Spark uses Gluten SparkPlugin to transform the physical plan to the Substrait plan, and then pass the Substrait plan to ClickHouse backend through JNI call on executors.
2. Based on Spark DataSource V2 interface, implementing a ClickHouse Catalog to support operating the ClickHouse tables, and then using Delta to save some metadata about ClickHouse like the MergeTree parts information, and also provide ACID transactions.
3. When querying from a ClickHouse table, it will fetch MergeTree parts information from Delta metadata and assign these parts into Spark partitions according to some strategies.
4. On Spark executors, each executor will load the 'libch.so' through JNI when starting, and then call the operators according to the Substrait plan which is passed from Spark Driver, like reading data from the MergeTree parts, writing the MergeTree parts, filtering data, aggregating data and so on.
5. Currently, the ClickHouse backend only supports reading the MergeTree parts from local storage, it needs to use a high-performance shared file system to share a root bucket on every node of the cluster from the object storage, like JuiceFS.


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

```

- Clang 12.0 or higher version ( Please refer to [How-to-Build-ClickHouse-on-Linux](https://clickhouse.com/docs/en/development/build/) )
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





### Testing on local





### Benchmark with TPC-H Q6 on Gluten with ClickHouse backend



#### Deploying on Cloud




#### Results



#### Performance



##### Result

![TPC-H Q6](./image/TPC-H_Q6_DAG.png)

##### Performance

Below table shows the TPC-H Q6 Performance in a multiple-thread test (--num-executors 6 --executor-cores 6) for Velox and vanilla Spark.
Both Parquet and ORC datasets are sf1024.

| TPC-H Q6 Performance | Velox (ORC) | Vanilla Spark (Parquet) | Vanilla Spark (ORC) |
| ---------- | ----------- | ------------- | ------------- |
| Time(s) | 13.6 | 21.6  | 34.9 |












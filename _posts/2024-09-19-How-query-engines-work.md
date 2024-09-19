---
layout: post
title: "How query engines work"
description: "How query engines work."
date: 2024-09-19
feature_image: images/posts/2024-09-19-How-query-engines-work-logo.jpg
tags: [bigdata]
---
Recently, I delved into Apache Ballista, a distributed computing platform. In the blog of its creator, Andy Grove, I discovered the book [_How Query Engines Work_](https://howqueryengineswork.com/00-acknowledgments.html), which provided me with deeper insights into the motivations behind the creation of this tool. It also helped me understand what Apache Arrow and DataFusion are, and how they are interconnected. Overall, I gained a solid understanding of how modern query engines are designed. While reading this book, I took notes that I now intend to share. This will not be an outline of the book. Article will focus on the main ideas, supplemented with information from other sources.

<!--more-->

# What is a query engine?
Query engines are the heart of data analysis, allowing us to extract meaningful insights from vast amounts of information. They convert high-level user queries (usually written in a language like SQL) into a series of steps that a computer can execute to fetch and manipulate the data. These steps involve finding the data, filtering it, sorting it, and combining it in specific ways. The engine then executes these steps on the underlying data storage systems (like databases, data lakes or message queues) to produce the desired output.

Think of a query engine like a chef in a kitchen: you (the user) give it an order (a query), and it prepares the meal (the result) by gathering ingredients (data) and following a recipe (query execution).

There are different types of query engines, such as **relational query engines** (used in relational databases like MySQL, Postgres), **big data query engines** (used in distributed data systems like Spark, Hive, Ballista), and **streaming query engines** (used for real-time data processing).

# Structure of a query engine
To begin with, I'll outline how a query engine operates at a high level, and then I'll delve into each component individually.
Here's a flow:
1. The client submits query to the engine.
2. The query is parsed into a logical plan.
3. After that, the logical plan is optimized.
4. The query planner than takes the optimized logical plan and creates a physical plan.
5. Further, executors, based on the physical plan, retrieve data from data sources and execute the query on this data.

For simplicity, let's assume that the client writes the query in the familiar SQL format. However, query engines do not process SQL queries directly within themselves. So, what does a query look like inside the engine?

# Logical plan
A **logical plan** is a high-level representation of a query's execution strategy.

When you write a query (e.g., `SELECT * FROM orders WHERE price > 100`), the query engine doesn't just execute it directly; it first needs to understand what steps are involved to get the correct result. A logical plan breaks down the query into a series of high-level operations (like filtering, sorting, or joining data) that describe what needs to be done, without specifying how to do it. You can think of a logical plan as a directed acyclic graph (DAG) of logical plan nodes, with each node representing a specific operation. A logical plan is abstract and does not consider the specific data storage or execution details.

It is like a recipe that a query engine creates to decide how to execute a query.

## Logical plan structure
Logical plans represent operations like **Scan**, **Projection**, **Selection**, and **Aggregate**. Each operation is represented by a node in the plan, defining the structure of query processing.

### Scan
The **scan** logical plan represents the fundamental operation of fetching data from a data source, with an optional projection, serving as the starting point for all queries in the query engine.
### Projection
**Projection** applies a set of expressions to incoming data, generating a new set of attributes. This can be as simple as selecting specific columns (e.g., `SELECT a, b, c FROM foo`) or involve more complex expressions, such as calculations or type conversions (e.g., `SELECT (CAST(a AS float) * 3.141592)) AS foo_float FROM foo`).
### Selection (Filter)
The **selection** logical plan filters rows based on a condition. This is represented by the `WHERE` clause in SQL.
### Aggregate
The **aggregate** logical plan calculates aggregate values (e.g., sum, min, max, average) for groups of data. A simple example would be `SELECT job_title, AVG(salary) FROM employee GROUP BY job_title`.

# Parser
Now, let's go back to the point where we need to transform the client's SQL query into a logical plan. This task is handled by the parser. The main steps of the parser are:
- **Tokenization**:  Breaking down the SQL string into individual tokens (keywords, identifiers, operators).
- **Parsing**:  Analyzing the token sequence to understand the query's structure and meaning.

# Physical plan
While logical plans define what to do, **physical plans** specify how to do it. They provide detailed instructions on how to execute the query, including:
* **Data access methods**: How to retrieve data from the source (e.g., full table scan, index scan).
* **Join algorithms**: Which algorithm to use for joining tables (e.g., hash join, sort-merge join).
* **Data partitioning and distribution**: How data is divided and distributed across multiple nodes in a distributed environment.

It's better to keep logical and physical plans separate because there can be multiple ways to execute a particular operation. For example, there could be separate physical plans for single-process and distributed execution, or CPU and GPU execution.

Now we have a basic understanding of what a logical plan and a physical plan are. However, it's still unclear who creates them and how they function under the hood. Let’s start by clarifying the first question.

# Query planner
The **query planner** is the brain of the query engine. It takes the logical plan as input and generates an optimized physical plan. The query planner may choose different physical plans based on configuration options or based on the target platform's hardware capabilities. The planner's goal is to choose the physical plan with the lowest estimated cost, leading to the fastest and most resource-efficient query execution.

Ok, let's move to the next block and dive deeper under the hood of these processes. First of all we need to get data somewhere.

# Data sources
Query engines need to interact with diverse data sources, such as databases, files, and cloud storage. To make it easier for users and developers, many query engines provide a uniform interface to interact with different types of data sources. This means you can run the same query across multiple data sources without worrying about the underlying differences.

There are some challenges with data sources:
- **Data Format Compatibility**: The engine must be able to understand and convert various data formats like CSV, JSON, Parquet, and Avro into its own internal format for processing.
- **Network Latency**: Accessing data over the network (e.g., from cloud storage or remote databases) can introduce delays, so the query engine needs strategies to minimize this.
- **Schema Evolution**: The structure of the data (its schema) may change over time, and the query engine must handle these changes gracefully.

To work with data from a data source internally, a query engine needs to represent this data in a format that is convenient for its operations. So, let's talk a bit more about it.

# Query engine's type system
A query engine's type system determines how it represents and manipulates data. When supporting multiple data sources, the type system must be capable of accommodating various data types from different sources.

When designing a query engine, an important consideration is whether it will process data row-by-row or use a columnar format.

In many modern query engines each step in the physical plan acts as an iterator over rows. While this model is straightforward to implement, it often introduces overhead for each row processed. These overheads will be catastrophic in the world of big data.

A more efficient approach is to process data in batches rather than one row at a time. If these batches are organized in a columnar format, "vectorized processing" can be used to leverage SIMD (Single Instruction Multiple Data) instructions. This allows the CPU to process multiple values in a column with a single instruction. Additionally, this concept can be extended by using GPUs to handle even larger amounts of data in parallel.

Since we're exploring the architecture of query engines within the context of Apache Ballista, a tool designed for big data processing, it's easy to guess that Ballista uses a columnar data format. Ballista achieves this by leveraging Apache Arrow.

The core idea behind Arrow is to provide a standardized, in-memory, columnar format optimized for vectorized processing, in order to eliminate redundant serialization and deserialization when transferring data between systems. The Arrow columnar format includes a language-agnostic in-memory data structure specification, metadata serialization, and a protocol for serialization and generic data transport.

# DataFrames
**DataFrames** in query engines serve as a powerful abstraction that simplifies the process of querying and manipulating data. They provide a user-friendly interface for building logical query plans.

A DataFrame is a data structure that represents data in a table-like format, similar to a spreadsheet. It consists of rows (each representing a record) and columns (each representing a specific attribute or field of the data). Data Frames offer a high-level API for performing various operations such as filtering, sorting, grouping, joining, and aggregating data using a simple and intuitive syntax, without worrying about the underlying data representation or storage.

DataFrames abstract away the complexity of data storage and processing, letting users focus on the “what” rather than the “how” of data operations.

Under the hood, DataFrames in query engines are typically built on top of logical plans and physical plans. When you perform an operation on a DataFrame (like filtering or joining), it is translated into a series of operations in the logical plan, which the engine then optimizes and executes.

# Query optimizations
I’d like to highlight another important feature of query engines that helps conserve resources during query execution - query optimization. The logical plans provided by users are not always the most efficient representations of queries. Before selecting the appropriate physical plan based on the logical one, the query engine performs optimization on the logical plan.

I won’t go into detail about every type of optimization, but I will briefly outline the types of optimizations and methodologies to give you a general idea of what happens during the query optimization phase.

Optimizations are categorized into two types: logical and physical.

**Logical optimizations** focus on transforming the logical query plan without considering the physical implementation details. These optimizations typically involve:
* **Predicate pushdown**:  Pushing filter conditions down closer to the data source to reduce the amount of data processed. For example, filtering data early when reading it from a database or file, rather than after loading all the data.
* **Projection pushdown**:  Selecting only the necessary columns early in the plan to avoid unnecessary data transfer.
* **Constant folding**:  Precomputing constant expressions during planning. For example, `SELECT 2 + 2` would be optimized to `SELECT 4`.
* **Simplification of expressions**: Rewriting expressions to simpler or more efficient forms, such as turning `x * 2` into `x + x`.
* **Join reordering**:  Changing the order of joins to minimize intermediate data.
* **Subquery flattening**: Converts nested subqueries into simpler forms that are easier to optimize and execute.

**Physical optimizations** occur after logical optimizations and deal with the actual implementation of the query plan. They involve:
* **Choosing the right join algorithm**:  Selecting the most efficient join method based on data characteristics.
* **Index usage**:  Leveraging indexes to speed up data access.
* **Parallel execution**:  Distributing the workload across multiple CPU cores or machines.
* **Vectorized processing**:  Processing data in batches rather than row-by-row. This takes advantage of modern CPU features, making data processing faster.
* **Caching**:  Storing intermediate results to avoid recomputation.

Logical and physical optimizations represent different phases of the process, but optimizations can also be categorized by methodology. In this case, we distinguish between rule-based and cost-based approaches.

**Rule-based optimization** applies predefined rules to transform the logical plan. These rules are based on common sense and best practices.

**Cost-based optimization** relies on a cost model to estimate the cost of different execution plans. The planner chooses the plan with the lowest estimated cost.

# Parallel and distributed query execution
Since the query engine is typically designed to handle a large number of users and vast amounts of data, distributed query execution is essential. The simplest approach is parallel query execution.

**Parallel query execution** involves utilizing multiple CPU cores on a single machine to execute parts of the query concurrently. Here are a few concepts that make parallel query execution possible:
- **Partitioning** is a fundamental technique used in parallel query execution. Data is divided into smaller, non-overlapping subsets called partitions. Partitions can be processed independently and in parallel. Common partitioning strategies include range partitioning, hash partitioning, and list partitioning

- **Embarrassingly parallel operators**. Some operators can run in parallel on partitions without significant overhead. Projection and Filter operators are prime examples. These operators can be applied independently to each input partition, producing corresponding output partitions.

- **Parallel aggregation**. Partial aggregations are performed on individual partitions. Results are then combined to produce the final aggregate.

When dealing with large datasets (terabytes or petabytes of data), processing on a single machine is impractical because of the limitations in memory, CPU, and storage. **Distributed query execution** extends concept of parallel query execution to a cluster of machines, allowing for processing massive datasets that wouldn't fit on a single node.

To somewhat over-simplify the concept of distributed query execution, the goal is to be able to create a physical query plan which defines how work is distributed to a number of "executors" in a cluster. Distributed query plans will typically contain new operators that describe how data is exchanged between executors at various points during query execution.

As the query requires coordination across executors, we need to build a **scheduler** (coordinator). The concept of a distributed query scheduler is not complex at a high level. The scheduler needs to examine the whole query and break it down into stages that can be executed in isolation (usually in parallel across the executors) and then schedule these stages for execution based on the available resources in the cluster. Once each query stage completes then any subsequent dependent query stages can be scheduled. This process repeats until all query stages have been executed.

The scheduler could also be responsible for managing the compute resources in the cluster so that extra executors can be started on demand to handle the query load.

Key components in distributed query processing:
- **Scheduler (coordinator)**: This component breaks down the query into smaller tasks and distributes them to different worker nodes. It also combines the results from these nodes to produce the final output.
- **Worker nodes**: Queries are decomposed into subqueries that are executed on individual nodes. These nodes perform local computations, and the results are aggregated by a central node or coordinator.
- **Data partitioning**: Data is divided into smaller, manageable chunks (partitions) that are spread across different nodes. Each node processes its partition independently.
- **Data shuffling**: This is the movement of data across nodes, usually during stages like joins or group-by operations. It’s an expensive operation, so query engines try to minimize shuffling by intelligent data placement and partitioning strategies.
- **Fault tolerance**: Distributed query execution needs to handle node failures. Systems like Hadoop (with HDFS) and Spark (with RDDs) ensure that data is replicated across nodes and tasks can be re-executed if a failure occurs.

# Relationship between Apache Arrow, DataFusion, and Ballista
Apache Arrow is a library which provides a standardized memory representation for columnar data. It also provides a set of libraries for performing common operations on this data.

DataFusion is a library for executing queries (query engine) that uses the Apache Arrow memory model and its libraries. It is designed to run within a single process, using threads for parallel query execution.

Ballista is a distributed compute platform built on DataFusion. It allows executing DataFusion queries across multiple nodes. It’s basically a scheduler for DataFusion.

# Conclusion
In this article, we delved into the inner workings of query engines. We explored their fundamental components, examined their interrelationships, and discovered how queries are executed at a low level. Let's briefly recap the key steps:
1. **Parsing**: The first step is to parse the query, which means breaking it down into its individual components (like `SELECT`, `FROM`, `WHERE`) and checking for any syntax errors.
2. **Logical planning**: The planner then creates a logical plan that describes what operations need to be performed (e.g., filtering, joining, sorting), without specifying how to execute them. This plan represents the intent of the query.
3. **Optimization**: The logical plan is then optimized to improve performance. This involves rearranging operations, eliminating unnecessary steps, and choosing the most efficient logical operations.
4. **Physical planning**: After optimization, the logical plan is translated into a physical plan, which specifies how to execute the query, including the choice of algorithms and methods for each step (e.g., which type of join to use).
5. **Execution**: Finally, the query engine executes the physical plan to get the desired results.

I hope this article helps in your future work with query engines and provides a solid foundation for further, more in-depth exploration, should the need arise.
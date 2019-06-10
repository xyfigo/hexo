---
title: Flink适用场景
date: 2019-01-17 10:19:51
tags: Flink
categories: Technology
---

本文翻译自Flink Documents：https://flink.apache.org/usecases.html

# 事件驱动型应用

## 定义

事件驱动型应用是一个有状态的应用，应用从一个或多个事件流中获取事件，并且通过触发计算、更新状态或其他外部行为的方式对获取的事件做出响应。

事件驱动型应用是采用分离计算和数据存储层的设计方式对传统应用的一种改进。在传统应用架构中，应用从远程事务性数据库中读取和持久化数据。

与此相反，事件驱动型应用基于状态流处理应用。在它的设计中，数据和计算位于一处（co-located），以实现本地化数据访问（内存或硬盘）。通过在远程持久化存储中周期性地写入检查点来实现容错性。下图描述了传统应用和事件驱动型应用的不同。

{% asset_img usecases-eventdrivenapps.png Differences %}

## 事件驱动型应用的优势

相比于从远程数据库中查询数据，事件驱动型应用从本地访问数据，得以在吞吐量和延迟时间上取得更好地性能。检查点是以异步和增量的方式在远程持久化存储中周期性的写入。通常在分层架构中，应用的多个层次都共享同一个数据库。因此，数据库的任何改动，例如因为应用更新或服务扩展需要改变数据展现，都需要协调应用的方方面面。因为时间驱动型应用只对自己的数据负责，所以改变数据展现或者扩展应用需要更少的协调工作。

## Flink是如何支持事件驱动型应用的

Flink的许多优秀的特性都围绕这些概念。Flink提供丰富的状态单元集合，它能够管理大数据容量（上至太字节）的一致性保证。此外，Flink提供ProcessFunction以支持事件时间、可定制的窗口逻辑和细粒度的时间控制，以此实现高级的业务逻辑。而且Flink提供了复杂事件处理（CEP）程序库来发现数据流中的模式。

另外，Flink还为事件驱动型应用提供了另一个优秀的特性：保存点（savepoint）。保存点是一个持久化的状态，可以被用来作为兼容应用的一个始点。给应用一个保存点，应用就此开始更新和调整，或者使用多个应用版本开始进行A/B测试。

## 典型的事件驱动型应用

- [欺诈监测](https://sf-2017.flink-forward.org/kb_sessions/streaming-models-how-ing-adds-models-at-runtime-to-catch-fraudsters/)
- [异常检测](https://sf-2017.flink-forward.org/kb_sessions/building-a-real-time-anomaly-detection-system-with-flink-mux/)
- [基于规则的预警](https://sf-2017.flink-forward.org/kb_sessions/dynamically-configured-stream-processing-using-flink-kafka/)
- [业务流程监控](https://jobs.zalando.com/tech/blog/complex-event-generation-for-business-process-monitoring-using-apache-flink/)
- [Web应用程序（社交网络）](https://berlin-2017.flink-forward.org/kb_sessions/drivetribes-kappa-architecture-with-apache-flink/)

# 数据分析应用

分析型任务从原始数据中提取信息并获得知识。典型的分析一般表现为批处理查询或者有限的事件记录数据集。为了在分析结果中合并最新的数据，必须在分析中增加被分析的数据集，并且重新运行查询或分析应用，将结果写入存储系统或者给出分析报告。

如果拥有一个精良的流处理引擎，分析可以实时的进行。流查询应用不再是读取有限的数据集合，而是获取实时事件流并持续地生产和更新分析结果，与此同时事件被消费。结果写入外部数据库或者维护一个内部的状态。数据可视化应用从外部数据库读取最新的结果或者直接查询应用内部的状态。

如下图所示，Apache Flink同时支持流处理和批处理型分析应用。

{% asset_img usecases-analytics.png analytics %}

# 流分析应用的优势

与批处理应用相比，持续流分析应用由于消除周期性的数据导入和执行查询，不仅仅可以更快的从事件中获得知识。与批处理查询相比，流查询不需要处理由于周期性导入和输入数据等导致的人为造成的数据边界。

另一个方面是更简单的应用架构。批处理分析管道包括了几个独立组件周期性调度数据输入和执行查询。管道是一种可靠性的操作，这非常重要，因为一个组件的错误会影响管道后续的过程。与此相反，运行在Flink精密的流处理器之上的流处理分析应用包括了从数据输入到持续结果计算所有的过程。因此，它能够依靠引擎的错误恢复机制。

## Flink如何支持数据分析应用

Flink对流分析和批处理分析都有很好的支持。具体的说，它提供了一个对批处理和流处理标准一致的，并且与ANSI兼容的SQL接口。SQL查询计算对记录的静态数据集或实时事件流都可以得到相同的计算结果。对用户定义函数（UDF）丰富的支持，确保用户代码能够在SQL查询中执行。如果需要更多的客户逻辑，Flink的DataStream API或DataSetAPI提供更多的底层控制。此外，Flink的Gelly库面向批处理数据集提供大规模和高性能图分析的算法和基础功能。

## 典型的数据分析应用

- [电信网络的质量监控](http://2016.flink-forward.org/kb_sessions/a-brief-history-of-time-with-apache-flink-real-time-monitoring-and-analysis-with-flink-kafka-hb/)
- [在移动应用中分析产品更新和实验评估](https://techblog.king.com/rbea-scalable-real-time-analytics-king/)
- [消费分析中的专用实时数据分析](https://eng.uber.com/athenax/)
- 大规模图分析

# 数据管道应用

ETL是一种在不同存储系统之间转换和移动数据的普通方式。需要经常周期性的触发ETL任务，从事务数据库系统到分析数据库或数据仓库拷贝数据。

数据管道应用于ETL任务的目的相似。它能够在一个存储系统到另一个存储系统之间转换和扩充数据。但是，它以连续的流模式运行而不是周期性的调用。因此，它能够从持续产生数据的数据源读取记录，并以较低的延迟将之移动到目的地。例如，数据管道可以监控文件系统目录的新文件，并将新文件数据记录到时间日志中。另一个应用能够将事件记录到数据库中，或者递增地构建和优化查询索引。

下图描述了周期性ETL任务和持续数据管道之间的区别。

{% asset_img usecases-datapipelines.png datapipelines %}

## 数据管道的优势

连续数据管道比周期性ETL任务的明显优势在于移动数据的低延迟。而且，因为数据管道可以连续地消费和发出数据，它具有更多使用场景下的通用性。

## Flink对数据管道的支持

Flink的SQL接口（或表API）和对UDF的支持可以解决大量的普通数据转换和扩充任务。更通用的DataStream API可以实现数据管道更高级的需求。Flink提供丰富的与其他存储系统的连接器，例如Kafka、Kinesis、Elasticsearch、JDBC数据库。它也为文件系统提供连续数据源，用来以时间段的形式监控数据目录。

## 典型的数据管道应用

- [电子商务中实时查询索引的构建](https://data-artisans.com/blog/blink-flink-alibaba-search)
- [电子商务中连续的ETL](https://jobs.zalando.com/tech/blog/apache-showdown-flink-vs.-spark/)
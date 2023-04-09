# Architecture

本页介绍 Apache Pinot 设计背后的指导原则。在这里，您将学习分布式系统架构，该架构允许 Pinot 根据集群中的节点数量线性扩展查询性能。还将介绍用于在 offline (batch) 或 real-time (stream) 模式下摄取 (ingest) 和查询 (query) 数据的两种不同类型的表。

> 建议首先阅读[基本概念](https://github.com/dquaner/apache-pinot/blob/main/Concepts.md)以更好地理解本页中使用的术语。
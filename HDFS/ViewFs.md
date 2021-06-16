# ViewFs

[TOC]

## 1、Introduction

> The View File System (ViewFs) provides a way to manage multiple Hadoop file system namespaces (or namespace volumes). It is particularly useful for clusters having multiple namenodes, and hence multiple namespaces, in [HDFS Federation](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/Federation.html). ViewFs is analogous to client side mount tables in some Unix/Linux systems. ViewFs can be used to create personalized namespace views and also per-cluster common views.

View File System (ViewFs) 提供了一种管理多个 hadoop 文件系统名称空间（或名称空间卷）的方法。

在 HDFS Federation 中，对于具有多个 namenode(即多个命名空间)的集群来说，它特别有用。ViewFs 类似于某些 Unix/Linux 系统中的客户端挂载表。

ViewFs 可以用于创建个性化的名称空间视图，也可以用于创建每个集群的公共视图。

> This guide is presented in the context of Hadoop systems that have several clusters, each cluster may be federated into multiple namespaces. It also describes how to use ViewFs in federated HDFS to provide a per-cluster global namespace so that applications can operate in a way similar to the pre-federation world.

本指南在具有多个集群的Hadoop系统上下文中介绍，每个集群可以联合到多个名称空间中。

它还描述了如何在联邦 HDFS 中使用 ViewFs 来提供每个集群的全局名称空间，以便应用程序可以以类似于前联邦世界的方式进行操作。

## 2、The Old World (Prior to Federation)

### 2.1、Single Namenode Clusters

### 2.2、Pathnames Usage Patterns

### 2.3、Pathname Usage Best Practices

## 3、New World – Federation and ViewFs

### 3.1、How The Clusters Look

### 3.2、A Global Namespace Per Cluster Using ViewFs

### 3.3、Pathname Usage Patterns

### 3.4、Pathname Usage Best Practices

### 3.5、Renaming Pathnames Across Namespaces

## 4、Multi-Filesystem I/0 with Nfly Mount Points

### 4.1、Basic Configuration

### 4.2、Advanced Configuration

### 4.3、Network Topology

### 4.4、Example Nfly Configuration

### 4.5、How Nfly File Creation works

### 4.6、FAQ

## 5、Appendix: A Mount Table Configuration Example
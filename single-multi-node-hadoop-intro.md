# Introduction to Hadoop: Single-Node and Multi-Node Systems

## 1. Introduction to Hadoop

**Apache Hadoop** is an open-source framework used for storing and processing large amounts of data across multiple computers.

Hadoop is mainly based on two important concepts:

* **HDFS (Hadoop Distributed File System)** – Used to store large datasets.
* **YARN (Yet Another Resource Negotiator)** – Used to manage resources and run applications such as MapReduce.

Hadoop can be configured in different modes depending on the purpose:

1. **Single-Node (Pseudo-Distributed) Mode**
2. **Multi-Node (Fully Distributed) Mode**

---

## 2. Single-Node Hadoop System

A **Single-Node Hadoop system** runs all Hadoop services on one computer.

It is mainly used for:

* Learning Hadoop
* Development
* Testing
* Practicing HDFS and MapReduce
* Running small experiments

Although everything runs on one machine, Hadoop services behave similarly to a distributed cluster.

### Architecture

```text
              Single Computer
                    |
        +-----------+-----------+
        |                       |
       HDFS                    YARN
        |                       |
   +----+----+             +----+----+
   |         |             |         |
NameNode  DataNode    ResourceManager NodeManager
   |
SecondaryNameNode
```

### Main Components

| Component         | Purpose                                          |
| ----------------- | ------------------------------------------------ |
| NameNode          | Manages HDFS metadata                            |
| DataNode          | Stores actual data blocks                        |
| SecondaryNameNode | Performs checkpoints of NameNode metadata        |
| ResourceManager   | Manages cluster resources                        |
| NodeManager       | Executes tasks and manages resources on the node |

### Example

If Hadoop is installed on your Ubuntu computer:

```text
Computer
   |
   +-- NameNode
   +-- DataNode
   +-- SecondaryNameNode
   +-- ResourceManager
   +-- NodeManager
```

All these services run on the **same physical or virtual machine**.

---

## 3. Multi-Node Hadoop System

A **Multi-Node Hadoop system** consists of multiple computers connected through a network.

Each computer performs a specific role in the Hadoop cluster.

It is mainly used for:

* Big data processing
* Large-scale storage
* Distributed computing
* Production environments
* Processing very large datasets

### Basic Architecture

```text
                    Hadoop Cluster
                         |
             +-----------+-----------+
             |                       |
          Master Nodes            Worker Nodes
             |                       |
      +------+------+          +-----+-----+
      |             |          |           |
  NameNode    ResourceManager DataNode  NodeManager
                                |
                         +------+------+
                         |             |
                      DataNode      DataNode
                      NodeManager   NodeManager
```

---

## 4. Master and Worker Nodes

### Master Node

The master node manages the Hadoop cluster.

It generally contains:

* **NameNode**
* **ResourceManager**

The NameNode manages HDFS metadata, while the ResourceManager manages computing resources.

### Worker Node

Worker nodes perform actual storage and processing.

They generally contain:

* **DataNode**
* **NodeManager**

A cluster may have many worker nodes.

Example:

```text
Master Node
    |
    +-- NameNode
    +-- ResourceManager
    |
    +---------------------------+
    |            |              |
    v            v              v
Worker 1      Worker 2       Worker 3
    |            |              |
 DataNode     DataNode        DataNode
 NodeManager  NodeManager     NodeManager
```

---

## 5. Single-Node vs Multi-Node Hadoop

| Feature               | Single-Node          | Multi-Node                  |
| --------------------- | -------------------- | --------------------------- |
| Number of computers   | 1                    | 2 or more                   |
| Purpose               | Learning and testing | Large-scale processing      |
| Data storage          | One machine          | Distributed across machines |
| Processing            | One machine          | Multiple machines           |
| Setup                 | Easy                 | More complex                |
| Cost                  | Low                  | Higher                      |
| Scalability           | Limited              | Highly scalable             |
| Network communication | Mostly local         | Between multiple machines   |
| Production use        | Usually no           | Yes                         |

---

## 6. How Data Is Stored

In a multi-node Hadoop cluster, large files are divided into **blocks**.

For example:

```text
Large File
    |
    +---- Block 1
    +---- Block 2
    +---- Block 3
    +---- Block 4
```

These blocks can be distributed across different DataNodes:

```text
              Large File
                  |
       +----------+----------+
       |          |          |
       v          v          v
   DataNode 1  DataNode 2  DataNode 3
       |          |          |
    Block 1    Block 2    Block 3
                  |
               Block 4
```

HDFS can also maintain multiple copies of blocks using **replication**, which improves fault tolerance.

---

## 7. Hadoop Single-Node Setup Example

A typical single-node installation can be configured as:

```text
Ubuntu Linux
     |
     +-- Hadoop 3.4.0
           |
           +-- HDFS
           |    +-- NameNode
           |    +-- DataNode
           |    +-- SecondaryNameNode
           |
           +-- YARN
                +-- ResourceManager
                +-- NodeManager
```

The system can then be used to run MapReduce programs.

For example:

```text
Local File
    |
    v
   HDFS
    |
    v
 MapReduce
    |
    +-- Mapper
    |
    +-- Shuffle & Sort
    |
    +-- Reducer
    |
    v
HDFS Output
```

---

## 8. Advantages of Single-Node Hadoop

* Easy to install
* Easy to configure
* Requires only one computer
* Useful for learning Hadoop
* Good for development and testing
* No need for multiple physical machines

### Limitations

* Limited storage capacity
* Limited processing power
* No real distributed environment
* Limited fault tolerance
* Not suitable for large production workloads

---

## 9. Advantages of Multi-Node Hadoop

* Large storage capacity
* Parallel data processing
* Highly scalable
* Fault tolerant
* Can handle very large datasets
* Suitable for production environments

### Limitations

* More difficult to configure
* Requires multiple machines or virtual machines
* Requires network configuration
* More maintenance is required

---

## 10. Simple Comparison

The main difference can be understood as:

```text
SINGLE-NODE

        One Computer
             |
     +-------+-------+
     |               |
    HDFS            YARN
     |               |
 NameNode         ResourceManager
 DataNode         NodeManager
```

```text
MULTI-NODE

                    Hadoop Cluster
                         |
                  +------+------+
                  |             |
               Master        Workers
                  |             |
             NameNode       DataNodes
             Resource       NodeManagers
             Manager
                  |             |
                  +------+------+ 
                         |
                  Distributed
                   Processing
```

---

## 11. Conclusion

Hadoop provides a distributed platform for **storing and processing large datasets**.

A **Single-Node Hadoop system** runs all Hadoop services on one computer and is ideal for learning, development, and testing.

A **Multi-Node Hadoop system** distributes storage and processing across multiple computers and is designed for large-scale and production workloads.

In simple terms:

```text
Single-Node  →  One computer → Learning & Testing

Multi-Node   →  Multiple computers → Large-scale Processing
```

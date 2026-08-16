# Hadoop 3.4.0 Installation Guide on Ubuntu

This guide explains how to install and configure **Apache Hadoop 3.4.0** on an **Ubuntu Linux system** in **Pseudo-Distributed (Single-Node) Mode**.

---

## Prerequisites & Dependencies

Open your terminal using:

```text
Ctrl + Alt + T
```

Update the system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 1. Install Java (JDK)

Hadoop requires Java to run. Java 8 or Java 11 can be used with Hadoop 3.4.0.

Install OpenJDK 11:

```bash
sudo apt install openjdk-11-jdk -y
```

Verify the Java installation:

```bash
java -version
```

You should see output similar to:

```text
openjdk version "11.x.x"
```

---

## 2. Install & Configure Passwordless SSH

Hadoop uses SSH to communicate between its master and worker services.

### Install SSH

```bash
sudo apt install openssh-server openssh-client -y
```

### Generate an SSH Key

Run:

```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
```

Press **Enter** for all prompts.

### Configure Authorized Keys

Copy the public key into `authorized_keys`:

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

Set the required permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

### Test Passwordless SSH

```bash
ssh localhost
```

If prompted with:

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type:

```text
yes
```

If you successfully log in, type:

```bash
exit
```

to return to your normal shell.

---

# Step 1: Download & Extract Hadoop 3.4.0

## 1. Download Hadoop

Download the Apache Hadoop 3.4.0 release package:

```bash
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
```

## 2. Extract the Archive

```bash
tar -xvzf hadoop-3.4.0.tar.gz
```

## 3. Move Hadoop to `/usr/local/hadoop`

Move the extracted Hadoop directory:

```bash
sudo mv hadoop-3.4.0 /usr/local/hadoop
```

Change ownership so that your current user can manage the Hadoop installation:

```bash
sudo chown -R $USER:$USER /usr/local/hadoop
```

Verify the installation directory:

```bash
ls /usr/local/hadoop
```

You should see directories such as:

```text
bin
etc
include
lib
libexec
sbin
share
```

---

# Step 2: Set Environment Variables

Open your `.bashrc` file:

```bash
nano ~/.bashrc
```

Add the following configuration at the bottom of the file:

```bash
# Java Home Configuration
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

# Hadoop Configuration
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME

# Hadoop Native Libraries
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native

# Hadoop PATH
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
```

Save the file:

```text
Ctrl + O
```

Press:

```text
Enter
```

Exit Nano:

```text
Ctrl + X
```

Reload the environment variables:

```bash
source ~/.bashrc
```

## Verify Environment Variables

Check `JAVA_HOME`:

```bash
echo $JAVA_HOME
```

Expected:

```text
/usr/lib/jvm/java-11-openjdk-amd64
```

Check `HADOOP_HOME`:

```bash
echo $HADOOP_HOME
```

Expected:

```text
/usr/local/hadoop
```

Check the Hadoop executable:

```bash
which hadoop
```

Expected:

```text
/usr/local/hadoop/bin/hadoop
```

Finally, verify Hadoop:

```bash
hadoop version
```

You should see information similar to:

```text
Hadoop 3.4.0
```

---

# Step 3: Configure Hadoop XML Files

Navigate to the Hadoop configuration directory:

```bash
cd /usr/local/hadoop/etc/hadoop
```

The main configuration files are:

```text
hadoop-env.sh
core-site.xml
hdfs-site.xml
mapred-site.xml
yarn-site.xml
```

---

## A. Configure `hadoop-env.sh`

Open the file:

```bash
nano hadoop-env.sh
```

Find:

```bash
# export JAVA_HOME=
```

or add the following line at the bottom:

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

Save and exit.

---

## B. Configure `core-site.xml`

Open:

```bash
nano core-site.xml
```

Replace the existing `<configuration></configuration>` section with:

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

### Purpose

The `fs.defaultFS` property specifies the default HDFS filesystem.

In this configuration:

```text
hdfs://localhost:9000
```

means that the Hadoop client communicates with the local NameNode through port `9000`.

---

## C. Configure `hdfs-site.xml`

Open:

```bash
nano hdfs-site.xml
```

Replace the existing configuration with:

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///usr/local/hadoop/data/namenode</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///usr/local/hadoop/data/datanode</value>
    </property>
</configuration>
```

### Configuration Explanation

| Property                | Purpose                                    |
| ----------------------- | ------------------------------------------ |
| `dfs.replication`       | Number of copies of each HDFS block        |
| `dfs.namenode.name.dir` | Location where NameNode metadata is stored |
| `dfs.datanode.data.dir` | Location where DataNode blocks are stored  |

Since this is a single-node installation, replication is set to:

```text
1
```

---

## D. Configure `mapred-site.xml`

Open:

```bash
nano mapred-site.xml
```

Replace the existing configuration with:

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

### Purpose

This configuration tells Hadoop to use **YARN** as the framework for running MapReduce applications.

---

## E. Configure `yarn-site.xml`

Open:

```bash
nano yarn-site.xml
```

Replace the existing configuration with:

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

### Purpose

This configuration enables the required MapReduce shuffle service and allows important Hadoop environment variables to be passed to YARN containers.

---

# Step 4: Format NameNode and Start Hadoop Daemons

## 1. Format the HDFS NameNode

Before starting Hadoop for the first time, format the NameNode:

```bash
hdfs namenode -format
```

Look for a message similar to:

```text
Storage directory /usr/local/hadoop/data/namenode has been successfully formatted
```

> **Important:** Run `hdfs namenode -format` only when initializing a new HDFS filesystem. Re-formatting an existing NameNode can erase its HDFS metadata.

---

## 2. Start HDFS Services

Start the HDFS daemons:

```bash
start-dfs.sh
```

This starts:

* NameNode
* DataNode
* SecondaryNameNode

---

## 3. Start YARN Services

Start YARN:

```bash
start-yarn.sh
```

This starts:

* ResourceManager
* NodeManager

---

## 4. Verify Running Daemons

Run:

```bash
jps
```

A correctly running single-node Hadoop installation should show:

```text
NameNode
DataNode
SecondaryNameNode
ResourceManager
NodeManager
Jps
```

Example:

```text
12345 NameNode
12346 DataNode
12347 SecondaryNameNode
12348 ResourceManager
12349 NodeManager
12350 Jps
```

---

# Step 5: Verify Hadoop Using Web Interfaces

Hadoop provides web-based interfaces for monitoring HDFS and YARN.

## Hadoop NameNode UI

Open the following address in your browser:

```text
http://localhost:9870
```

The NameNode dashboard provides information about:

* HDFS capacity
* Live DataNodes
* Filesystem information
* HDFS storage
* Cluster status

---

## YARN ResourceManager UI

Open:

```text
http://localhost:8088
```

The ResourceManager dashboard provides information about:

* Running applications
* Cluster resources
* NodeManager status
* Memory usage
* CPU usage
* YARN applications

---

# Useful Hadoop Commands

## Check Hadoop Version

```bash
hadoop version
```

## Check HDFS Status

```bash
hdfs dfsadmin -report
```

## List HDFS Root Directory

```bash
hdfs dfs -ls /
```

## Create an HDFS Directory

```bash
hdfs dfs -mkdir /test
```

## Upload a Local File to HDFS

```bash
hdfs dfs -put filename.txt /test/
```

## List Files in HDFS

```bash
hdfs dfs -ls /test
```

## Download a File from HDFS

```bash
hdfs dfs -get /test/filename.txt .
```

## Stop HDFS

```bash
stop-dfs.sh
```

## Stop YARN

```bash
stop-yarn.sh
```

---

# Final Verification Checklist

After completing the installation, verify the following:

* [ ] Java is installed
* [ ] `JAVA_HOME` is configured
* [ ] SSH server is installed
* [ ] Passwordless SSH works
* [ ] Hadoop 3.4.0 is extracted
* [ ] `HADOOP_HOME` is configured
* [ ] Hadoop is available in `PATH`
* [ ] `hadoop version` works
* [ ] `core-site.xml` is configured
* [ ] `hdfs-site.xml` is configured
* [ ] `mapred-site.xml` is configured
* [ ] `yarn-site.xml` is configured
* [ ] NameNode is formatted
* [ ] HDFS daemons are running
* [ ] YARN daemons are running
* [ ] `jps` shows all required processes
* [ ] NameNode UI opens at `http://localhost:9870`
* [ ] ResourceManager UI opens at `http://localhost:8088`

---

# Hadoop Single-Node Architecture

```text
                    Ubuntu Linux
                         |
                         |
                +--------+--------+
                |                 |
             HDFS               YARN
                |                 |
        +-------+-------+    +----+---------+
        |       |       |    |              |
   NameNode  DataNode  Secondary     ResourceManager
                         NameNode            |
                                             |
                                        NodeManager
                                             |
                                      MapReduce Jobs
```

This completes the installation and configuration of **Hadoop 3.4.0 in Pseudo-Distributed (Single-Node) Mode on Ubuntu Linux**.

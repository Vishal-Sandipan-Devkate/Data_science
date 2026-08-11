# Running a MapReduce Job on Hadoop 3.4.0

This guide explains how to run a **Java MapReduce WordCount program** on a Hadoop 3.4.0 installation running in **Pseudo-Distributed (Single-Node) Mode**.

The process covers:

1. Verifying Hadoop cluster health
2. Creating local input data
3. Writing a Java MapReduce program
4. Compiling the Java program
5. Creating a JAR file
6. Uploading input data to HDFS
7. Running the MapReduce job using YARN
8. Reading and validating the output
9. Understanding the MapReduce execution pipeline
10. Troubleshooting common errors

---

# Step 1: Verify the Cluster Health

Before running a MapReduce job, make sure that **HDFS** and **YARN** are running.

## Start Hadoop Services

If the services are not already running:

```bash
start-dfs.sh
start-yarn.sh
```

## Check Running Processes

Run:

```bash
jps
```

You should see the following Hadoop daemons:

```text
NameNode
DataNode
SecondaryNameNode
ResourceManager
NodeManager
Jps
```

The five important Hadoop services are:

| Daemon            | Purpose                                 |
| ----------------- | --------------------------------------- |
| NameNode          | Manages HDFS metadata                   |
| DataNode          | Stores HDFS data blocks                 |
| SecondaryNameNode | Performs NameNode checkpoint operations |
| ResourceManager   | Manages YARN resources                  |
| NodeManager       | Executes tasks on worker nodes          |

---

# Step 2: Create Local Input Data

Create a dedicated project directory:

```bash
mkdir -p ~/mapreduce_project
cd ~/mapreduce_project
```

Create a sample text file:

```bash
cat << 'EOF' > input_data.txt
hadoop is a distributed storage and processing framework
mapreduce is the processing engine for hadoop
hadoop runs on top of linux
linux is fast and reliable for distributed systems
EOF
```

Verify the file:

```bash
cat input_data.txt
```

Expected input:

```text
hadoop is a distributed storage and processing framework
mapreduce is the processing engine for hadoop
hadoop runs on top of linux
linux is fast and reliable for distributed systems
```

---

# Step 3: Write the MapReduce Program in Java

Create the Java source file:

```bash
nano WordCount.java
```

Paste the following code:

```java
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

    // Mapper:
    // Reads each line and produces (word, 1)
    public static class TokenizerMapper
            extends Mapper<Object, Text, Text, IntWritable> {

        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public void map(
                Object key,
                Text value,
                Context context
        ) throws IOException, InterruptedException {

            StringTokenizer itr =
                    new StringTokenizer(value.toString().toLowerCase());

            while (itr.hasMoreTokens()) {

                // Remove basic punctuation
                String token =
                        itr.nextToken().replaceAll("[^a-zA-Z0-9]", "");

                if (!token.isEmpty()) {
                    word.set(token);
                    context.write(word, one);
                }
            }
        }
    }

    // Reducer:
    // Receives (word, [1,1,1,...]) and calculates the total count
    public static class IntSumReducer
            extends Reducer<Text, IntWritable, Text, IntWritable> {

        private IntWritable result = new IntWritable();

        public void reduce(
                Text key,
                Iterable<IntWritable> values,
                Context context
        ) throws IOException, InterruptedException {

            int sum = 0;

            for (IntWritable val : values) {
                sum += val.get();
            }

            result.set(sum);
            context.write(key, result);
        }
    }

    // Driver:
    // Configures and submits the MapReduce job
    public static void main(String[] args) throws Exception {

        if (args.length != 2) {
            System.err.println(
                    "Usage: WordCount <HDFS input path> <HDFS output path>"
            );
            System.exit(-1);
        }

        Configuration conf = new Configuration();

        Job job = Job.getInstance(conf, "Word Count Job");

        job.setJarByClass(WordCount.class);

        // Set Mapper
        job.setMapperClass(TokenizerMapper.class);

        // Combiner performs local aggregation
        job.setCombinerClass(IntSumReducer.class);

        // Set Reducer
        job.setReducerClass(IntSumReducer.class);

        // Output key and value types
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // HDFS input and output paths
        FileInputFormat.addInputPath(
                job,
                new Path(args[0])
        );

        FileOutputFormat.setOutputPath(
                job,
                new Path(args[1])
        );

        System.exit(
                job.waitForCompletion(true) ? 0 : 1
        );
    }
}
```

Save the file:

```text
Ctrl + O
```

Press **Enter**, then exit:

```text
Ctrl + X
```

Verify the source file:

```bash
ls
```

You should see:

```text
WordCount.java
input_data.txt
```

---

# Step 4: Compile the Java Program

Hadoop provides the `hadoop classpath` command to obtain the required Hadoop libraries.

Compile `WordCount.java`:

```bash
javac -classpath $(hadoop classpath) -d . WordCount.java
```

Check the generated class files:

```bash
find . -name "*.class"
```

You should see files similar to:

```text
./WordCount.class
./WordCount$TokenizerMapper.class
./WordCount$IntSumReducer.class
```

---

# Step 5: Create the JAR File

Package the compiled Java classes into a JAR file:

```bash
jar -cvf wordcount.jar *.class
```

Verify:

```bash
ls -lh wordcount.jar
```

You should now have:

```text
WordCount.java
WordCount.class
WordCount$TokenizerMapper.class
WordCount$IntSumReducer.class
input_data.txt
wordcount.jar
```

> **Note:** The JAR must contain the `WordCount.class` and its inner classes.

---

# Step 6: Prepare Input Data in HDFS

MapReduce reads input data from **HDFS**, so the local input file must be uploaded to HDFS.

## Create the HDFS Input Directory

```bash
hdfs dfs -mkdir -p /user/hadoop/wc_input
```

## Upload the Input File

```bash
hdfs dfs -put -f input_data.txt /user/hadoop/wc_input/
```

## Verify the File

```bash
hdfs dfs -ls /user/hadoop/wc_input
```

Expected output will contain:

```text
/user/hadoop/wc_input/input_data.txt
```

You can also verify its contents directly from HDFS:

```bash
hdfs dfs -cat /user/hadoop/wc_input/input_data.txt
```

---

# Step 7: Submit the MapReduce Job

The output directory **must not already exist**.

Run the MapReduce job:

```bash
hadoop jar wordcount.jar WordCount \
    /user/hadoop/wc_input \
    /user/hadoop/wc_output
```

The command follows this structure:

```text
hadoop jar <JAR> <Main-Class> <HDFS-input> <HDFS-output>
```

In this example:

```text
JAR          = wordcount.jar
Main Class   = WordCount
Input        = /user/hadoop/wc_input
Output       = /user/hadoop/wc_output
```

During execution, Hadoop will display progress similar to:

```text
map 0% reduce 0%
map 50% reduce 0%
map 100% reduce 0%
map 100% reduce 50%
map 100% reduce 100%
```

At the end, you should see a successful completion message.

---

# Step 8: Read and Validate the Output

After the job completes, list the output directory:

```bash
hdfs dfs -ls /user/hadoop/wc_output
```

You should see files similar to:

```text
/user/hadoop/wc_output/_SUCCESS
/user/hadoop/wc_output/part-r-00000
```

## Output Files

| File           | Description                                 |
| -------------- | ------------------------------------------- |
| `_SUCCESS`     | Marker indicating successful job completion |
| `part-r-00000` | Actual output generated by the reducer      |

Read the reducer output:

```bash
hdfs dfs -cat /user/hadoop/wc_output/part-r-00000
```

Expected output:

```text
a              1
and            2
distributed    2
engine         1
fast           1
for            2
framework      1
hadoop         3
is             3
linux          2
mapreduce      1
of             1
on             1
processing     2
reliable       1
runs           1
storage        1
systems        1
the            1
top            1
```

The exact ordering is normally lexicographical because Hadoop sorts keys before passing them to the reducer.

---

# How the WordCount Program Works

The MapReduce program consists of three main components:

```text
              WordCount Program
                     |
        +------------+------------+
        |            |            |
      Mapper      Combiner     Reducer
        |            |            |
     (word,1)    local sum     final sum
```

---

## 1. Mapper

The Mapper receives each input line.

For example:

```text
hadoop is a distributed storage
```

The Mapper generates:

```text
(hadoop, 1)
(is, 1)
(a, 1)
(distributed, 1)
(storage, 1)
```

The important line is:

```java
context.write(word, one);
```

It emits:

```text
(word, 1)
```

---

# 2. Combiner

The Combiner is an optional optimization.

The program uses:

```java
job.setCombinerClass(IntSumReducer.class);
```

Suppose a Mapper generates:

```text
(hadoop, 1)
(hadoop, 1)
(hadoop, 1)
```

The Combiner can locally aggregate these values into:

```text
(hadoop, 3)
```

This reduces the amount of data transferred during the shuffle phase.

---

# 3. Shuffle and Sort

After the Mapper finishes, Hadoop performs **shuffle and sort**.

Values belonging to the same key are grouped together.

For example:

```text
(hadoop, 1)
(hadoop, 1)
(hadoop, 1)
```

becomes:

```text
(hadoop, [1, 1, 1])
```

The reducer then receives:

```text
key    = hadoop
values = [1, 1, 1]
```

---

# 4. Reducer

The Reducer calculates the final count.

The important logic is:

```java
int sum = 0;

for (IntWritable val : values) {
    sum += val.get();
}
```

For:

```text
hadoop → [1, 1, 1]
```

the reducer calculates:

```text
1 + 1 + 1 = 3
```

and produces:

```text
hadoop    3
```

---

# Behind the Scenes: MapReduce Execution Pipeline

A MapReduce job can be understood as four major stages:

| Stage                            | What Happens                                              | Example             |
| -------------------------------- | --------------------------------------------------------- | ------------------- |
| **1. Input Splitting & Mapping** | Input is divided into splits and Mapper processes records | `(hadoop, 1)`       |
| **2. Combiner**                  | Local aggregation reduces intermediate data               | `(hadoop, 3)`       |
| **3. Shuffle & Sort**            | Same keys are grouped and sorted                          | `(hadoop, [1,1,1])` |
| **4. Reducer**                   | Reducer calculates the final result                       | `hadoop 3`          |

The overall flow is:

```text
                    HDFS Input
                        |
                        v
                +---------------+
                | Input Split   |
                +---------------+
                        |
                        v
                +---------------+
                |    Mapper     |
                +---------------+
                        |
                 (word, 1)
                        |
                        v
                +---------------+
                |    Combiner   |
                +---------------+
                        |
                  Local totals
                        |
                        v
                +---------------+
                | Shuffle & Sort|
                +---------------+
                        |
                (word, [1,1,...])
                        |
                        v
                +---------------+
                |    Reducer    |
                +---------------+
                        |
                        v
                HDFS Output
```

---

# MapReduce Job Execution Through YARN

Because the configuration uses:

```xml
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
```

the MapReduce job runs through **YARN**.

The simplified execution flow is:

```text
                 Client
                   |
                   | Submit Job
                   v
          +-------------------+
          | ResourceManager   |
          +-------------------+
                   |
                   v
          +-------------------+
          | Application       |
          | Master            |
          +-------------------+
                   |
                   v
          +-------------------+
          |   NodeManager     |
          +-------------------+
                   |
            +------+------+
            |             |
            v             v
         Mapper        Reducer
            |             |
            +------+------+
                   |
                   v
                 HDFS
```

---

# Monitor the Job Using YARN

While the job is running, open:

```text
http://localhost:8088
```

The YARN ResourceManager UI allows you to monitor:

* Running applications
* Application status
* Application attempts
* Memory allocation
* CPU resources
* Mapper progress
* Reducer progress
* Task attempts
* Application logs

---

# Common Problems and Solutions

## 1. Output Directory Already Exists

### Error

You may see:

```text
Output directory already exists
```

Hadoop does not overwrite an existing output directory by default.

### Solution

Delete the previous output directory:

```bash
hdfs dfs -rm -r -skipTrash /user/hadoop/wc_output
```

Then run the job again:

```bash
hadoop jar wordcount.jar WordCount \
    /user/hadoop/wc_input \
    /user/hadoop/wc_output
```

---

## 2. Hadoop Services Are Not Running

Check:

```bash
jps
```

Make sure these processes are present:

```text
NameNode
DataNode
SecondaryNameNode
ResourceManager
NodeManager
```

If necessary:

```bash
start-dfs.sh
start-yarn.sh
```

---

## 3. Input File Not Found

Check the HDFS input directory:

```bash
hdfs dfs -ls /user/hadoop/wc_input
```

If the file is missing:

```bash
hdfs dfs -put input_data.txt /user/hadoop/wc_input/
```

---

## 4. Check HDFS Health

Run:

```bash
hdfs dfsadmin -report
```

This displays information about the DataNode and HDFS storage.

---

## 5. Check Hadoop Version

Verify that Hadoop is correctly configured:

```bash
hadoop version
```

Expected:

```text
Hadoop 3.4.0
```

---

# Useful Commands

### List HDFS Files

```bash
hdfs dfs -ls /
```

### List Input Files

```bash
hdfs dfs -ls /user/hadoop/wc_input
```

### Read Input File

```bash
hdfs dfs -cat /user/hadoop/wc_input/input_data.txt
```

### Read MapReduce Output

```bash
hdfs dfs -cat /user/hadoop/wc_output/part-r-00000
```

### Remove Output

```bash
hdfs dfs -rm -r -skipTrash /user/hadoop/wc_output
```

### Check HDFS Report

```bash
hdfs dfsadmin -report
```

### Check Hadoop Processes

```bash
jps
```

### Stop Hadoop Services

```bash
stop-yarn.sh
stop-dfs.sh
```

---

# Complete Command Summary

Once Hadoop is installed and configured, the complete workflow can be summarized as:

```bash
# 1. Start Hadoop
start-dfs.sh
start-yarn.sh

# 2. Verify services
jps

# 3. Create project
mkdir -p ~/mapreduce_project
cd ~/mapreduce_project

# 4. Create input data
cat << 'EOF' > input_data.txt
hadoop is a distributed storage and processing framework
mapreduce is the processing engine for hadoop
hadoop runs on top of linux
linux is fast and reliable for distributed systems
EOF

# 5. Create WordCount.java
nano WordCount.java

# 6. Compile Java program
javac -classpath $(hadoop classpath) -d . WordCount.java

# 7. Create JAR
jar -cvf wordcount.jar *.class

# 8. Create HDFS input directory
hdfs dfs -mkdir -p /user/hadoop/wc_input

# 9. Upload input
hdfs dfs -put -f input_data.txt /user/hadoop/wc_input/

# 10. Run MapReduce job
hadoop jar wordcount.jar WordCount \
    /user/hadoop/wc_input \
    /user/hadoop/wc_output

# 11. Check output
hdfs dfs -ls /user/hadoop/wc_output

# 12. Display result
hdfs dfs -cat /user/hadoop/wc_output/part-r-00000
```

---

# Final Checklist

Before running the job:

* [ ] Hadoop 3.4.0 is installed
* [ ] Java is configured
* [ ] HDFS is running
* [ ] YARN is running
* [ ] `jps` shows the required daemons
* [ ] Input file exists locally
* [ ] `WordCount.java` is created
* [ ] Java source compiles successfully
* [ ] `wordcount.jar` is created
* [ ] Input file is uploaded to HDFS
* [ ] HDFS output directory does not already exist
* [ ] MapReduce job completes successfully
* [ ] `part-r-00000` contains the word counts
* [ ] YARN UI is accessible at `http://localhost:8088`

---

# Conclusion

This exercise demonstrates the complete lifecycle of a Hadoop MapReduce job:

```text
Local File
    |
    v
HDFS Input
    |
    v
Mapper
    |
    v
Combiner
    |
    v
Shuffle & Sort
    |
    v
Reducer
    |
    v
HDFS Output
```

The **Mapper** converts input text into `(word, 1)` pairs, the **Combiner** performs optional local aggregation, the **Shuffle and Sort** phase groups identical words, and the **Reducer** calculates the final word frequencies.

The job is managed by **YARN** and the final results are stored in **HDFS**.

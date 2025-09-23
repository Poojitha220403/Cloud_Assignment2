# Assignment 2: Document Similarity using MapReduce

**Name:** Poojitha Jayareddygari
**Student ID:** 801426875

---

## Approach and Implementation

### Mapper Design

* **MapperA**:

  * Input: A line of text in the format

    ```
    DocumentID word1 word2 word3 ...
    ```
  * Output:

    * `(word, docId)` to build the **inverted index**.
    * `(docId, 1)` for counting document sizes.

* **MapperB**:

  * Input: Inverted index output (word → list of docs).
  * Output: All possible `(doc1, doc2)` pairs sharing the word with value `1`.

This ensures we can count overlaps between documents for Jaccard similarity.

---

### Reducer Design

* **ReducerA**:

  * Consumes MapperA output.
  * Builds the inverted index (`word → docs`).
  * Counts unique words per document and writes them into `docSizes`.

* **ReducerB**:

  * Consumes MapperB output (pairs).
  * Aggregates counts of shared words (`overlap`).
  * Loads document sizes from `docSizes`.
  * Computes **Jaccard Similarity**:

    $$
    J(A,B) = \frac{|A \cap B|}{|A \cup B|}
    $$
  * Emits:

    ```
    DocumentX, DocumentY Similarity: <score>
    ```

---

### Overall Data Flow

1. Input dataset loaded into **HDFS** under `/input`.
2. **Job A**: MapperA + ReducerA build `docSizes` and `inverted index`.
3. **Job B**: MapperB + ReducerB use these to generate all document pairs and compute similarity.
4. Output is written to `/output/final/part-r-00000` in HDFS.

---

## Setup and Execution

### 1. Start the Hadoop Cluster

For single node:

```bash
docker compose up -d
```

### 2. Build the Code

```bash
mvn clean package -DskipTests
```

### 3. Copy JAR to Namenode

```bash
docker cp target/DocumentSimilarity-0.0.1-SNAPSHOT.jar namenode:/DocumentSimilarity-0.0.1-SNAPSHOT.jar
```

### 4. Load Dataset to HDFS

```bash
docker exec -it namenode hdfs dfs -rm -r /input /output
docker exec -it namenode hdfs dfs -mkdir /input
docker cp dataset/ds1.txt namenode:/ds1.txt
docker exec -it namenode hdfs dfs -put /ds1.txt /input
```

### 5. Run MapReduce Job

```bash
docker exec -it namenode hadoop jar /DocumentSimilarity-0.0.1-SNAPSHOT.jar \
  com.example.controller.DocumentSimilarityDriver /input /output
```

### 6. View Output

```bash
docker exec -it namenode hdfs dfs -cat /output/final/part-r-00000
```

### 7. Save Output Locally

```bash
docker exec -it namenode hdfs dfs -get /output/final/part-r-00000 /tmp/single_node_output.txt
docker cp namenode:/tmp/single_node_output.txt ./output/single_node_output.txt
```

---

## Repository Structure

```
Cloud_Assignment2/
├── dataset/
│   └── ds1.txt
├── output/
│   ├── results_ds1.txt         # 3-node output
│   ├── single_node_output.txt  # 1-node output
├── src/main/java/com/example/
│   ├── DocumentSimilarityMapper.java
│   ├── DocumentSimilarityReducer.java
│   └── controller/DocumentSimilarityDriver.java
├── docker-compose.yml
├── hadoop.env
└── README.md
```

---

## Results

### Sample Input

```
Document1 This is a sample document containing words
Document2 Another document that also has words
Document3 Sample text with different words
```

### Sample Output

```
Document1, Document2 Similarity: 0.56
Document1, Document3 Similarity: 0.42
Document2, Document3 Similarity: 0.50
```

### Obtained Output

**From 3 datanodes** (`output/results_ds1.txt`):

```
Document1, Document2 Similarity: 0.01
Document1, Document3 Similarity: 0.01
Document2, Document3 Similarity: 0.94
```

**From single node** (`output/single_node_output.txt`):

```
Document1, Document2 Similarity: 0.01
Document1, Document3 Similarity: 0.01
Document2, Document3 Similarity: 0.94
```

---

## Performance Comparison

| Configuration   | Execution Time | Notes                            |
| --------------- | -------------- | -------------------------------- |
| **3 Datanodes** | 13 sec         | Parallel execution across nodes  |
| **1 Datanode**  | 24 sec         | All tasks handled by single node |


---

## Challenges and Solutions

* **Problem:** Reducer initially crashed parsing non-integer values.
  **Solution:** Fixed ReducerB to properly compute overlaps and load `docSizes`.

* **Problem:** HDFS output conflict (`output already exists`).
  **Solution:** Cleaned `/input` and `/output` before re-running jobs.

* **Problem:** Results not visible locally.
  **Solution:** Used `hdfs dfs -get` and `docker cp` to copy files from namenode to local repo.

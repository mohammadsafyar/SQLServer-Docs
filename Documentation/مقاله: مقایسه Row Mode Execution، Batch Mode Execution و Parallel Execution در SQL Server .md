# Comprehensive Comparison of Row Mode Execution, Batch Mode Execution, and Parallel Execution in SQL Server

In this article, we aim to deeply explore the three primary query processing methods in SQL Server:  
- **Row Mode Execution**: The traditional row-by-row approach  
- **Batch Mode Execution**: The modern batch-oriented approach  
- **Parallel Execution**: Parallel processing using multiple cores  

These methods each have their strengths and weaknesses, and their performance varies depending on the type of query, data volume, and hardware. Our goal is to explain how they work, their advantages, disadvantages, and use cases, helping you choose the best method for your needs.

---

## 1. What is Row Mode Execution?

### Definition
Row Mode Execution is the traditional and foundational method of processing queries in SQL Server, dating back to its early versions. In this approach, each row of data is processed individually and independently. It’s like a meticulous worker handling items one by one, checking each before moving to the next. By default, SQL Server uses this method for most queries unless the Query Optimizer determines that another method, like Batch Mode or Parallelism, would perform better.

### How It Works
- **Row-by-Row Processing**: Each CPU core processes only one row at a time. For example, if applying a filter like `WHERE Price > 100`, SQL Server reads each row, checks the condition, and then moves to the next row.  
- **Parallelism**: If the system has multiple cores, SQL Server can distribute the data across cores, allowing each core to process its portion in parallel. For instance, with a table of 50 million rows:  
  - Core 1: Rows 1 to 25 million  
  - Core 2: Rows 25,000,001 to 50 million  
  This distribution reduces overall processing time, but each core still processes rows sequentially.  
- **Data Distribution**: SQL Server uses a cost-based optimization algorithm (Cost-Based Optimization) to distribute data across cores, aiming for all cores to finish simultaneously. However, the distribution isn’t always perfectly even due to factors like:  
  - **Data Skew**: For example, if some data requires more processing due to complex filters or heavy JOINs.  
  - **Available Resources**: If a core is busy with other tasks, it might receive less data.

### Real-World Example
Imagine a sales table with 10 million rows, and you want to calculate the total sales for 2023 (`SELECT SUM(Sales) FROM Orders WHERE Year = 2023`). In Row Mode:  
- SQL Server reads all 10 million rows individually.  
- It checks each row to see if `Year = 2023`.  
- If you have multiple cores (e.g., 4 cores), the data is split into four 2.5-million-row chunks, and each core processes its portion independently.

### Advantages
- **Simplicity**: It’s fast and straightforward for small queries, like retrieving details for a specific customer.  
- **Suitable for Small Tables**: If your table has only a few hundred rows, this method doesn’t need more complex approaches.

### Disadvantages
Due to the individual processing of each row and the presence of Context Switching between processing stages, some overhead (Overhead) is created. For example, when the CPU moves from one row to the next, it must reload memory and instructions, which is time-consuming.  
It’s slower for complex queries or large tables: In scenarios like generating reports from multi-million-row tables or heavy JOINs, this method becomes significantly slower because it processes each row individually.

### Control and Monitoring
- **MAXDOP**: You can limit the number of involved cores by setting `MAXDOP` (Maximum Degree of Parallelism). For example, setting `MAXDOP = 4` limits usage to a maximum of 4 cores.  
- **DMVs**: Use Dynamic Management Views like `sys.dm_exec_requests` or `sys.dm_os_waiting_tasks` to see how work is distributed across cores and whether delays exist.

---

## 2. What is Batch Mode Execution?

### Definition
Batch Mode Execution is an advanced method designed by SQL Server to improve efficiency for processing large datasets. Instead of processing individual rows, it groups data into smaller batches (typically 64 to 900 rows) and performs operations on entire batches simultaneously. This method is most commonly used with columnstore indexes (Columnstore Indexes), but since SQL Server 2019, it’s also available for some queries on row-based tables (Rowstore) under "Batch Mode on Rowstore."

### How It Works
- **Data Batching**: Data is divided into small groups. The batch size depends on SQL Server’s optimization and typically ranges from 64 to 900 rows.  
- **Vectorized Processing**: Instead of processing rows individually, operations like sums, filters, or JOINs are performed on entire batches using CPU vector instructions (like SIMD - Single Instruction, Multiple Data). This allows the CPU to process hundreds of rows at once.  
- **Connection with Columnstore**: Columnstore indexes store data column-wise and compressed, making them ideal for Batch Mode. This method only reads the necessary columns, avoiding the need to process entire rows.

### Simple and Expanded Example
Imagine a sales table with 1 million rows, and you want to calculate the total of the "Price" column for orders in 2023 (`SELECT SUM(Price) FROM Sales WHERE OrderDate >= '2023-01-01' AND OrderDate < '2024-01-01'`):  
- **Row Mode**: SQL Server reads all 1 million rows, checks the date condition, and calculates the sum. If each row takes 1 microsecond, the total time is about 1 second.  
- **Batch Mode**: Data is divided into batches of 1,000 (1,000 batches). Each batch is processed simultaneously, for example, in 500 nanoseconds. The total time can drop to 0.5 milliseconds (depending on hardware). Since only the `Price` and `OrderDate` columns are read, memory usage is also lower.

### Advantages
- **Higher Speed**: In real-world scenarios, performance can improve by 2 to 4 times (or even 10 times in large databases).  
- **Optimized Resource Usage**: Batch processing uses memory and CPU more efficiently, requiring less Memory Grant.  
- **Large Data Analysis**: Ideal for complex operations like aggregations (sums, averages), heavy filters, or JOINs on multi-million-row tables.  
- **Compression**: Columnar data can be compressed up to 10 times, reducing I/O operations.  
- **CPU Efficiency**: Tests show CPU processing time can decrease by up to 50% because SIMD instructions are used.

### Why It Pairs Well with Columnstore
- **Columnar Structure**: In Row Mode, entire rows are read, but in Columnstore, only necessary columns (e.g., `Price` and `OrderDate`) are loaded.  
- **Advanced Compression**: Data fits into RAM, speeding up processing.  
- **SIMD**: Modern CPUs (like Intel AVX) can process multiple data points simultaneously, aligning perfectly with Batch Mode.

### Limitations
- **Dependency on Columnstore**: In versions before 2019, it only worked with columnstore indexes.  
- **Simple Queries**: It’s not optimal for lightweight operations (like `SELECT * FROM Customers WHERE ID = 1`), where Row Mode is faster.  
- **Hardware Requirements**: For optimal performance, it requires a powerful CPU (with SIMD support) and sufficient RAM.

### Detecting Usage
In the **Execution Plan** in SSMS, if you see terms like "Batch Mode Scan," "Batch Mode Hash Join," or "Columnstore Index Scan," this method is active.

### Advantages of Using Batch Execution Mode
- **Optimized Memory Usage**: Batch processing significantly reduces memory consumption and the Memory Grant required for queries, enabling large queries to run with fewer resources.  
- **Faster and More Efficient Execution of Large Queries**: Batch processing allows SQL Server to handle massive datasets more quickly, especially in multi-million or billion-row tables.  
- **Improved Query Performance**: Processing multiple rows simultaneously reduces wait times for complex queries, enhancing user experience.  
- **Better CPU Utilization**: Using vector instructions (like SIMD) can reduce CPU processing time for each query by up to 50% in some scenarios, maximizing efficiency.

---

## 3. What is Parallel Execution?

### Definition
Parallel Execution occurs when SQL Server splits a query into multiple pieces and executes each piece with a separate thread on different CPU cores. This method acts like a team where each member focuses on a part of the task to complete the project faster.

### How It Works
- **Data Partitioning**: The table is divided into smaller sections, and each section is assigned to a core.  
- **Parallel Execution**: Each core works independently, and the results are combined at the end.  
- **Example**: Suppose you want to filter a 20-million-row table (`SELECT * FROM Orders WHERE Status = 'Shipped'`). SQL Server splits the table into four 5-million-row sections, and each core processes one section.

### Differences from Row Mode and Batch Mode
- **Row Mode + Parallelism**: Each core processes rows sequentially but on its own section. For example, it examines 5 million rows one by one.  
- **Batch Mode + Parallelism**: Each core processes batches of data (e.g., 1,000 rows at a time), which is more efficient because it handles multiple rows simultaneously.

### Advantages
- **Reduced Execution Time**: For heavy queries (like annual reports or large JOINs), the overall time is significantly reduced.  
- **Efficient CPU Usage**: If you have a multi-core system (e.g., 16 cores), Parallel Execution can fully leverage the hardware’s power.

### Disadvantages
- **Resource Complexity**: Managing threads and coordinating between cores introduces overhead, sometimes causing delays (Thread Synchronization Overhead).  
- **Data Skew**: If data is unevenly distributed (e.g., 80% of rows match a filter while 20% don’t), one core might finish quickly while others remain busy.

---

## Comparison of Row Mode and Batch Mode

| Feature               | Row Mode Execution                  | Batch Mode Execution                  |
|-----------------------|-------------------------------------|---------------------------------------|
| Processing Unit       | Individual rows                     | Small batches (64–900 rows)            |
| Speed                 | Slower for large datasets          | 2–4 times faster (up to 10 times)     |
| Resource Usage        | More overhead, higher RAM/CPU      | More efficient, lower RAM, better CPU |
| Index Relation        | Independent of index type          | Optimized for Columnstore             |
| Use Case              | Small, simple queries              | Large, complex data analysis          |

---

## Key Questions Answered

### 1. Does each core process 25 million records one by one in Row Mode?
Yes, in Row Mode, each core processes one record at a time and proceeds sequentially:  
- Core 1: Records 1 to 25 million  
- Core 2: Records 25,000,001 to 50 million  
For example, if there’s a filter like `WHERE Amount > 1000`, each core checks each row individually.

### 2. Is the distribution of records among cores equal?
Usually, yes, but not always:  
- **Influencing Factors**:  
  - **Cost-Based Optimization**: SQL Server tries to distribute the workload so all cores finish simultaneously.  
  - **Data Skew**: If a data section is heavier (e.g., due to complex JOINs or filters), distribution becomes uneven.  
  - **Resources**: If a core is busy, it might receive less data.  
- In practice, SQL Server manages this balance using tools like the **Exchange Operator** in the Execution Plan.

---

## Conclusion
- **Row Mode Execution**: A simple and reliable method for lightweight queries and small tables, but it performs poorly and has more overhead with large datasets.  
- **Batch Mode Execution**: The best choice for analyzing large datasets, especially with columnstore indexes. Its high speed, low resource usage, and efficient CPU utilization (including optimized memory usage, faster query execution, and up to 50% CPU time reduction) make it ideal for OLAP systems and data warehouses.  
- **Parallel Execution**: Reduces execution time by using multiple cores and can combine with both methods above for maximum performance.  

### Optimization Tips
- **Control with MAXDOP and Resource Governor**: Adjust `MAXDOP` to limit cores (e.g., 8 for a 16-core server) and use Resource Governor to allocate resources among users.  
- **Columnstore and Batch Mode**: For data warehouses (Data Warehouse) and OLAP systems, create columnstore indexes and leverage Batch Mode. For example, in multi-hundred-million-row tables, it can multiply reporting speed.  
- **Review Execution Plan**: Always check the execution plan in SSMS to see which operators (like Hash Join or Index Scan) are used and whether it’s optimized.  

These three methods are like tools in a toolbox: each has a specific purpose, and choosing the right one depends on your scenario (data volume, query type, and hardware). Understanding these concepts can elevate your database performance to a higher level!

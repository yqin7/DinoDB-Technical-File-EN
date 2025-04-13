# 🌟 DinoDB-Technical-File

DinoDB is a simple and efficient database system implementation, focusing on the development of core database components.

> **Note**: This repository contains technical documentation and executable artifacts for the DinoDB system, not the complete source code. These documents detail the system architecture, algorithm implementations, and working principles of key components. The executable files in the run_me_exe_files directory (dinodb.exe, dinodb_client.exe, dinodb_stress.exe) can be run directly without additional compilation.
>
> **It is recommended to use Typora to read all documents in this project.**

## 📚 Document Description

This project includes the following technical documents:

- **B+ Tree.md** - Detailed introduction to B+ tree index structure implementation, including node structure, insertion, query, deletion, and update
- **Concurrency for B+ Tree.md** - Explanation of B+ tree concurrent control implementation, focusing on pessimistic lock-crawling strategy
- **Join.md** - Explanation of hash join algorithm implementation and optimization based on Bloom filters
- **Pager.md** - Description of page manager design, including LRU cache mechanism and page replacement strategy
- **Transaction.md** - Elaboration of transaction manager and concurrency control design, including 2PL protocol and deadlock detection
- **RecoveryManager.md -** Detailed explanation of fault recovery mechanism based on WAL and simplified ARIES protocol

## 🚀 Project Features
* **📊 Data Structure**: B+ tree-based index structure, supporting efficient queries and range scans
* **⚡ Concurrency Control**: Using pessimistic lock-crawling strategy to implement efficient concurrent access to B+ trees, supporting read-read concurrency and read-write exclusion
* **🔄 Transaction Management**: Implementing strict 2PL (Two-Phase Locking) protocol with transaction manager to ensure transaction isolation
* **💾 Page Management**: Implementing page manager with LRU (Least Recently Used) cache mechanism to optimize memory usage
* **🔍 Join Algorithm**: Efficient query processing based on partitioned hash join, using Bloom filters to optimize performance
* **🔁 Fault Recovery**: Implementing database recovery mechanisms based on WAL (Write-Ahead Logging) and ARIES protocol to ensure database durability and consistency

## 🧩 Core Components

### 🌲 B+ Tree Index
B+ tree is a self-balancing tree data structure that supports insertion, deletion, and range query operations. The B+ tree implemented in this project has the following characteristics:
* Degree of 202
* All data stored in leaf nodes, internal nodes only store index keys
* Leaf nodes connected via right sibling pointers, supporting sequential scanning

Supported operations:
* Insert
* Find
* Update
* Delete
* SelectRange (range query)
* Select (full query)

**Concurrency Control:** Uses pessimistic lock-crawling strategy, allowing multiple read operations to execute concurrently while being mutually exclusive with write operations. Through the "Lock-Crabbing" mechanism, locks are dynamically acquired and released during tree traversal:

- When traversing, first lock the parent node, then lock the child node, and release the parent node lock when confirming the child node won't split
- Read operations use read locks (RLock), write operations use write locks (WLock), ensuring read-read concurrency
- Using node split propagation mechanism to handle dynamic balancing of tree structure in high-concurrency environments
- Achieves 118 times higher concurrent throughput in multi-threaded environments compared to single-threaded B+ tree implementation

### 🔄 Join Algorithm

Implementation based on partitioned hash join, features:

* Partitioning table data using hash functions
* Optimizing probe phase performance with Bloom filters
* Supporting multiple join modes such as key-key, key-value, value-key

Workflow:

1. Build phase: Create temporary hash indexes for left and right tables
2. Probe phase: Use Bloom filters and hash tables to find matching records

### 📝 Page Manager (Pager)

Pager is responsible for managing the interaction between in-memory data pages and disk files, main features:
* Implementing LRU cache mechanism to optimize memory usage
* Managing page states through PinnedList, UnpinnedList, and FreeList
* Providing interfaces for page acquisition, release, and refresh

LRU strategy:
* Newly accessed pages are placed at the end of pinnedList (most recently used)
* Pages no longer in use are placed at the end of unpinnedList (recently used but candidates for eviction)
* When new pages are needed, they are obtained from the beginning of freeList or unpinnedList (least recently used)

### 🔐 Transaction Manager
Implements transaction management based on strict 2PL protocol, ensuring ACID properties of the database:
* Managing resource and lock mapping relationships through ResourceLockManager
* Using WaitsForGraph for deadlock detection
* Supporting transaction start, commit, rollback, and lock/unlock operations

Main components:
* ResourceLockManager: Manages mapping between resources and corresponding mutex locks
* WaitsForGraph: Detects deadlocks between transactions using graph algorithms
* Transactions Map: Maintains active transactions and their held resource locks

### 📝 Recovery Manager

Implements fault recovery based on WAL (Write-Ahead Logging) and simplified ARIES protocol:

- Following the WAL principle of "write log first, then execute operation"
- Recording all data modification operations through a single log file
- Implementing three-phase recovery process: analysis, redo, and undo

Main functions:

- Log recording: Tracking all table creations, data modifications, and transaction state changes
- Checkpoint: Periodically creating consistency snapshots of the database to reduce recovery time
- Crash recovery: Rebuilding the database to a consistent state after system failure through logs
  - Analysis phase: Identifying active transactions at the time of crash
  - Redo phase: Re-executing all transaction operations
  - Undo phase: Rolling back operations of uncommitted transactions

## 🔧 Usage Instructions

### Running Environment

DinoDB is a self-contained database system developed in Go language, and the provided binary files (in the run_me_exe_files folder) can be run directly:

- **Operating System**: Supports Linux, macOS, and Windows
- **Dependencies**: No need to install additional Go language environment or other database systems
- **Permissions**: May need to add execution permissions for executable files (use `chmod +x` on Linux/Mac)

### Compilation

If you need to compile from source code (not necessary, you can directly use the provided pre-compiled binary files):

```bash
# Compile server
go build -buildvcs=false -o dinodb ./cmd/dinodb

# Compile client
go build -buildvcs=false -o dinodb_client ./cmd/dinodb_client

# Compile stress testing tool
go build -buildvcs=false -o dinodb_stress ./cmd/dinodb_stress
```

### Starting the Server

```bash
# Start the DinoDB server, specifying project and port
./dinodb -project recovery -p 8335
```

After the server starts successfully, it will display:
```
dinodb server started listening on localhost:8335
```

### Starting the Client

Start the client in another terminal window:

```bash
# Connect to the DinoDB server at the specified port
./dinodb_client -p 8335
```

After connecting successfully, you will see the REPL interface:
```
Welcome to the dinodb REPL! Please type '.help' to see the list of available commands.
dinodb> 
```

### Available Commands
Use the `.help` command to see all available commands:

```
dinodb> .help
transaction: Handle transactions. usage: transaction <begin|commit>
create: Create a table. usage: create <btree|hash> table <table>
select: Select elements from a table. usage: select from <table>
find: Find an element. usage: find <key> from <table>
checkpoint: Saves a checkpoint of the current database state and running transactions. usage: checkpoint
abort: Simulate an abort of the current transaction. usage: abort
pretty: Print out the internal data representation. usage: pretty
insert: Insert an element. usage: insert <key> <value> into <table>
update: Update en element. usage: update <table> <key> <value>
crash: Crash the database. usage: crash
delete: Delete an element. usage: delete <key> from <table>
lock: Grabs a write lock on a resource. usage: lock <table> <key>
```

### Command Usage Examples

Below are examples of some common operations:

#### Creating a Table
```
dinodb> create btree table test
```

#### Read-Only Operations
The following operations do not need transaction wrapping because they only read data and do not modify the database state:
```
dinodb> find 2 from test
dinodb> select from test
dinodb> pretty from test
```

#### Viewing Data Structure
```
dinodb> pretty from test
[0] Leaf (root) size: 4
 |--> (1, 10)
 |--> (2, 20)
 |--> (3, 30)
 |--> (5, 500)
```

#### Transaction Operations for Modifying Data
All operations that modify data need to be executed within a transaction:

##### Inserting Data
```
dinodb> transaction begin
dinodb> insert 1 10 into test
dinodb> insert 2 20 into test
dinodb> insert 3 30 into test
dinodb> transaction commit
```

##### Updating Data
```
dinodb> transaction begin
dinodb> update test 2 25
dinodb> transaction commit
```

##### Deleting Data
```
dinodb> transaction begin
dinodb> delete 3 from test
dinodb> transaction commit
```

##### Multi-Operation Transaction
```
dinodb> transaction begin
dinodb> insert 5 500 into test
dinodb> update test 1 15
dinodb> delete 3 from test
dinodb> transaction commit
```

#### Using Locks (Explicit Locking)
```
dinodb> transaction begin
dinodb> lock test 1
dinodb> lock test 2
// Execute other operations that need to access these resources
// Note: Explicit locking is usually not necessary, as operations like update and delete automatically acquire the necessary locks
dinodb> transaction commit
```

#### Database Recovery Operations

##### Simulating Database Crash

```
dinodb> crash
Connection to server lost. Please restart the client.
```

##### Simulating Transaction Abort

```
dinodb> transaction begin
dinodb> insert 7 700 into test
dinodb> abort
Transaction aborted.
```

##### Recovery Process Demonstration

```
# 1. Create table and add data
dinodb> transaction begin
dinodb> insert 1 100 into test
dinodb> insert 2 200 into test
dinodb> transaction commit

# 2. Create checkpoint
dinodb> checkpoint

# 3. More operations
dinodb> transaction begin
dinodb> insert 3 300 into test
dinodb> update test 1 150
dinodb> transaction commit

# 4. Simulate crash
dinodb> crash

# 5. Restart server
# In a new terminal:
./dinodb -project recovery -p 8335

# 6. Reconnect client
# In another terminal:
./dinodb_client -p 8335

# 7. View recovered data
dinodb> select from test
```

### Testing

```bash
# Complete testing (cannot test without code)
go test './test/concurrency/...' -race -timeout 180s -v

# Stress testing with 8 concurrent threads
./dinodb_stress -index=btree -workload=workloads/i-a-sm.txt -n=8 -verify

# Stress testing, if the above command reports an error, it is recommended to use an absolute path
./dinodb_stress -index=btree -workload="C:\Users\huo00\OneDrive\Documents\DinoDB-Technical-File\workloads\i-i-md.txt" -n=8 -verify
```

## 📊 Performance Test Results
* **B+ Tree Insertion**: In multi-threaded environments, sequential insertion performs better than random insertion
* **Join Operations**: Execution time increases linearly with data volume, and performance is stable across different match rates
* **Select Operations**: Performance is relatively stable between 1-8 threads, but performance degradation occurs with 16 threads

## 🔮 Future Outlook
* Add commands for range operations, such as greater than, less than, etc.

## 📫 Getting the Code
Due to course requirements (cannot publicly share code with future students), the source code cannot be made public at this time. If you are interested in the project, please send an email to huo000311@outlook.com to request the code.
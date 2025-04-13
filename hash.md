# Hash

# 1. Important Concepts

## 1.1 Extendible Hashing

- Extendible hashing is a dynamic hashing technique that can **expand according to data growth** without requiring complete rebuilding
- The extendible hashing implemented in this project uses two key concepts: **Directory** and **Bucket**
- Features:
  - **Progressive extension**: Only buckets that overflow will split, not the entire table being rebuilt
  - **Dynamic balancing**: Load balancing automatically adjusts, reducing collision probability
  - **Efficient lookup**: Regardless of data volume, lookup operations have constant time complexity O(1)

## 1.2 Global Depth and Local Depth

- **Global Depth**:
  - Definition: The number of bits of hash values used in the directory of the entire hash table
  - Function: Determines the size of the directory (2^global depth), increases as the table expands
  - Storage location: In the hash table metadata
  ```go
  type HashTable struct {
      globalDepth int64    // Global depth of the hash table
      buckets     []int64  // Array of bucket page numbers
      // ...other fields
  }
  ```

- **Local Depth**:
  - Definition: The number of bits of hash values used by a single bucket
  - Function: Indicates how many times a bucket has split
  - Storage location: In the header of each bucket's page
  ```go
  type HashBucket struct {
      localDepth int64     // Local depth of the bucket
      numKeys    int64     // Number of keys in the bucket
      // ...other fields
  }
  ```

- **Depth Relationship**:
  - Local depth of any bucket ≤ Global depth of the hash table
  - When a bucket's local depth reaches the global depth and needs to split, the global depth must be increased first

## 1.3 Relationship between Directory Entries, Hash Values, and Buckets

- **Directory Entry** is the basic unit of the hash table index structure, stored in the `buckets` array
- **Directory Entry Content**: Stores the page number of the physical bucket, not a direct reference to the bucket object
- **Mapping Process**: Key → Hash Value → Directory Index → Page Number → Physical Bucket

```
              Hash Table
         +----------------+
         |  globalDepth=2 |
         +----------------+      Directory Entries(binary)     Physical Buckets
         |    buckets     | ---> [00] ---> Page #0 ---> Bucket 0 (localDepth=1) 
         |                |      [01] ---> Page #1 ---> Bucket 1 (localDepth=2)
         |                |      [10] ---> Page #0 ---> Bucket 0 (Same as [00])
         |                |      [11] ---> Page #2 ---> Bucket 2 (localDepth=2)
         +----------------+
```

- **(! Important) Many-to-One Relationship (multiple directory entries pointing to one physical bucket)**:
  - In the above diagram, directory entries [00] and [10] point to the same physical bucket (bucket 0)
  - Reason: When local depth is less than global depth, it means the bucket only cares about matching keys in its lowest bits, ignoring higher bits. Global depth equals how many means there must be 2^global depth directory entries.
    - Bucket 0's local depth=1, only using the lowest 1 bit of the hash value for distinction
  - Directory entries [01] and [11] point to different buckets because these buckets have local depth=2, using all 2 bits
- **Hash Prefix Matching**:
  - For a bucket with depth d, all hash values sharing the same d-bit prefix are mapped to that bucket
  - Example: When bucket's localDepth=1, keys with the same lowest 1 bit in their hash values all map to the same bucket

## 1.4 Bucket Storage Structure

```go
const DEPTH_OFFSET int64 = 0
const DEPTH_SIZE int64 = binary.MaxVarintLen64
const NUM_KEYS_OFFSET int64 = DEPTH_OFFSET + DEPTH_SIZE
const NUM_KEYS_SIZE int64 = binary.MaxVarintLen64
const BUCKET_HEADER_SIZE int64 = DEPTH_SIZE + NUM_KEYS_SIZE
const ENTRYSIZE int64 = binary.MaxVarintLen64 * 2                        
const MAX_BUCKET_SIZE int64 = (PAGESIZE - BUCKET_HEADER_SIZE) / ENTRYSIZE
```

- All buckets are stored in fixed-size pages with the following structure:

  - **Bucket Header**:
    - `localDepth` (8 bytes): Local depth of the bucket
    - `numKeys` (8 bytes): Current number of keys stored in the bucket

  - **Entry Area**:
    - Each entry is 16 bytes (key 8 bytes, value 8 bytes)
    - Based on page size, a bucket can typically hold about 250-300 entries

- Page layout example (4096-byte page):
  ```
  +------------------------+
  | localDepth (8 bytes)   |  binary.MaxVarintLen64 = 10 bytes
  +------------------------+                                ===> Header equals 20 bytes
  | numKeys (8 bytes)      |  binary.MaxVarintLen64 = 10 bytes
  +------------------------+
  | entry 0 (16 bytes)     |
  | key (8 bytes) | val (8 bytes) |
  +------------------------+
  | entry 1 (16 bytes)     |
  | key (8 bytes) | val (8 bytes) |
  +------------------------+
  |        ...            |
  +------------------------+
  | entry 254 (16 bytes)   |
  | key (8 bytes) | val (8 bytes) |
  +------------------------+
  ```

# 2. Core Fields

## 2.1 HashTable

```go
type HashTable struct {
    globalDepth int64        // Global depth of the hash table
    buckets     []int64      // Array of bucket page numbers, index (in binary form) corresponds to lookup keys in the hash table
    pager       *pager.Pager // Pager associated with the hash table
    rwlock      sync.RWMutex // Read-write lock for the hash table
}
```

### A. globalDepth

- Function: Records the global depth of the entire hash table, i.e., the number of bits of hash values used
- Purpose:
  - Determines the directory size of the hash table (2^globalDepth)
  - Limits the maximum local depth of buckets
  - Guides hash value calculation and bucket lookup
- Example:
  ```go
  // Calculate the hash value for a given key, using global depth
  hash := Hasher(key, table.globalDepth)
  ```

### B. buckets

- Function: Stores an array of page numbers for all bucket pages, effectively the directory of the hash table
- Purpose:
  - Maps hash values to corresponding buckets
  - Supports multiple directory entries pointing to the same bucket (buckets sharing a prefix)
  - When the table expands, the array size doubles
- Example:
  ```go
  // Get the corresponding bucket page number based on hash value
  pagenum := table.buckets[hash]
  bucket, err := table.GetBucketByPN(pagenum)
  ```

### C. pager

- Function: Pointer to the page storage manager
- Purpose:
  - Manages page transfer between disk and memory
  - Provides operations for creating, retrieving, and releasing pages
  - Handles caching and dirty page flushing
- Example:
  ```go
  // Get a new page
  newPage, err := pager.GetNewPage()
  // Get an existing page
  page, err := pager.GetPage(pageNum)
  // Release a page
  pager.PutPage(page)
  ```

### D. rwlock

- Function: Hash table level read-write lock
- Purpose:
  - Protects concurrent access to the hash table structure
  - Allows multiple read operations to occur concurrently
  - Ensures the mutual exclusivity of write operations (such as splitting)
- Example:
  ```go
  // Acquire read lock
  table.RLock()
  // Release read lock
  table.RUnlock()
  // Acquire write lock
  table.WLock()
  // Release write lock
  table.WUnlock()
  ```

## 2.2 HashBucket

```go
type HashBucket struct {
    localDepth int64       // Local depth of the bucket
    numKeys    int64       // Number of keys/entries in the bucket
    page       *pager.Page // Page containing bucket data
}
```

### A. localDepth

- Function: Records the local depth of the bucket, representing the length of hash prefix used by the bucket to distinguish elements
- Purpose:
  - Indicates how many times the bucket has split
  - Controls which directory entries point to this bucket
  - Determines how records are reallocated during splitting
- Example:
  ```go
  // Increase the bucket's local depth (during splitting)
  bucket.updateLocalDepth(bucket.localDepth + 1)
  ```

### B. numKeys

- Function: Records the current number of key-value pairs stored in the bucket
- Purpose:
  - Tracks the fill level of the bucket
  - Determines whether splitting is needed (when numKeys >= MAX_BUCKET_SIZE)
  - Helps iterate through all entries in the bucket
- Example:
  ```go
  // Determine if splitting is needed after insertion
  split := bucket.numKeys >= MAX_BUCKET_SIZE
  ```

### C. page

- Function: Points to the physical page storing the bucket data
- Purpose:
  - Provides direct access to bucket data
  - Manages page locking and releasing
  - Tracks whether the page has been modified (dirty page)
- Example:
  ```go
  // Get page number
  pageNum := bucket.page.GetPageNum()
  // Update page data
  bucket.page.Update(data, offset, size)
  ```

## 2.3 HashIndex

```go
type HashIndex struct {
    table *HashTable   // Underlying hash table
    pager *pager.Pager // Pager supporting this index/hash table
}
```

### A. table

- Function: Pointer to the underlying hash table
- Purpose: Provides access to core operations of the hash table

### B. pager

- Function: Pointer to the pager
- Purpose: Manages disk storage and memory cache for the table

# 3. Core Functions

## 3.1 Insert Operation

```go
func (table *HashTable) Insert(key int64, value int64) error
```

### A. Parameter Introduction

- Parameters:
  - `key int64`: Key to insert
  - `value int64`: Associated value
- Returns:
  - `error`: If insertion fails, returns an error; otherwise returns nil
- Purpose:
  - Inserts a new key-value pair into the hash table, triggering bucket splitting if needed

### B. Complete Flow

**1. Acquire Table-Level Write Lock**

- Call `table.WLock()` to acquire the hash table's write lock, ensuring thread safety throughout the insertion process
- Use `defer table.WUnlock()` to ensure the lock is released when the function ends

**2. Calculate Hash Value**

- Calculate the hash value for the key using global depth: `hash := Hasher(key, table.globalDepth)`
- The hash function ensures the value is within the directory size range (0 to 2^globalDepth-1)

**3. Get Target Bucket**

- Find the corresponding bucket's page number from the directory: `table.buckets[hash]`
- Get and lock the bucket: `bucket, err := table.GetAndLockBucket(hash, WRITE_LOCK)`
- Ensure resources are released when the function ends: `defer table.pager.PutPage(bucket.page)` and `defer bucket.WUnlock()`

**4. Perform Insertion**

- Call the bucket's insertion method: `split := bucket.Insert(key, value)`
- The insertion method adds the entry to the bucket and returns whether splitting is needed

**5. Handle Bucket Splitting**

- If splitting is not needed (split is false), return directly
- If splitting is needed, call `table.split(bucket, hash)` to handle the splitting logic

### C. Example

- Execute `table.Insert(42, 100)`, inserting key 42, value 100
- Assuming key 42 has a hash value of 5, look up the bucket with page number `table.buckets[5]`
- Insert entry (42, 100) into that bucket
- If the bucket is full after insertion, trigger the splitting operation

## 3.2 Split Bucket Operation

```go
func (table *HashTable) split(bucket *HashBucket, hash int64) error
```

### A. Parameter Introduction

- Parameters:
  - `bucket *HashBucket`: Bucket that needs to split
  - `hash int64`: Hash value that triggered the split
- Returns:
  - `error`: If splitting fails, returns an error; otherwise returns nil
- Purpose:
  - Splits a full bucket into two and reallocates its entries

### B. Complete Flow

**1. Calculate New and Old Hash Values**

- Calculate old hash suffix: `oldHash := (hash % powInt(2, bucket.localDepth))`
- Calculate new hash suffix: `newHash := oldHash + powInt(2, bucket.localDepth)`
- These suffix values are used to determine which bucket pointers need to be updated

**2. Check and Extend Table**

- If the bucket's local depth equals the table's global depth: `bucket.localDepth == table.globalDepth`
- Then call `table.ExtendTable()` to increase the table's global depth and double the directory size

**3. Create New Bucket**

- Increase the original bucket's local depth: `bucket.updateLocalDepth(bucket.localDepth + 1)`
- Create a new bucket with the same local depth: `newBucket, err := newHashBucket(table.pager, bucket.localDepth)`
- Ensure the new bucket's page is released when the function ends: `defer table.pager.PutPage(newBucket.page)`

**4. Reallocate Entries**

- Temporarily store all entries: `tmpEntries := make([]entry.Entry, bucket.numKeys)`
- Iterate through entries and reallocate based on the new hash depth:
  ```go
  for _, entry := range tmpEntries {
    if Hasher(entry.Key, bucket.localDepth) == newHash {
        newBucket.modifyEntry(newNKeys, entry)
        newNKeys++
    } else {
        bucket.modifyEntry(oldNKeys, entry)
        oldNKeys++
    }
  }
  ```
- Update the number of keys in both buckets:
  ```go
  bucket.updateNumKeys(oldNKeys)
  newBucket.updateNumKeys(newNKeys)
  ```
  
- **Allocation Rules**:
  - **If an entry's new hash value matches the new hash prefix (typically the highest bit is 1), move it to the new bucket**
  - **If an entry's new hash value matches the original hash prefix (typically the highest bit is 0), keep it in the original bucket**
  - This ensures entries can still be found using the same hash prefix

**5. Update Directory Pointers**

- Update all directory entries pointing to the new hash suffix to point to the new bucket:
  ```go
  power := bucket.localDepth
  for i := newHash; i < powInt(2, table.globalDepth); i += powInt(2, power) {
    table.buckets[i] = newBucket.page.GetPageNum()
  }
  ```

**6. Recursively Handle Extreme Cases**

- Check if the buckets after splitting still overflow:
  ```go
  if oldNKeys >= MAX_BUCKET_SIZE {
    return table.split(bucket, oldHash)
  }
  if newNKeys >= MAX_BUCKET_SIZE {
    return table.split(newBucket, newHash)
  }
  ```
- If either bucket still overflows, recursively call split to handle it

### C. Design Considerations

**1. Why Recursive Splitting is Needed**

- **Uneven Data Distribution**: Hash functions may cause many keys to map to the same bucket
- **Extreme Case Handling**: After splitting, all or most elements may still be in the same bucket
- **Robustness**: Recursion ensures a stable state is reached regardless of data distribution

### D. Bucket Split Diagram Explanation

```
Before Splitting:
              Hash Table
         +----------------+
         |  globalDepth=2 |
         +----------------+      Directory Entries(binary)     Physical Buckets
         |    buckets     | ---> [00] ---> Page #0 ---> Bucket 0 (localDepth=2, full)
         |                |      [01] ---> Page #1 ---> Bucket 1
         |                |      [10] ---> Page #2 ---> Bucket 2
         |                |      [11] ---> Page #3 ---> Bucket 3
         +----------------+

After Splitting (Bucket 0 splits, Global Depth increases):
              Hash Table
         +----------------+
         |  globalDepth=3 |
         +----------------+      Directory Entries(binary)     Physical Buckets
         |    buckets     | ---> [000] ---> Page #0 ---> Bucket 0' (localDepth=3)
         |                |      [001] ---> Page #1 ---> Bucket 1
         |                |      [010] ---> Page #2 ---> Bucket 2
         |                |      [011] ---> Page #3 ---> Bucket 3
         |                |      [100] ---> Page #4 ---> Bucket 4 (new bucket, localDepth=3)
         |                |      [101] ---> Page #1 ---> Bucket 1
         |                |      [110] ---> Page #2 ---> Bucket 2
         |                |      [111] ---> Page #3 ---> Bucket 3
         +----------------+
```

- The above diagram shows the bucket splitting process of an extendible hash table:

  - **Initial State**: Global depth is 2, with 4 directory entries (00,01,10,11) pointing to different buckets

  - **Bucket Split Timing**: When a bucket is full, bucket.numKeys >= MAX_BUCKET_SIZE, where MAX_BUCKET_SIZE = (PAGESIZE - BUCKET_HEADER_SIZE) / ENTRYSIZE = (4096 - 20) / 20 = 203
    - Increase the bucket's local depth
    - Create a new bucket
    - Reallocate records based on the increased bit
    - If local depth equals global depth, the directory size may need to double

  - **After Splitting**:
    - Original and new buckets are distinguished based on longer hash prefixes
    - The directory may grow to accommodate more buckets
    - Some directory entries may point to the same bucket (if they share the same hash prefix)

## 3.3 Find Operation

```go
func (table *HashTable) Find(key int64) (entry.Entry, error)
```

### A. Parameter Introduction

- Parameters:
  - `key int64`: Key to find
- Returns:
  - `entry.Entry`: Found entry object
  - `error`: If the key doesn't exist, returns an error; otherwise returns nil
- Purpose:
  - Find the entry corresponding to the key in the hash table

### B. Complete Flow

**1. Acquire Table-Level Read Lock**

- Call `table.RLock()` to acquire the hash table's read lock
- Ensure the lock is released when returning

**2. Calculate Hash Value**

- Calculate the hash value for the key: `hash := Hasher(key, table.globalDepth)`
- Check if the hash value is valid

**3. Get Target Bucket**

- Get and lock the bucket: `bucket, err := table.GetAndLockBucket(hash, READ_LOCK)`
- Release the table-level read lock, allowing other operations to continue
- Ensure resources are released when the function ends: `defer table.pager.PutPage(bucket.page)` and `defer bucket.RUnlock()`

**4. Find in Bucket**

- Call the bucket's find method: `foundEntry, found := bucket.Find(key)`
- If not found, return an error
- If found, return the entry

## 3.4 Update Operation

```go
func (table *HashTable) Update(key int64, value int64) error
```

### A. Parameter Introduction

- Parameters:
  - `key int64`: Key to update
  - `value int64`: New value
- Returns:
  - `error`: If update fails, returns an error; otherwise returns nil
- Purpose:
  - Update the value of an existing key in the hash table

### B. Complete Flow

**1. Acquire Table-Level Read Lock**

- Call `table.RLock()` to acquire the hash table's read lock
- Release table lock after getting the bucket

**2. Calculate Hash Value and Get Bucket**

- Calculate the hash value for the key
- Get and lock the bucket: `bucket, err := table.GetAndLockBucket(hash, WRITE_LOCK)`

**3. Perform Update**

- Call the bucket's update method: `err2 := bucket.Update(key, value)`
- If the key doesn't exist, return an error

## 3.5 Delete Operation

```go
func (table *HashTable) Delete(key int64) error
```

### A. Parameter Introduction

- Parameters:
  - `key int64`: Key to delete
- Returns:
  - `error`: If deletion fails, returns an error; otherwise returns nil
- Purpose:
  - Remove the specified key-value pair from the hash table

### B. Complete Flow

Similar to Update, but calls the bucket's Delete method:
```go
err2 := bucket.Delete(key)
```

# 4. Concurrency Control

## 4.1 Locking Strategy

- **Two-Level Locking**:

  - Table-Level Lock: Protects the entire hash table structure
  - Bucket-Level Lock: Protects the contents of individual buckets

- **Lock Types**:

  ```go
  type BucketLockType int
  const (
      NO_LOCK    BucketLockType = 0
      WRITE_LOCK BucketLockType = 1
      READ_LOCK  BucketLockType = 2
  )
  ```

- **Hierarchical Locking**:

  - Read Operations: First acquire table read lock, then bucket read lock, then release table lock
  - Write Operations: Acquire table write lock, then bucket write lock
  - Split Operations: Maintain table write lock, acquire bucket write lock

## 4.2 Lock Types Used by Basic Operations

| Operation      | Table-Level Lock | Bucket-Level Lock | Lock Holding Strategy                                     |
| -------------- | ---------------- | ----------------- | --------------------------------------------------------- |
| **Find**       | Read (RLock)     | Read (RLock)      | Release table lock after getting bucket, release bucket lock after operation |
| **Insert**     | Write (WLock)    | Write (WLock)     | Maintain table and bucket locks throughout the operation until completion |
| **Update**     | Read (RLock)     | Write (WLock)     | Release table lock after getting bucket, release bucket lock after operation |
| **Delete**     | Read (RLock)     | Write (WLock)     | Release table lock after getting bucket, release bucket lock after operation |

**Special Note**:

- Insert operation uses a table-level write lock because it may trigger bucket splitting and table extension
- Update and Delete only modify the contents of a single bucket, not the table structure, so they only need a table read lock

## 4.3 Lock Usage Patterns

**1. Table-Level Lock Methods**

```go
func (table *HashTable) WLock()   { table.rwlock.Lock() }
func (table *HashTable) WUnlock() { table.rwlock.Unlock() }
func (table *HashTable) RLock()   { table.rwlock.RLock() }
func (table *HashTable) RUnlock() { table.rwlock.RUnlock() }
```

**2. Bucket-Level Lock Methods**

```go
func (bucket *HashBucket) WLock()   { bucket.page.WLock() }
func (bucket *HashBucket) WUnlock() { bucket.page.WUnlock() }
func (bucket *HashBucket) RLock()   { bucket.page.RLock() }
func (bucket *HashBucket) RUnlock() { bucket.page.RUnlock() }
```

**3. Safe Bucket Acquisition**

```go
func (table *HashTable) GetAndLockBucket(hash int64, lock BucketLockType) (*HashBucket, error)
```

# 5. Persistence

## 5.1 Structure Persistence

- **Hash Table Metadata**:
  - Stored in `<tablename>.meta` file
  - Contains global depth and bucket page number array
  
- **Bucket Data**:
  - Stored in `<tablename>` main file
  - Each bucket occupies one page

## 5.2 Reading and Writing

- **Reading Hash Table**:
  ```go
  func ReadHashTable(bucketPager *pager.Pager) (*HashTable, error)
  ```
  
- **Writing Hash Table**:
  ```go
  func WriteHashTable(bucketPager *pager.Pager, table *HashTable) error
  ```

# 6. Hash Table vs B+ Tree

**Hash Table Advantages:**

1. **Lookup Performance**
   - Hash Table: O(1) constant time complexity, unaffected by record count
   - B+ Tree: O(log n) logarithmic time complexity, increases with record count

2. **Simple Operations**
   - Hash Table: Simple key-value lookups are very efficient
   - B+ Tree: Needs to maintain complex tree structure and balance

**Hash Table Disadvantages:**

1. **Range Queries**
   - Hash Table: Doesn't support range queries, requires full table scan
   - B+ Tree: Naturally supports range queries, leaf node linked list allows quick traversal

2. **Sorted Iteration**
   - Hash Table: Cannot guarantee sequential access
   - B+ Tree: Supports iterating data in key order

3. **Space Utilization**
   - Hash Table: Buckets may not be fully utilized, especially after splitting
   - B+ Tree: Node fill factor is typically high (>50%)

4. **Extension Overhead**
   - Hash Table: Extension may require doubling directory size
   - B+ Tree: Progressive growth, no need for large-scale structural changes

**Application Scenario Selection:**

- Use Hash Table for:
  - Workloads dominated by point queries
  - Simple key-value storage
  - Unordered data storage
  
- Use B+ Tree for:
  - Frequent range queries
  - Need for sorted data access
  - Prefix search functionality
  - High space efficiency requirements
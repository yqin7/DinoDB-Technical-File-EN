# Pager

# 1. Important Concepts

## 1.1 LRU Implementation

1. Newly accessed pages are placed at the end of the pinnedList (most recently used)

   Pages no longer in use are placed at the end of the unpinnedList (recently used) (candidates for eviction)

   When obtaining a new page, if the freeList has no free pages, get from the beginning of the unpinnedList (least recently used)

2. Databases typically use a "dirty page delayed write-back" strategy. When a page is modified, it is not immediately written back to disk, but rather marked as a dirty page and kept in the unpinnedList and retained in memory for as long as possible, so that if it needs to be read again, it can be read from memory. LRU ensures that recently modified hot pages are kept in memory as much as possible.

3. Note: The freeList is not replenished; it is only used as an initial pool of free pages. After the freeList is emptied, if a new page is required, it is obtained from the unpinnedList after flushing to disk.

## 1.2 Reasons for Not Using LFU

1. LFU easily falls into a "frequency trap": pages that were frequently accessed in the past but are no longer used will occupy memory for a long time due to high frequency counts.

2. LFU is not friendly to newly added but important pages. That is, important pages newly added to the unpinnedList start with a frequency count of 1, and when `GetNewPage()` is called, they will be immediately flushed to disk and removed from memory. If that page needs to be accessed again, it must be accessed from disk.
3. More complex implementation, requiring additional data structures to maintain frequency counts and sorting

# 2. Important Parameters

## 2.1 freeList

```go
freeList     *list.List // List is a linked list, each node is L
```

A list of pre-allocated but unused pages, these pages have memory allocated but have not been used yet

## 2.2 unpinnedList 

```go
unpinnedList *list.List
```

A list of unpinned pages, these pages are already in memory but are not currently being used (have been released, pinCount = 0), and can be evicted

## 2.3 pinnedList      

```go
pinnedList   *list.List
```

A list of pinned pages, these are pages currently being used by the database (not yet released, (pinCount > 0))

## 2.4 pageTable

```go
pageTable map[int64]*list.Link
```

PageTable is a memory cache index that tracks all pages in memory. It does not distinguish whether pages have been flushed to disk, only caring whether the page is in memory

Its structure is:

- key: int64 type, representing the page number (pagenum)
- value: *list.Link type, a pointer to a linked list node, the value of each linked list node is a page

# 3. Complete Parameters

```go
type Pager struct {
    file         *os.File   // File descriptor for the disk file supporting the pager
    numPages     int64      // Total number of pages accessible by the pager (including both on disk and in memory)
    freeList     *list.List // List of pre-allocated but not yet used free pages
    unpinnedList *list.List // List of pages in memory that have not been evicted but are not currently being used
    pinnedList   *list.List // List of memory pages currently being used by the database
    // Page table mapping page numbers to corresponding pages (stored in the node of the list to which the page belongs)
    pageTable map[int64]*list.Link
    ptMtx     sync.Mutex    // Mutex to protect concurrent access to the page table
}
```

```go
type Page struct {
    pager      *Pager       // Pointer to the pager to which this page belongs
    pagenum    int64        // Unique identifier for the page, also represents its storage location in the pager file
    pinCount   atomic.Int64 // Active reference count for this page
    dirty      bool         // Flag indicating whether the page data has been changed and needs to be written to disk
    rwlock     sync.RWMutex // Read-write lock for the page structure itself
    data       []byte       // Serialized data (actual 4096-byte page content)
    updateLock sync.Mutex   // Mutex for updating page data
}
```

# 4. Functions

## 4.1 GetNewPage Main Function for Getting a New Page

```go
func (pager *Pager) GetNewPage() (page *Page, err error)
```

### A. Parameter Introduction

- Parameters:
  - None
- Returns:
  - `page *Page`: Newly allocated page
  - `error`: If allocation fails, returns an error; otherwise returns nil
- Purpose:
  - Allocate a new page, using the next available page number

### B. Complete Flow

**1. Acquire Mutex Lock**

- Call `pager.ptMtx.Lock()` to acquire the page table mutex lock, ensuring thread safety
- Use `defer pager.ptMtx.Unlock()` to ensure the lock is released when the function ends

**2. Request New Page**

- Call internal function `pager.newPage(pager.numPages)` to request a new page
- The parameter is the current `numPages`, used as the new page number
- If no page is available, return an error

**3. Initialize Page**

- Mark the page as dirty (dirty = true), ensuring data will eventually be written to disk
- Set the page number to the current numPages value
- **Reference count has already been set to 1 in newPage**

**4. Update Data Structures**

- Add the new page to the end of the pinnedList
- Create a mapping from page number to page in the pageTable
- Increment the pager's numPages count, preparing for the next page allocation

**5. Return Result**

- Return the newly allocated page and nil error

## 4.2 NewPage Internal Function for Getting a New Page

```go
func (pager *Pager) newPage(pagenum int64) (newPage *Page, err error) 
```

### A. Parameter Introduction

- Parameters:
  - `filePath string`: Path to the database file
- Returns:
  - `pager *Pager`: Created Pager object
  - `error`: If creation fails, returns an error; otherwise returns nil
- Purpose:
  - Create and initialize a new Pager object, open or create the specified database file

### B. Complete Flow

1. First try to get an available page from the freeList (free page list)
2. If the freeList is empty, try to evict a page from the unpinnedList (unpinned page list)

     - **After getting the page, need to flush its data to disk first (this is the only disk flushing timing in the page manager, the project uses lazy flushing)**

     - Remove the old mapping of the page from the pageTable


3. If pages cannot be obtained by either method, return an ErrRanOutOfPages error

4. After successfully obtaining a page:

     - Update the page number

     - Reset the dirty page flag

     - Set the reference count to 1


## 4.3 GetPage Get an Existing Page

```go
func (pager *Pager) GetPage(pagenum int64) (page *Page, err error)
```

### A. Parameter Introduction

- Parameters:
  - `pagenum int64`: Page number to get
- Returns:
  - `page *Page`: Retrieved page
  - `error`: If retrieval fails, returns an error; otherwise returns nil
- Purpose:
  - Get the page with the specified page number, loading it from disk if not in memory

### B. Complete Flow

**1. Acquire Mutex Lock**

- Call `pager.ptMtx.Lock()` to acquire the page table mutex lock
- Use `defer pager.ptMtx.Unlock()` to ensure the lock is released when the function ends

**2. Verify Page Number (pagenum) Validity**

**3. Look for the Page in Memory (pageTable):**

- If it exists and is in the unpinnedList, transfer it to the end of the pinnedList
  - Update pageTable mapping relationship
  - **Call `page.Get()` to increment reference count before returning**


- No changes if in pinnedList

**4. Page Not in Memory (pageTable):**

- **Call `pager.newPage(pagenum)` to get a new page frame, in the `newPage(pagenum)` function the reference count will be set to 1**
- Set page number and dirty page flag
- Call `pager.FillPageFromDisk(page)` to load data from disk
- If loading fails, return the page to the freeList and return an error
- Add the page to the pinnedList and update the pageTable

## 4.4 PutPage Release Page Reference

```go
func (pager *Pager) PutPage(page *Page) (err error)
```

### A. Parameter Introduction

- Parameters:
  - `page *Page`: Page to release
- Returns:
  - `error`: If release fails, returns an error; otherwise returns nil
- Purpose:
  - Decrease the page's reference count, moving the page from pinnedList to unpinnedList if the count reaches 0

- Use Case: When an operation (query, update) completes access to a page, it needs to release its reference to that page to avoid indefinitely occupying memory

### B. Complete Flow

**1. Acquire Mutex Lock**

- Call `pager.ptMtx.Lock()` to acquire the page table mutex lock
- Use `defer pager.ptMtx.Unlock()` to ensure the lock is released when the function ends

**2. Decrease Reference Count**

- Call `page.Put()` to decrease the page's reference count and get the return value
- Check if the return value is less than 0, less than 0 indicates an error (unbalanced reference count)

**3. Check for Reference Count of 0**

- Get the linked list node corresponding to the page in the pageTable

- Remove the node from the pinnedList

- Add the page to the end of the unpinnedList

- Update the pageTable, creating a key-value mapping between pageNum and the new node in the unpinnedList.

## 4.5 FlushPage Write Page Back to Disk

```go
func (pager *Pager) FlushPage(page *Page)
```

### A. Parameter Introduction

- Parameters:
  - `page *Page`: Page to flush
- Returns:
  - None
- Purpose:
  - If the page is marked as dirty, write its contents back to disk

### B. Complete Flow

**1. Check if the Page is Dirty**

- Call `page.IsDirty()` to check if the page has been modified

- If the page is not dirty, the function returns directly without any operation

**2. Write to Disk**

- Calculate the page's offset position in the file (pagenum * Pagesize), write position = page number * page size, write content = page data
- Use `pager.file.WriteAt()` to write the page data to the correct file position
- The data written is the complete content of the page (`page.data`)

**3. Reset Dirty Flag**

- Call `page.SetDirty(false)` to clear the dirty flag
- Indicates that the page's memory content and disk content are now consistent

## 4.6 FlushAllPages

```go
func (pager *Pager) FlushAllPages()
```

### A. Parameter Introduction

- Parameters:
  - None
- Returns:
  - None
- Purpose:
  - Write all modified pages (dirty pages) back to disk
  - Batch data flushing, called when the system shuts down or creates a checkpoint, ensuring all modified data is saved to disk.

### B. Complete Flow

**1. Define Handler Function**

- Create a function to handle each page list node
- The function gets the page object from the node
- Call `pager.FlushPage(page)` for each page

**2. Traverse Pinned Page List**

- Call `pager.pinnedList.Map(writer)` to apply the handler function to all pages in the pinnedList
- Check and flush all dirty pages that are in use

**3. Traverse Unpinned Page List**

- Call `pager.unpinnedList.Map(writer)` to apply the handler function to all pages in the unpinnedList
- Check and flush all replaceable but not yet replaced dirty pages

**4. Complete Flushing**

- All dirty pages have been written to disk
- The dirty flag of all pages has been reset
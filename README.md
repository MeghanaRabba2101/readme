# CS525 – Advanced Database Organization

## Assignment 3 – Record Manager Implementation

**Student:**  
Meghana Rabba (A20572009)  
Rohan Singh Rajendra Singh (A20572007)  
**Course:** CS525 (Fall 2025)

## A. Objective

The goal of this assignment was to design and implement a **Record Manager** layer that supports schema management, record-level CRUD operations, scanning, and persistent metadata storage on top of the provided **Storage Manager** and **Buffer Manager** layers.  
The Record Manager must serialize schemas, manage free slots efficiently, maintain page headers, and guarantee record consistency and primary key uniqueness.

## B. Data Structures Implemented

### 1. rm_file_header

We implemented this structure to hold all **table-level metadata**.  
It is physically stored at the beginning of page 0 in the table file.  
The fields include:  
- total_live_rows  
- payload_size_bytes  
- total_pages_in_file  
- next_tid_counter  
- first_free_page  

We update this structure after every insert, delete, and page addition, and it is written to disk using `write_header_and_schema()`.

### 2. rm_page_header

Each data page begins with this structure.  
It tracks how many record slots are free (`free_slots`) and the link to the next free page (`next_free_page`).  
We implemented `init_data_page_header()` to initialize it with all slots free and later modify it whenever slot availability changes.

### 3. rm_table_mgmt

This structure manages an opened table in memory.  
It holds:  
- a dedicated buffer pool  
- a reusable page handle  
- schema pointer  
- file name  
- cached rm_file_header  

This struct is created in `openTable()` using `calloc`, stored in `rel->mgmtData`, and freed in `closeTable()`.

### 4. rm_scan_mgmt

This structure stores the scan state.  
It keeps the predicate expression, the last scanned RID, and a flag to track whether the scan has started.  
Created in `startScan()` and destroyed in `closeScan()`.

## C. Helper Functions

These internal static functions make the Record Manager modular and reusable. They were implemented first to simplify page and schema operations.

### 1. bytes_for_nullmap

We implemented this to calculate how many bytes are needed for the null bit map in a record.  
It takes the number of attributes and returns `(numAttr + 7) / 8`, rounding bits to bytes.  
This ensures correct offset alignment for variable schemas.

### 2. compute_slot_length

This computes the physical space needed per record slot.  
We sum the size of flag, TID, nullmap, and payload width (from `hdr->payload_size_bytes`).  
This ensures every slot has identical byte length and fixed offsets.

### 3. usable_bytes_in_data_page

This simply returns the usable area within a page by subtracting the size of `rm_page_header` from the constant `PAGE_SIZE`.  
We use it to calculate how many slots can fit per page.

### 4. slots_per_page

We divide usable bytes by slot length to determine slot count.  
This is used to initialize new pages and for iteration in insert and scan operations.

### 5. offset_of_attr

This calculates the byte offset of a given attribute in the payload.  
We iterate over all attributes before the target one, adding:  
- 4 bytes for DT_INT or DT_FLOAT  
- 1 byte for DT_BOOL  
- typeLength[i] for DT_STRING  
This offset is then used in `getAttr()` and `setAttr()` to access values efficiently.

### 6. data_area_ptr

Implemented to return a pointer to the start of data slots in a page, skipping the header.  
This keeps all slot computations modular.

### 7. slot_ptr

Computes a slot's starting address using:  
`data_area_ptr(page) + slot_index * slot_length`.  
We use this pointer to locate flag, tid, nullmap, and payload sections of a record.

### 8. slot_flag_ptr, slot_tid_ptr, slot_nullmap_ptr, slot_payload_ptr

We implemented these four helpers to return byte pointers to each logical region of a slot.  
They perform simple pointer arithmetic over the slot start address, ensuring consistent access when writing or reading flags, TIDs, and payloads.

### 9. serialize_schema_into

Implemented to store schema metadata compactly into page 0.  
We manually write into a byte buffer the number of attributes, each name length and name, data types, type lengths, key size, and key indices.  
We used pointer arithmetic (`char *p`) and macros `NEED()` and `WRI()` to ensure no buffer overflow occurs.  
This is called from `write_header_and_schema()` to persist schema.

### 10. deserialize_schema_from

This reverses serialization: it reads schema bytes and rebuilds the Schema structure dynamically.  
We read number of attributes, then allocate and copy each attribute name, data type, and key field.  
It is called from `read_header_and_schema()` during `openTable()`.

### 11. write_header_and_schema

We implemented this to write both metadata and schema into page 0 in one operation.  
It pins page 0 via buffer manager, clears the page, copies the file header, serializes schema into remaining bytes, marks page dirty, and unpins it.

### 12. read_header_and_schema

Reads both the rm_file_header and schema bytes from page 0 and reconstructs them using `deserialize_schema_from()`.  
This keeps the open-table operation consistent with its creation metadata.

### 13. init_data_page_header

Used when a new data page is appended.  
We pin the page, create an rm_page_header with free_slots equal to slots_per_page(), zero out the rest of the page, and unpin after marking dirty.

### 14. pull_data_page_header and 15. push_data_page_header

These are symmetric helpers for reading and writing rm_page_header.  
Both pin a page, perform memcpy, and unpin with optional markDirty().

### 16. free_list_remove_if_full

We implemented this to keep the free-page linked list accurate.  
When a page becomes full (free_slots = 0), it removes that page from the free list.  
If it was the head, we update `hdr.first_free_page`.  
Otherwise, we traverse all pages until we find the one linking to it and bypass it.

### 17. ensure_at_least_one_data_page

Checks if the table has at least one data page (page ≥ 1).  
If not, opens file using `openPageFile()`, appends a block, and initializes it using `init_data_page_header()`.  
Updates total_pages_in_file and writes updated header and schema back to disk.

### 18. find_insertion_slot

Implements the algorithm to find the next available slot for record insertion.  
If `hdr.first_free_page == -1`, we append a new page, initialize it, and link it as the new free list head.  
Then we pin that page, scan through its slots for one with `RM_SLOT_EMPTY` or `RM_SLOT_TOMBSTONE`, assign RID, and unpin.  
If a slot isn't found (header inconsistency), we set free_slots to 0 and retry recursively.

### 19. pk_payloads_equal

Compares payloads on all primary key attributes.  
For INT/FLOAT/BOOL we use memcpy to compare values; for STRING we use `strncmp` on declared length.  
Returns true if all key fields match.

### 20. enforce_pk_uniqueness

Before insert or update, we scan all pages and slots for live records and call `pk_payloads_equal()` against candidate payloads.  
If a duplicate is found (and RID differs), returns RC_FAILED.  
This ensures no duplicate primary key violations occur.

## D. Record Manager API Functions

### 1. initRecordManager

Implemented as a one-line bootstrapper that calls `initStorageManager()` to prepare the storage subsystem.

### 2. shutdownRecordManager

Placeholder returning RC_OK; no cleanup required because Record Manager does not maintain global state.

### 3. createTable

We implemented this by calling `createPageFile()` to create a new file and writing an initial rm_file_header.  
We computed payload size using `getRecordSize()`, then initialized a small buffer pool and used `write_header_and_schema()` to store schema and metadata in page 0.  
The buffer pool is shut down at the end to persist all changes.

### 4. openTable

This allocates an rm_table_mgmt structure, initializes buffer pool, and calls `read_header_and_schema()` to rebuild schema and header.  
If the schema payload size was zero, it recomputes and writes back updated metadata.  
Finally, it ensures a data page exists by calling `ensure_at_least_one_data_page()`.

### 5. closeTable

Pins page 0, writes updated header + schema using `write_header_and_schema()`, then calls `shutdownBufferPool()`.  
Finally, frees file_name, schema pointer, and rm_table_mgmt structure.

### 6. deleteTable

Implemented as a direct call to `destroyPageFile(name)`.  
Deletes table file permanently.

### 7. getNumTuples

Returns the current value of `hdr.total_live_rows` from rm_table_mgmt.  
Used for quick stats reporting.

### 8. insertRecord

Implements the full insertion workflow:  
- Validates pointers and enforces PK uniqueness.  
- Ensures at least one data page exists.  
- Uses `find_insertion_slot()` to get available RID.  
- Pins the destination page and writes record bytes to the slot's payload.  
- Assigns next TID, resets nullmap, sets slot flag to live, updates free_slots, and unpins.  
- If page becomes full, calls `free_list_remove_if_full()` to unlink it.  
- Increments live row count and writes updated header + schema to page 0.

### 9. deleteRecord

Pins page, retrieves slot pointer, checks if it's live, sets flag to `RM_SLOT_TOMBSTONE`, increments free_slots, and possibly links page into free list.  
If free_slots was 0, it becomes 1 and the page is added as free list head.  
Updates live row count in header and writes back to page 0.

### 10. updateRecord

Validates record pointer and RID, ensures primary key uniqueness (ignoring self), pins page, overwrites payload bytes directly, marks page dirty, and unpins.  
Does not modify TID or slot flag.

### 11. getRecord

Pins page, retrieves slot pointer, checks if record is live, copies payload bytes into user-provided record's data buffer, assigns RID, and unpins the page.

### 12. startScan

Allocates rm_scan_mgmt, stores predicate pointer, sets cursor to (page 1, slot –1), and attaches it to RM_ScanHandle.  
Prepares the structure for sequential scanning.

### 13. next

Implements sequential record iteration.  
Loops through pages starting from current cursor, pins each page, iterates through slots, and checks flag for RM_SLOT_LIVE.  
If predicate is given, calls `evalExpr()` to evaluate.  
If true, copies record payload into provided Record and updates cursor.  
Unpins page after every pass.  
Returns RC_RM_NO_MORE_TUPLES when all slots are exhausted.

### 14. closeScan

Frees rm_scan_mgmt structure allocated during startScan and resets scan handle fields to NULL.

### 15. getRecordSize

Iterates through schema attributes, adding fixed byte sizes based on type (INT/FLOAT/BOOL) and declared length for STRING.  
This size is used for computing slot layout and page headers.

### 16. createSchema

This function dynamically allocates and initializes Schema.  
We validate all input pointers, allocate Schema and its arrays (`attrNames`, `dataTypes`, `typeLength`, `keyAttrs`), then copy each name and type individually.  
All allocations are wrapped in validation; if any step fails, previously allocated memory is freed to prevent leaks.  
Diagnostic messages are printed for debugging.

### 17. freeSchema

Iterates through attrNames to free individual strings, then frees all arrays and finally the Schema pointer itself.

### 18. createRecord

Allocates Record structure and a zeroed data buffer based on record size.  
Initializes RID to (–1, –1) as a placeholder until insertion assigns a valid ID.

### 19. freeRecord

Frees both data buffer and record struct.

### 20. getAttr

Computes offset for target attribute, allocates a Value struct, sets its datatype, and copies raw bytes from record->data to value->v.*.  
For strings, allocates new memory and null-terminates the copy.

### 21. setAttr

Computes offset within record->data and writes value bytes directly based on Value type.  
For strings, ensures no overflow by truncating or padding to schema-defined length.  
Used to populate records before insertion or update.

## E. Design Characteristics

a. **Free List Management:** Implemented as a linked list between page headers to efficiently reuse space.  
b. **Schema Serialization:** Compact binary encoding on page 0 preserves schema permanently.  
c. **Primary Key Enforcement:** Done before insert or update by scanning all pages.  
d. **Buffer Management Integration:** All page modifications are done through pin/markDirty/unpin workflow to guarantee write consistency.  
e. **Defensive Memory Handling:** Every allocation and pointer dereference is validated.  
f. **Modularity:** Helper functions isolate offset and slot calculations, keeping core APIs concise and error-free.

## F. Testing Instructions

To run the test cases, follow these steps in terminal:

```bash
make clean
make test
./test_assign3_1
```

To check if all test cases passed:

```bash
if ./test_assign3_1 | grep -q "FAILED"; then 
  echo "Some tests failed";
else
  echo "All tests passed";
fi
```

## G. Test Results

All test cases in `test_assign3_1.c` passed successfully, verifying:
<img width="1459" height="359" alt="image" src="https://github.com/user-attachments/assets/1989c30a-06a2-4d10-9da9-87b2bc6ca7c4" />
<img width="1055" height="200" alt="image" src="https://github.com/user-attachments/assets/623974de-22e0-4f5d-8969-2a786caf784e" />
<img width="1008" height="162" alt="image" src="https://github.com/user-attachments/assets/3c2503bf-f069-4a70-9aa7-932f845683c1" />



**Submitted by:**  
**Meghana Rabba** (A20572009)  
**Rohan Singh Rajendra Singh** (A20572007)  
Illinois Institute of Technology  
CS525 – Advanced Database Organization  
Fall 2025

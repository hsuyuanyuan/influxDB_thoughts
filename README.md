
###**Issue #1:**	The index contains the full-length keys.  For each key, the index requires the following bytes: 2 (KeyLen)  + N (Key)  + 1 (Type) + 2 (Count) + NumEntries * 28 (EntryLen)

**Thoughts:** 

1.  Assign each Key with a 6-byte ID, including 4-byte Measurement ID + 2-byte Series ID. With the 2-byte Series ID, we support 64K series in each measurement, which should be enough to accommodate the normal usage of tag sets and fields;
2.  When a measurement is deleted, we do not reuse that Measurement ID. Therefore, any newly assigned Measurement ID does not conflict with previously assigned ones. This would simplify many operations and avoid meta data corruption. With 4 billion slots for Measurement IDs, this should not impact the normal usage;
3.  The Key-ID mapping table is stored in the Meta node cluster to provide fault tolerance for this critical information. When a Data node restarts or joins, it makes a local copy of the mapping table. Updates to the mapping table is handled by the Meta node cluster and synced to the Data nodes;

**Benefits:** 

1. Reduce the space usage for each key and save on both disk and memory, which would alleviate the OOM issue at startup;
2. Remove the expensive string comparison with the full-length keys in every query;
3. If a user wants to modify the measurement names or tag values or field names, we need not to go through all TSM files to update the indices. We just need to update the mapping table;
4. Allow us to enrich the measurements and fields with other information, such as description, data type and valid data range etc.



###**Issue #2.**  At startup, the engine scans all TSM files and loads all the index regions into memory

**Thoughts:**

1. In the TSM file, add an index summary region directly before the existing index region. The entry in the index summary contains the Key ID, min / max time and offset to the corresponding entry in the index region. This is similar to the current indirect index built at startup.  Now with the fixed-length Key-ID, each entry in the index summary region has the same size.
2. In the footer, add the offset to the index summary region. Maybe we should also add the file-wise min / max time into the footer to speed up the filtering and lookup.
3. At startup, instead of loading the index region, just scan and load the index summary region, which should be orders of magnitude smaller than the index region. Individual index can be loaded later on demand;

**Benefits:**

1. Remove the need to construct the indirect index at startup, since the index summary region is essentially the indirect index;
2. Less disk IOs. This should speed up the startup process;
3. Less memory usage. This should help on the OOM issue at startup;


###**Issues #3.** Data for a measurement can be in any TSM file

**Thoughts:**

1. Data locality can be improved if we group measurements into collections. Then we need not to check every TSM files when reading a time series;
2. Use the highest byte in the 4-byte Measurement ID as the Collection ID;
3. The Collection ID is prefixed in the TSM file names to identify the collection they belong to. Data for a specific collection is only written to files in that collection;

**Benefits:**

1. Improved locality means reduced contention and better performance; 
2. Good preparation for multi-tenant solution in the future. Collections can have different retention policies and security settings;
	


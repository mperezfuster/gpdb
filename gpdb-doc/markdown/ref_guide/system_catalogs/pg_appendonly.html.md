# pg_appendonly 

The `pg_appendonly` table contains information about the storage options and other characteristics of append-optimized tables.

|column|type|references|description|
|------|----|----------|-----------|
|`relid`|oid| |The table object identifier \(OID\) of the compressed table.|
|`majorversion`|smallint| |The major version number of the pg\_appendonly table.|
|`minorversion`|smallint| |The minor version number of the pg\_appendonly table.|
|`segidxid`|oid| |Index on-disk segment file id.|
|`visimaprelid`|oid| |Visibility map for the table.|
|`visimapidxid`|oid| |B-tree index on the visibility map.|

> **Note** <sup>1</sup>QuickLZ compression is available only in the commercial release of VMware Greenplum.

**Parent topic:** [System Catalogs Definitions](../system_catalogs/catalog_ref-html.html)


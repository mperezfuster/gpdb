# System Catalog Functions

Greenplum Database provides the following system catalog functions:

> **Note** This list is provisional and may be incomplete.

-   [pg_stat_get_backend_subxact](#pg_stat_get_backend_subxact)

## <a id="statistics"></a>Statistics Functions

The following functions support collection and reporting of information about server activity.

### <a id="pg_stat_get_backend_subxact"></a>pg_stat_get_backend_subxacts

`pg_stat_get_backend_subxact(integer)` return type is `record`.

Returns a record of information about the subtransactions of the backend with the specified ID. The fields returned are `subxact_count`, which is the number of subtransactions in the backend's subtransaction cache, and `subxact_overflow`, which indicates whether the backend's subtransaction cache is overflowed or not.

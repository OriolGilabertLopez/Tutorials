# Delete allocated sapace in SQL Server 

The data space allocated is The amount of formatted files space made available for storing database data. 

This space normaly grows automatically, but never decreases after deletes (tables, precedres, etc.). This behavior ensures that future inserts are faster since space does not need to be reformatted. As shown in the [image](https://docs.microsoft.com/en-us/azure/azure-sql/database/file-space-manage) below


![allocatedSpace](https://docs.microsoft.com/en-us/azure/azure-sql/database/media/file-space-manage/storage-types.png)

To delete the data space allocated we can use teh following code:

```sql
DBCC SHRINKDATABASE ('<DataBaseName>', <target_percent>);
```

## Arguments

 * __`<DataBaseName>`__: Is the database name. 0 specifies that the current database is used.

 * __`<target_percent>`__: Is the percentage of free space that you want left in the database file after the database has been shrunk.
 
 ## NOTES:
 
 1. DBCC SHRINKDATABASE operations can be stopped at any point in the process, and any completed work is kept.
 2. If a database is originally created with a size of 10 MB in size. Then, it grows to 100 MB. The smallest the database can be reduced to is 10 MB, even if all the data in the database has been deleted
 3. A shrink operation doesn't preserve the fragmentation state of indexes in the database, and generally increases fragmentation to a degree. This result is another reason not to repeatedly shrink the database.

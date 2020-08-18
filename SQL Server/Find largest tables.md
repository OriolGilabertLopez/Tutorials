
# Fins Largest tables in a SQLS DB

To find the LARGEST TABLES we will use the following objects:

* __`sys.indexes`__: Contains a row per index or heap of a tabular object, such as a table, view, or table-valued function
  * `name`: Name of the index. name is unique only within the object. NULL = Heap
  * `index_id`: ID of the index. index_id is unique only within the object.
      * `0` = Heap
      * `1` = Clustered index
      * `> 1` = Nonclustered index

  * `object_id`:	ID of the object to which this index belongs

* __`sys.partitions`__: Contains a row for each partition of all the tables and most types of indexes in the database. Special index types such as Full-Text, Spatial, and XML are not included in this view. All tables and indexes in SQL Server contain at least one partition, whether or not they are explicitly partitioned
  * `object_id`:Indicates the ID of the object to which this partition belongs. Every table or view is composed of at least one partition
  * `index_id`: Indicates the ID of the index within the object to which this partition belongs
      * `0` = heap
      * `1` = clustered index
      * `2` or greater = nonclustered index
   * `rows`: Indicates the approximate number of rows in this partition

* __`sys.allocation_units`__: Contains a row for each allocation unit in the database
  * `container_id`: If type is 2, then __container_id = sys.partitions.partition_id__
  * `total_pages`: Total number of pages allocated or reserved by this allocation unit
  * `used_pages`: Number of total pages actually in use.
  * `data_pages`: Number of used pages that have: In-row data, LOB data and Row-overflow data

* __`sys.tables`__: Returns a row for each user table in SQL Server.



So, the construction of the code is as follows.


```sql
SELECT 
	t.NAME AS TableName, 
	i.name AS indexName, 
	SUM(p.rows) AS RowCounts, 
	SUM(a.total_pages) AS TotalPages, 
	SUM(a.used_pages) AS UsedPages,
	SUM(a.data_pages) AS DataPages,
	(SUM(a.total_pages) * 8) / 1024 AS TotalSpaceMB, 
	(SUM(a.used_pages) * 8) / 1024 AS UsedSpaceMB, 
	(SUM(a.data_pages) * 8) / 1024 AS DataSpaceMB
FROM 
	sys.tables as t 
INNER JOIN 
	sys.indexes as i
		ON t.OBJECT_ID = i.object_id 
INNER JOIN 
	sys.partitions as  p
		ON  i.object_id = p.OBJECT_ID 
		AND i.index_id = p.index_id 
INNER JOIN 
	sys.allocation_units as a 
		ON p.partition_id = a.container_id 
WHERE  
	t.NAME NOT LIKE 'dt%' 
	AND i.OBJECT_ID > 255 
	AND i.index_id <= 1 
	AND t.NAME = '<Table_Name>'
GROUP BY 
	t.NAME, 
	i.object_id, 
	i.index_id, 
	i.name
```

__NOTE:__ The __<Table_Name>__ is your table without having to use the schema prefix 


Soure: [Maybe unknown](https://social.msdn.microsoft.com/Forums/sqlserver/en-US/c6861dc1-b5fd-4749-b9e1-a180b3b9734c/size-of-sql-server-2008-r2-database-question?forum=sqlkjmanageability)

# REBUILD INDEX Procedure

This simple command rebuilds indexes of a specific table:

```sql
ALTER INDEX ALL ON Table_Name REBUILD;
```

## PROCEDURE `sp_RebildIndex`

But we can create a consistent procedure to rebuil indexes based on a threhold. 

### Arguments:
  1.  __`pct_frag`__: is the minimum threshold to select tables
 
### Code:
```sql
CREATE PROC  [dbo].[sp_RebildIndex] @pct_frag INT
AS 
	 

-- We check if some table exists. we will use this tables throughout the script and, if we use this procedure recursively, 
-- for example, in Azure Data Factory or other trigger scripts, we need to drop this tables. 

DROP TABLE IF EXISTS Tables_To_Rebild_Index_pct;
DROP TABLE IF EXISTS Table_RealTime_RebuildIndex;
DROP TABLE IF EXISTS Tables_IN;
	
	


-- We create a SQL table 
CREATE TABLE Tables_IN (TabsIN NVARCHAR(MAX)  NULL);
INSERT Tables_IN (TabsIN)  VALUES('<TableName_1>'),
                                 ('<TableName_2>') 
                                 ...
                                 ('<TableName_n>');

-- We create a temporaries SQL tables to store tableS with indexs (if exists)

SELECT * INTO  #Tables_With_Index
FROM(
	select  distinct OBJECT_NAME(IDX.OBJECT_ID) AS Table_Name, 
			IDX.name AS Index_Name, 
			IDXPS.index_type_desc AS Index_Type, 
			IDXPS.avg_fragmentation_in_percent AS Fragmentation_Percentage
	from 
		sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, NULL) IDXPS 
	inner join sys.indexes IDX  
		on (IDX.object_id = IDXPS.object_id 
			AND IDX.index_id = IDXPS.index_id) 
	where  
		OBJECT_NAME(IDX.OBJECT_ID) in (SELECT * FROM Tables_IN)
		and IDXPS.index_type_desc = 'NONCLUSTERED INDEX' -- filtering table with index!		
) AS select_1;
		
		
    
-- We filter according to @pct_frag threshold parameter and save the result in a table whose name is Tables_To_Rebild_Index_pct

SELECT * INTO  Tables_To_Rebild_Index_pct 
FROM (
		select  Table_name, 
				ROW_NUMBER() OVER(ORDER BY Table_name ASC) AS IdRow
		from ( 
				select  Table_name, max(Fragmentation_Percentage) as MAX_PCT
				from  #Tables_With_Index
				group by Table_name

	) AS select_2
	where MAX_PCT > @pct_frag -- minimal pct fragmentation

) AS select_3



-- we iterate through  the table rows ([dbo].[tables_to_rebild_Index_pct]) 
-- and execute rebuild index for each table along the WHILE statement 

-- VARIABLES

  -- 'COUNTER'. We will use this variable to extract the name of the table as we go through the loop
  DECLARE @cnt INT;

  -- Max valie of @cnt, When  @cnt = @NumTablesToRebuildIndex , then we must to stop the loop
  DECLARE @NumTablesToRebuildIndex INT;		

  -- Store the current table name
  DECLARE @CurrentTable  varchar(255);		

  -- Store the current table row
  DECLARE @CurrentExecution  varchar(500);	

  -- Timsetamp wehen we exectue the rebuid table index
  DECLARE @Date datetime;						

  -- duration of the execution
  DECLARE @ElapsedTime nvarchar(12);		



SET @cnt = 1		-- We set/initialize @cnt in 1
SET @NumTablesToRebuildIndex = (SELECT COUNT(*) FROM tables_to_rebild_Index_pct);


-- Information table in RT. Status table index.
CREATE TABLE Table_RealTime_RebuildIndex(
	[Tables] nvarchar(500),
	[Process] nvarchar(500),
	[ExecutionTime] nvarchar(12)
);	
		
 
-- LOOP
WHILE @cnt <= @NumTablesToRebuildIndex
BEGIN
-- Catching errors
	BEGIN TRY
					
		-- Table Name
		SET @CurrentTable = ( SELECT Table_name 
								FROM tables_to_rebild_Index_pct 
								WHERE IDROW = @cnt);
		SET @CurrentExecution =  (CONCAT('The table indexes are being rebuilt: ', @CurrentTable))
		SET @Date = GETDATE()
				
		-- Comments
		INSERT Table_RealTime_RebuildIndex ([Tables] , [Process]) 
		VALUES(@CurrentExecution, CONCAT(@cnt, ' of ', @NumTablesToRebuildIndex)); 


		-- REBUID INDEX COMMAND
		EXEC('ALTER INDEX ALL ON ' +  @CurrentTable + ' REBUILD')   -- 'EXEC' execute the str expression


		-- Execution Time
    		SET @ElapsedTime = SUBSTRING(CONVERT(VARCHAR, DATEADD(mcs, 
                                                                      DATEDIFF(mcs, @Date, GETDATE()),
                                                                      CAST('1900-01-01 00:00:00.000' AS DATETIME2))
                                           ), 11, 12)		   	
	-- update the column table values 
		UPDATE Table_RealTime_RebuildIndex
		SET [Tables]  = @CurrentTable,
		    [Process] = 'SUCCESS: ' ,
		    [ExecutionTime] = @ElapsedTime 
		WHERE Process = CONCAT(@cnt, ' de ', @NumTablesToRebuildIndex);

		SET @cnt = @cnt + 1;
					  

	END TRY
	BEGIN CATCH

		SET @ElapsedTime = SUBSTRING(CONVERT(VARCHAR, DATEADD(mcs, 
                                                                      DATEDIFF(mcs, @Date, GETDATE()),
                                                                      CAST('1900-01-01 00:00:00.000' AS DATETIME2))
                                           ), 11, 12)
                                   
		UPDATE Table_RealTime_RebuildIndex
		SET [Tables]  = @CurrentTable,
			[Process] =  (  SELECT CONCAT(
                                    'ERROR: ', ERROR_MESSAGE(),
                                    ' || ERROR NUMBER: ', ERROR_NUMBER(),
                                    ' || ERROR STATE: ', ERROR_STATE(),
                                    ' || ERROR SEVERITY: ', ERROR_SEVERITY())),
			[ExecutionTime] = @ElapsedTime
		WHERE Process = CONCAT(@cnt, ' de ', @NumTablesToRebuildIndex);


		SET @cnt = @cnt + 1;
				
	END CATCH;  --catching errors

		
	
END; -- End While;
```

# References
  1. Microsoft SQL Docs. [ALTER INDEX (Transact-SQL)](https://docs.microsoft.com/en-us/sql/t-sql/statements/alter-index-transact-sql?view=sql-server-ver15), 08/21/2019 .

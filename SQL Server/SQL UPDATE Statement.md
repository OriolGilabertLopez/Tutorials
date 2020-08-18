# SQL UPDATE Statement

In this section whe learn how to modify the existing records in a table.

## Simple UPDATE
```sql
UPDATE TablaToUpdate 
SET Column_1 = <value_1>,  
    Column_n = <value_n>, 
WHERE ColConditon = <recordsToUpdate>;
```
The __WHERE__ clause specifies which record(s) that should be updated. If you omit the WHERE clause, __all__ records in the table will be updated!

## Conditional UPDATE: Using values from other tables (LEFTs, INNERs, etc)
```sql
UPDATE Table_1 
SET Table_1.Col1 = T2.Col2, 
	Table_1.Col2 = T2.Col2 
FROM Table_1 AS T1 
LEFT JOIN Table_2 AS T2 
ON ( T1.PK1= T2.PK1 
 AND T1.PK2= T2.PK2)
```
__Note__:  Be careful when use __Table_1__ or __T1__, notice the SET clause use __Table_1__ instead of
 __T1__, so SQL Server does not accept in this structure the abbreviations. __Table_2__ does not follow the same rules.    

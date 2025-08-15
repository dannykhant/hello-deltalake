# Delta Lake 4.0

- Why?
    - Traditional data lakes use Parquet/CSV and face limitations
    - No ACID, Hard to handle schema changes, Not easy to track changes and rollback to earlier versions
- Delta Lake Features -
    - ACID transactions
    - Schema enforcement & evolution
    - Time travel & versioning
    - Unified batch & streaming
    - Data lineage & audit
- Architecture
    - Delta Format = Delta Log + Parquet Files
    - Delta Log takes care of all transactions
- Databricks
    - Uses S3 for the managed tables
- Delta DDL
    - SQL API
        
        ```sql
        create table my_catalog.my_schema.my_table
        (
        	id int not null unique,
        	name string
        )
        using delta;
        ```
        
    - Delta Python API
        
        ```python
        from delta.tables import DeltaTable, IdentityGenerator
        
        DeltaTable.createIfNotExists(spark)\
        	.tableName("my_catalog.my_schema.delta_two")\
        	.addColumn("id", "int")\
        	.addColumn("name", "string")\
        	.execute()
        ```
        
- Auto-generated Columns (Not available in SQL API as of now 4.0)
    - Identity Column
        
        ```python
        DeltaTable.createIfNotExists(spark)\
        	.tableName("my_catalog.my_schema.delta_two")\
        	.addColumn("id", datatype=IntegerType(), generatedAlwaysAs=IdentityGenerator())\
        	.addColumn("name", "string")\
        	.execute()
        ```
        
    - Computed Column
        
        ```python
        DeltaTable.createIfNotExists(spark)\
        	.tableName("my_catalog.my_schema.delta_two")\
        	.addColumn("name", "string")\
        	.addColumn("salary", "double")\
        	.addColumn("salaryAfterTax", "double", generatedAlwaysAs="salary*0.90")\
        	.execute()
        ```
        
- Delta Log
    - Delta Log writes the transaction actions in the JSON files
    - Then do exactly as written in there when reading the data, this gives Atomicity
- Schema Enforcement & Evolution
    - Delta always enforces the schema not to be mismatched
    - But when requires schema evolution, just add an option(”mergeSchema”, True) when writing
- Schema Overwrite
    - Delta always enforces the schema not to be mismatched even when overwriting because of time traveling feature
    - Add option(”overwriteSchema”, True) if requires to overwrite the schema
- DML
    
    ```sql
    update delta.`/Volumes/filepath/delta_one`
    set name = "new"
    where id = 5;
    ```
    
    - When updating a record (id = 5)
        - Let’s say - Part 1 has id 1, 2, 3 and Part 2 has id 4, 5, 6
        - After the update, Part 3 is created
        - In the transaction log, Delta will write -
            - Add P1 [1, 2, 3]
            - Add P2 [4, 5, 6]
            - Remove P2
            - Add P3 [4, 5, 6]
    - With Deletion Vector
        - In Part 3, Delta will write only the updated records.
        - In the transaction log, Delta will write -
            - Add P1 [1, 2, 3]
            - Add P2 [4, 5, 6]
            - Remove P2
            - Add P3 [5]
            - Add P2 with Deletion Vector [4, 6]
        - Finally Deletion Vector does the optimization by doing Coalesce
            - P1 + P2 + P3 ⇒ One New Partition
- Deletion Vector
    - Storage cost optimization technique
- Delta UPSERT
    
    ```python
    dlt_obj = DeltaTable.forPath(spark, "/Volumes/filepath/delta_one")
    
    dlt_obj.alias("dst").merge(df.alias("src"), "dst.id=src.id")\
    				.whenMatchedUpdateAll()\
    				.whenNotMatchedInsertAll()\
    				.execute()
    ```
    
- Delta Schema Changes
    
    ```sql
    alter table delta.`/Volumes/filepath/delta_one`
    rename column name to customer_name;
    ```
    
    - When updating the schema of delta table
    - The schema in the Parquet will not be changed
    - Only the transaction log in the Delta Log will be updated
- Column Mapping Mode
    
    ```sql
    alter table delta.`/Volumes/filepath/delta_one`
    set tableproperties
    (
    	'delta.minReaderVersion' = '2',
    	'delta.minWriterVersion' = '5',
    	'delta.columMapping.mode' = 'name'
    );
    ```
    
    - Column Mapping “name” mode needs to be enabled for the schema change.
- Table Utility Command
    - For checking table
        
        ```sql
        describe delta.`/Volumes/filepath/delta_one`;
        ```
        
    - For details
        
        ```sql
        describe detail delta.`/Volumes/filepath/delta_one`;
        ```
        
    - For extended details
        
        ```sql
        describe extended delta.`/Volumes/filepath/delta_one`;
        ```
        
    - Data Versioning & Time Traveling (Allows to go back 30 days as default)
        
        ```sql
        describe history delta.`/Volumes/filepath/delta_one`;
        
        restore delta.`/Volumes/filepath/delta_one` to version as of 3;
        restore delta.`/Volumes/filepath/delta_one` to timestamp as of '2025-07-05T00:00:00';
        ```
        
- Table Properties
    - The properties like enabling Deletion Vector, Append Only etc..
    - To view all properties
        
        ```sql
        show tblproperties delta.`/Volumes/filepath/delta_one`;
        ```
        
- Vacuum Command
    - Used to remove any unused parquet files (Tombstones) older than 7 days
        
        ```sql
        vacuum delta.`/Volumes/filepath/delta_one` retain 0 hours;
        ```
        
- Cloning Command
    - Deep Cloning
        
        ```sql
        create table my_catalog.my_schema.my_tbl
        clone delta.`/Volumes/filepath/delta_one` version as of 3;
        ```
        
    - Shallow Cloning (Managed Table Only)
        - Only clone the Delta Log
        
        ```sql
        create table my_catalog.my_schema.my_tbl
        shallow clone my_catalog.my_schema.other_tbl;
        ```
        
- Change Data Feed
    - Works like CDC in SQL databases
        
        ```sql
        alter table my_catalog.my_schema.my_tbl
        set tblproperties ('delta.enableChangeDataFeed' = 'true')
        ```
        
    - Extract the table changes
        
        ```sql
         -- table, version start, end
        select * from table_changes('my_catalog.my_schema.my_tbl', 1)
        ```
        
- Uniform (Universal Format)
    - Allows to read delta table with Iceberg or Hudi clients
        
        ```sql
        create table my_catalog.my_schema.my_tbl
        tblproperties (
        	'delta.enableIcebergCompatV2' = 'true',
        	'delta.universalFormat.enabledFormats' = 'iceberg'
        );
        ```
        
- Delta Lake Optimization
    - To fix too many smaller files problem (Reading lesser & bigger files is better)
    - This will optimize the performance of delta table
    - Optimize command does the coalesce of the many smaller parquet files into lesser bigger files
    - Z-ordering will sort in all partitions and then check the column stats (min, max) to go to the specific partition when reading data - The Data Skipping
        
        ```sql
        optimize delta.`/Volumes/filepath/delta_one`;
        
        optimize delta.`/Volumes/filepath/delta_one` zorder by (id);
        ```
        
- Liquid Clustering
    - Partitioning is no longer required because of this technology
    - It allows to **define clustering keys**, and then **incrementally and intelligently organizes data** around those keys without needing to rewrite the entire table each time
        
        ```sql
        alter table my_catalog.my_schema.my_tbl
        cluster by (id);
        
        alter table my_catalog.my_schema.my_tbl
        cluster by auto;
        ```
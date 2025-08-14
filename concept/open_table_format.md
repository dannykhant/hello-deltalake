# Open Table Format

- Open Table Format
    - Used to overcome the utilization issue of direct data lake usage
    - It is a (structured) metadata layer on top of data lake
    - It gives the features of a SQL database such as schema enforcement
    - It also gives the optimization and compression just to save storage cost
- Popular Open Table Formats
    - Apache Iceberg
    - Delta Lake
    - Apache Hudi
- Delta Lake
    - Delta Format Creation from dataframe
        
        ```python
        df.write.format("delta") \
        		.option("path", "/filepath/sink_data/my_data")
        		.save()
        		
        # to query it
        spark.sql(select * from delta.`/filepath/sink_data/my_data`).display()
        ```
        
    - Delta Format
        - Data is stored in Parquet format
        - It has Delta Log (Transaction Log) that will store the information of the transactions such as schema, filename, etc..
        - Delta Log is in JSON format
        - Delta Log creates a Parquet checkpoint file for every last 10 transactions to have log compaction
        - To view a JSON file on Databricks notebook
            
            ```sql
            %fs head /filepath/data/_delta_log/00000000000000.json
            ```
            
    - Delta Table Creation DDL
        
        ```sql
        create table schema.delta_table
        (
        	id int,
        	name string,
        	salary double
        )
        using delta
        location '/filepath/sink_data/my_delta_table'
        ```
        
    - DML
        - Delta does only soft delete on the parquet data
        - DML will create a new partition (P5) with the updated data of the old partition (P3) (Tombstoning)
        - DML will write the remove info (P3) in the json file of the DML transaction
        - DQL will read the info in json file and give you the data by extracting from the latest partition (P5)
    - Data Versioning
        - Every transaction will be stored as a version
            
            ```sql
            describe history schema.my_delta_table
            ```
            
    - Time Traveling
        - This can help to go back to any version
            
            ```sql
            select * from schema.my_delta_table version as of 3
            ```
            
    - Vacuum
        - This will remove all the unused parquet files that are older than 7 days as default
            
            ```sql
            vacuum schema.my_delta_table;
            ```
            
        - Dry Run provides the list of files that will be deleted
            
            ```sql
            vacuum schema.my_delta_table dry run;
            ```
            
        - Retain 0 hours
            
            ```sql
            set spark.databricks.delta.retentionDurationCheck.enabled = false;
            
            vacuum schema.my_delta_table retain 0 hours dry run;
            ```
            
    - Schema Enforcement & Evolution
        - Delta force data to comply the schema defined
        - The option “mergeSchema” can be used for Schema Evolution
    - Metadata Level Changes
        - Adding column, reordering column will be only saved in the Delta Log
            
            ```sql
            -- adding new column
            alter table schema.my_delta_table
            add column new_flag boolean;
            
            -- reordering columns
            alter table schema.my_delta_table
            alter column id after name;
            ```
            
        - Column Mapping
            - Some of the metadata changes require the column mapping
                
                ```sql
                alter table schema.my_delta_table
                drop column new_flag;
                ```
                
            - To Map the column, it requires to update some table properties
                
                ```sql
                alter table schema.my_delta_table
                set tableproperties
                (
                	'delta.minReaderVersion' = '2',
                	'delta.minWriterVersion' = '5',
                	'delta.columMapping.mode' = 'name'
                );
                
                -- rename the column
                alter table schema.my_delta_table
                rename column id to cust_id;
                ```
                
- Optimization Techniques
    - Reading less bigger files than many smaller files - is better
    - Optimize command will coalesce the partitions
    - Z-ordering can boost the process by sorting the data
    - Data Skipping will help the data reading faster by reading the statistics columns to go to the specific partition and read
    - Statistics columns are the first 32 columns of the table
        
        ```sql
        optimize schema.my_delta_table
        zorder by (id);
        ```
        
    - Optimize Write
        - This will make the executors to coalesce the data before writing
            
            ```python
            df.write.format("delta") \
            		.option("path", "/filepath/sink_data/my_data")\
            		.option("optimizeWrite", True)\
            		.save()
            ```
            
- Overwrite
    - Overwrite Mode needs to overwrite the schema as well if there is schema mismatch between the existing data and new one because of the schema enforcement
        
        ```python
        df.write.format("delta") \
        		.mode("overwrite")\
        		.option("path", "/filepath/sink_data/my_data")\
        		.option("optimizeWrite", True)\
        		.option("overwriteSchema", True)\
        		.save()
        ```
        
    - Overwriting doesn’t actually remove the old data because of the Time Traveling feature
        
        ```python
        select * from delta.`/filepath/sink_data/my_data` version as of 0;
        ```
        
- Deletion Vector
    - Deletion Vector save the delete information of the old partition in the deletion vector
    - It allows just to create a new partition with only the updated data.
        - The query - update table set name = ‘new name’ where id = 3
        - Partition1 has id 1 and 2
        - Partition2 has id 3 and 4
        - After the update, it will be -
            - Partition3 will be created with updated data and has only id 3
            - Final result = p1 + p3 + (p2 with deletion vector)
            - It will run the **Optimize** command finally to coalesce the data into a single partition
- Spark Structured Streaming in open table format
    - Streaming Query - it will exactly copy the data with Idempotency ability
    
    ```python
    df = spark.readStream.table("schema.new_table")
    df.writeStream.format("delta")\
    					.option("checkpointLocation", "/filepath/stream_table/checkpoint")\
    					.trigger(processingTime = "10 seconds")\
    					.toTable("schema.stream_table")
    ```
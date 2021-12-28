# S3 Redshift Batch ETL Pipeline

## Statement
The goal of this project is to load data from S3, process the data into analytics tables using Spark, and load them back into S3 as a set of dimensional tables in order to allow the analytics team to draw insights in what songs users are listening to. Also automate the loading of output S3 dimensional tables to Redshift  using Airflow DAGs.

## Introduction
A music streaming startup, Global Music, has grown their user base and song database even more and want to move their data warehouse to a data lake. Their data resides in S3, in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app. <br>

As their data engineer, you are tasked with building an ETL pipeline that extracts their data from S3, processes them using Spark, and loads the data back into S3 as a set of dimensional tables. This will allow their analytics team to continue finding insights in what songs their users are listening to. <br>

You'll be able to test your database and ETL pipeline by running queries given to you by the analytics team from Global Music and compare your results with their expected results. <br>

the company has decided that it is time to introduce more automation and monitoring to their data warehouse ETL pipelines and come to the conclusion that the best tool to achieve this is Apache Airflow. <br>

They have decided to bring you into the project and expect you to create high grade data pipelines that are dynamic and built from reusable tasks, can be monitored, and allow easy backfills. They have also noted that the data quality plays a big part when analyses are executed on top the data warehouse and want to run tests against their datasets after the ETL steps have been executed to catch any discrepancies in the datasets. <br>

The source data resides in S3 and needs to be processed in Global Music's data warehouse in Amazon Redshift. The source datasets consist of CSV logs that tell about user activity in the application and JSON metadata about the songs the users listen to. 

## Project Datasets
The data sources to ingest into data lake are provided by two public S3 buckets:

1. Songs bucket (s3://udacity-dend/song_data), contains info about songs and artists. 
All files are in the same directory.
2. Event bucket (s3://udacity-dend/log_data), contains info about actions done by users, what song are listening, ... 

<b>Log Dataset structure:</b>
![Log Dataset](./images/log_dataset.png)

<b>Song dataset structure:</b>
~~~~
{"num_songs": 1, "artist_id": "ARJIE2Y1187B994AB7", "artist_latitude": null, "artist_longitude": null
, "artist_location": "", "artist_name": "Line Renaud", "song_id": "SOUPIRU12A6D4FA1E1", 
"title": "Der Kleine Dompfaff", "duration": 152.92036, "year": 0}
~~~~

By the way we need two staging tables:

- Stage_events
- Stage_songs

Prerequisite <br>
Tables must be created in Redshift before executing the DAG workflow. The create tables script can be found in:

create_tables.sql

## ETL Pipeline
1.  Read data from S3
    
    -   Song data:  `s3://udacity-dend/song_data`
    -   Log data:  `s3://udacity-dend/log_data`
    
    The script reads song_data and load_data from S3.
    
3.  Process data using spark
    
    Transforms them to create five different tables listed below : 
    #### Fact Table
	 **songplays**  - records in log data associated with song plays i.e. records with page  `NextSong`
    -   _songplay_id, start_time, user_id, level, song_id, artist_id, session_id, location, user_agent_

	#### Dimension Tables
	 **users**  - users in the app
		Fields -   _user_id, first_name, last_name, gender, level_
		
	 **songs**  - songs in music database
    Fields - _song_id, title, artist_id, year, duration_
    
	**artists**  - artists in music database
    Fields -   _artist_id, name, location, lattitude, longitude_
    
	  **time**  - timestamps of records in  **songplays**  broken down into specific units
    Fields -   _start_time, hour, day, week, month, year, weekday_
    
4.  Load it back to S3
    
    Writes them to partitioned parquet files in table directories on S3.

5. Load the output tables from S3 to Redshift and automate the whole process using Airflow.

## Project Structure

The project is made up of an `etl.py` script for initializing the spark cluster, loading the data from S3, processing data into dimensional model and loading back to S3. The `dl.cfg` is stores the configuration parameters for reading and writing to S3. 

Using the song and log datasets, a star schema optimized for queries on song play analysis derived. The Fact table for this schema model was made of a single table - `songplays` - which records in log data associated with song plays .i.e. records with page `NextSong`. The query below applies a condition that filters actions for song plays:

`df = df.where(df.page == 'NextSong')`

The query below extracts columns from both song and log dataset to create `songplays` table

`songplays_table = spark.sql("""select row_number() over (order by log_table.start_time) as songplay_id, \
                                                        log_table.start_time, year(log_table.start_time) year, \
                                                        month(log_table.start_time) month, log_table.userId as user_id, \
                                                        log_table.level, song_table.song_id, song_table.artist_id, \
                                                        log_table.sessionId as session_id, log_table.location, \
                                                        log_table.userAgent as user_agent \
                                                        from log_table \
                                                        join song_table on (log_table.artist = song_table.artist_name and \
                                                        log_table.song = song_table.title and log_table.length = song_table.duration )""")`


* /
    * `create_tables.sql` - Contains the DDL for all tables used in this projecs
* dags
    * `udac_example_dag.py` - The DAG configuration file to run in Airflow
* plugins
    * operators
        * `stage_redshift.py` - Operator to read files from S3 and load into Redshift staging tables
        * `load_fact.py` - Operator to load the fact table in Redshift
        * `load_dimension.py` - Operator to read from staging tables and load the dimension tables in Redshift
        * `data_quality.py` - Operator for data quality checking
    * helpers
        * `sql_queries` - Redshift statements used in the DAG

## Data Quality Checks

In order to ensure the tables were loaded, 
a data quality checking is performed to count the total records each table has. 
If a table has no rows then the workflow will fail and throw an error message.

## Deployment
File `dl.cfg` is not provided here. File contains :


```
KEY=YOUR_AWS_ACCESS_KEY
SECRET=YOUR_AWS_SECRET_KEY
```

If you are using local as your development environemnt - Moving project directory from local to EMR 


 

     scp -i <.pem-file> <Local-Path> <username>@<EMR-MasterNode-Endpoint>:~<EMR-path>

Running spark job (Before running job make sure EMR Role have access to s3)

    spark-submit etl.py --master yarn --deploy-mode client --driver-memory 4g --num-executors 2 --executor-memory 2g --executor-core 2

## Results
Fig.1: `output_data` showing `Artists/`, `Songs/`, `Time/` and `Users/` path
![project-datalake-img1](https://user-images.githubusercontent.com/76578061/132271586-575b1511-c80b-4696-a9a4-770c91c44bf7.png)

Fig.2: Sample `artist_data.parquet/` data loaded back into S3
![project-datalake-img2](https://user-images.githubusercontent.com/76578061/132271624-dcebd7ac-ae54-4bb4-b5f1-c327b2f87adb.png)

Fig.3: Sample `songs_table.parquet/` data loaded back into S3
![project-datalake-img3](https://user-images.githubusercontent.com/76578061/132271661-78894259-d4c9-4854-a972-bd7e258a5061.png)

Fig.4: Sample `time_data.parquet/` partitioned by `year` and `month` loaded into S3
![project-datalake-img4](https://user-images.githubusercontent.com/76578061/132271703-bf77fde2-80b9-464c-a162-b6dc7882f0a5.png)

Fig.5: Sample `users_data.parquet/` data loaded back into S3
![project-datalake-img5](https://user-images.githubusercontent.com/76578061/132271732-6e8bb465-b161-425b-8178-fc6901e1f34e.png)

The DAG is implemented to load data from S3 into staging tables in Redshift, then dimensional tables and a fact table are created in Redshift out of the staging tables:

![DAG](/images/DAG.png)

The data resides in S3 objects as JSON files, and so the SQL scripts use the COPY command in Redshift to import all data. 

The below illustrates runs of this DAG (although during development):

![Runs](/images/Runs.png)

## Technologies Used
* Python
* configparser
* pyspark
* datetime
* S3 Buckets
* Redshift
* Airflow

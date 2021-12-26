# S3 Data Lake with Spark

## Statement
The goal of this project is to load data from S3, process the data into analytics tables using Spark, and load them back into S3 as a set of dimensional tables in order to allow the analytics team to draw insights in what songs users are listening to.

## Introduction
A music streaming startup, Global Music, has grown their user base and song database even more and want to move their data warehouse to a data lake. Their data resides in S3, in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app. <br>

As their data engineer, you are tasked with building an ETL pipeline that extracts their data from S3, processes them using Spark, and loads the data back into S3 as a set of dimensional tables. This will allow their analytics team to continue finding insights in what songs their users are listening to. <br>

You'll be able to test your database and ETL pipeline by running queries given to you by the analytics team from Global Music and compare your results with their expected results. 

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

## Technology and Libraries
* Jupyter Notebook
* configparser
* pyspark
* datetime

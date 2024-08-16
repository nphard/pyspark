# bigdata pipeline 
this repo describes the bigdata processing pipeline best practices. 
the ETL job normally deems as a dirty job, and many jobs have various upstreaming data sources (maybe hundrends of them ) from prouction systems, and each task is unique or customized scripts, if not managed well, may cause chaos for the data pipeline maintainers.  

this is a general guide for the data processing/ML processing best practises or some useful code snippets might be helpful to the data pipeline jobs. 

# infra consideration
There are many different data ETL platforms. From the traditional Teradata, Vertica, Oracle, to the big data tech stack Cloudera (hadoop), and until recently the snowflake, databricks, and the cloud provider AWS/Azure/Google, they all have their own bigdata solutions and platforms, so the usage might be different. 
Here we would considering the open source solutions, and mainly focused on spark. other tools might also be included later. 

# tools consideration
Data processing normally have following several stages: 
1. data sources integration (connection) various data source data extract solutions. for databases, like oracle, with goldgate, and for postgres and mysql, there is CDC connectors. there are also requirements for the files / dumpfiles / csvs / excels etc. 
2. datalake storage. in the old times, there is only one solutions: hadoop (hdfs), now the modern solutions delta lake from databricks, and iceberg, and various cloud providers objects storages.
3. job scheduler. then according to different jobs, there are batch jobs/streaming jobs, and maybe thousands of tables requires daily/hourly/minutes tasks to be scheduled .   
4. ETL jobs. there are data cleansing/ data transformation/ table joining/depend on each other, so the job scheduler should be able to schedule these dependencies accordingly. normally would be handled in one tool. airflow is a popular opensource scheduler solutions for the data etl. it's used for many production environments. there are some commercial solutions, for hadoop it's workbench in Cloudera. 
5. ingest the results into result destination.  it depends on how you are going to use the data. if for charts display(which is 80% of the case), then we should ingest the result data set into a mpp database or relational database (oracle/postgres if no bigger result) or in a distributed mpp (greenplum/clickhouse/Doris/Starrocks). because of the destination might be difference systems, so the final datasets for different destinations might be diffcult, so there are presto (the OLAP query engine) which could support many data destinations. 
Notes: this also called data layers:  data lake -> data warehousing -> data mart
a typical datalake is hadoop (store the ODS layer. ), and then in the hive (for the data warehousing, to store the various cleaned data cubes), finanaly in the database or mpp for the display of the charts or the api calls ( called data mart). 


# best practise consideration
We want to be independent for the data storage. The computing and ingestion could be separated with the data storage nowadays, it ofen called an architecture that storage and computing separation. 

there are some pros and cons for the 








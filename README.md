# Change Data Capture
## Postgres DB >> Debezium >> Cloud Pub/Sub >> BigQuery

### Objective

Replicate Postgres DB changes (INSERT, UPDATE, DELETE) thru CDC Debezium Connector by reading Postgres DB WAL (write ahead log) file.
Captured DB changes are sent to Cloud Pub/Sub and pushed to BigQuery thru the template Dataflow Job, Pub/Sub to BigQuery.

### Prerequisites
Commands executed in this tutorial where performed in WSL (Windows Subsystem for Linux) in an Ubuntu distribution.
Needs Java to be installed in the Ubuntu distribution. You can check it with command *java --version*. If not installed please install it by running the following commands.
- *sudo apt-get update*
- *sudo apt install openjdk-11-jre-headless*

Create a folder where we will be storing all the files needed, such as Debezium Connector files, SQL scripts, Cloud SQL Proxy files, etc. As of this example we created a folder called *debezium-server-postgres* by runnging command:
- *mkdir debezium-server-postgres*

Install the Postgres database client by running the following command:
- *sudo apt-get install postgre-client*

### Cloud SQL Postgres Instance Deployment

1) Deploy a Cloud SQL instance (Postgres) with the name of your preference in Google Cloud Console.
As of this example, these are the chosen names for instance, database, schema and table:  

    - instance name: *cdc-postgres-instance*  
    - database name: *postgres*  
    - schema: *inventory*
    - table: *customers*  

    IMPORTANT: Expand the "Show Configurations Options" to customize your instance and go to FLAGS section. Select from the dropdown the FLAG "cloudsql.logical_decoding" and set it to ON. This feature will allow logical decoding on the Cloud SQL instance to convert WAL (write ahead log) entries to JSON format. These JSON entries will be published to the Cloud Pub/Sub topic that we will later create.

2) Once the Cloud SQL database (Postgres) has been deployed, connect to your SQL instance using Google Cloud SDK shell with command:  
- *gcloud sql connect \<*instance-name*\> --database=\<*database*\> --user=\<*user*\>*

3) Copy and paste SQL statements in file script-inventory.sql and run them to create the template data to be used for the puspose of this tutorial.  

4) Execute the following command to allow database replication for the user the Debezium Connector will use to connect. After performing this step Google Cloud SDK shell can be closed.

- *ALTER USER existing_user WITH REPLICATION;*

### Cloud SQL Auth Proxy

To connect to our Cloud SQL instance we will be using Cloud SQL proxy.

Download the Cloud SQL Proxy using the following command:  
- *VERSION=v1.21.0*  
- *wget "https://storage.googleapis.com/cloudsql-proxy/$VERSION/cloud_sql_proxy.linux.amd64" -O cloud_sql_proxy*

Give execution permissions to the file using the following command:  
- *chmod +x cloud_sql_proxy*

Connect to your Cloud SQL instance using the following command:  
- *./cloud_sql_proxy --instances=\<project\>:\<location\>:\<instance-name\>=tcp:0.0.0.0:5432*

Check tables existance by running the following query:  
 - *SELECT table_schema, table_name*  
*FROM information_schema.tables*  
*WHERE table_schema = 'inventory'*  
*ORDER BY table_name;*

Leave this terminal open as we will be issuing SQL statements to generate changes in the database data and test debezium connector and its change data capture functionality.

### Cloud Pub/Sub - Topics creation

Create a Cloud Pub/Sub topic with the following naming convertion:  
\<instance-name\>.\<schema\>.\<table\>

Our example will use the customers table (one of the table created with the SQL script we ran before).
Topic name for this example: *cdc-postgres-instance.inventory.customers*

When creating the topic, select the option "Add a default subscription".

### Debezium Connector

1) Download from the Debezium connector from:  
https://debezium.io/documentation/reference/1.6/operations/debezium-server.html

2) Once downloaded, extract the content and paste it in the *debezium-server-postgres* folder we created.

3) Before putting the Debezium Connector to run, we will first need to edit the properties on the configuration file provided by Debezium. On the folder we created before, the *debezium-server-postgres*, enter subfolder *conf* and rename file *application.properties.example* to *application.properties*.

4) Open the file to edit it. Copy / paste the list of properties below and kindly replace with your correspondent property values.

        debezium.sink.pravega.scope=''  
        debezium.sink.type=pubsub  
        debezium.sink.pubsub.project.id=mimetic-might-312320  
        debezium.sink.pubsub.ordering.enabled=false  
        debezium.format.value=json  
        debezium.format.value.schemas.enable=false  
        debezium.source.connector.class=io.debezium.connector.postgresql.PostgresConnector  
        debezium.source.offset.storage.file.filename=data/offsets.dat  
        debezium.source.offset.flush.interval.ms=0  
        debezium.source.database.hostname=localhost  
        debezium.source.database.port=5432  
        debezium.source.database.user=postgres  
        debezium.source.database.password=postgres  
        debezium.source.database.dbname=postgres  
        debezium.source.database.server.name=cdc-postgres-instance  
        debezium.source.table.include.list=inventory.customers  
        #debezium.source.schema.include.list=transactional  
        debezium.source.plugin.name=wal2json  

    Main properties that need to be edited based on your project preferences / configurations:

        - debezium.sink.pubsub.project.id
        - debezium.source.database.user
        - debezium.source.database.password
        - debezium.source.database.dbname
        - debezium.source.database.server.name
        - debezium.source.table.include.list

5) Create folder data and and an empty file called "*offsets.dat*".

- *mkdir data*
- *touch data/offsets.dat*

6) Execute from the terminal the executable file run.sh with the following command:

- *run.sh*

### Testing the Debezium Connector and checking messages reception on Google Cloud Pub/Sub

1) Execute a quick SELECT statement to check the available information in the *customers* table.

    *SELECT * FROM postgres.inventory.customers;*

    id  | first_name | last_name |         email  
    ------+------------+-----------+-----------------------  
    1001 | Sally      | Thomas    | sally.thomas@acme.com  
    1002 | George     | Bailey    | gbailey@foobar.com  
    1003 | Edward     | Walker    | ed@walker.com  
    1004 | Anne       | Kretchmar | annek@noanswer.org  

2) We will update user 1001 with the following statement to test the Debezium Connector capturing feature:

 - *UPDATE postgres.inventory.customers  
    SET first_name = 'your-name',  
    last_name = 'your last-name',  
    email = 'your email'  
    WHERE id = 1001;*  

3) Once executed, go to Google Cloud Pub/Sub and verify the reception of the message on the subscription of the topic create by selecting option "View Messages" >> "Pull"

    You should be able to see a message with the following message body:

    {  
    "before":{  
    &nbsp;&nbsp;&nbsp;&nbsp;"id":1001,  
    &nbsp;&nbsp;&nbsp;&nbsp;"first_name":"Sally",  
    &nbsp;&nbsp;&nbsp;&nbsp;"last_name":"Thomas",  
    &nbsp;&nbsp;&nbsp;&nbsp;"email":"sally.thomas@acme.com"  
    },  
    "after":{  
    &nbsp;&nbsp;&nbsp;&nbsp;"id":1001,  
    &nbsp;&nbsp;&nbsp;&nbsp;"first_name":"your-name",  
    &nbsp;&nbsp;&nbsp;&nbsp;"last_name":"your last-name",  
    &nbsp;&nbsp;&nbsp;&nbsp;"email":"your email"  
    },  
    "source":{  
    &nbsp;&nbsp;&nbsp;&nbsp;"version":"1.6.0.Final",  
    &nbsp;&nbsp;&nbsp;&nbsp;"connector":"postgresql",  
    &nbsp;&nbsp;&nbsp;&nbsp;"name":"cdc-postgres-instance",  
    &nbsp;&nbsp;&nbsp;&nbsp;"ts_ms":1626716201707,  
    &nbsp;&nbsp;&nbsp;&nbsp;"snapshot":"false",  
    &nbsp;&nbsp;&nbsp;&nbsp;"db":"postgres",  
    &nbsp;&nbsp;&nbsp;&nbsp;"sequence":"[null,\"118013376\"]",  
    &nbsp;&nbsp;&nbsp;&nbsp;"schema":"inventory",  
    &nbsp;&nbsp;&nbsp;&nbsp;"table":"customers",  
    &nbsp;&nbsp;&nbsp;&nbsp;"txId":2731,  
    &nbsp;&nbsp;&nbsp;&nbsp;"lsn":118069040,  
    &nbsp;&nbsp;&nbsp;&nbsp;"xmin":null  
    },  
    "op":"u",  
    "ts_ms":1626716196328,  
    "transaction":null  
    }  

Note that there are two main structures in the json file which are "before" and "after", reflecting the update performed in Postgres database. Also additional metadata information is included in the "source" structure and on the root of the json file, which is valuable information that we will later use in BigQuery to get the most recent version of each record in the table.

### Streaming Inserts to BigQuery with Cloud Dataflow

1) Go to Cloud Dataflow and select option "Create Job from Template".

2) Fill in a Job Name of your preference, e.g: *cdc-pubsub-bigquery*.

3) Select Dataflow template: "Pub/Sub Topic to BigQuery".

4) Insert the Pub/Sub topic as projects/\<project\>/topics/\<topic-name\>

5) Insert the BigQuery output table as \<project\>:\<dataset\>.\<table\>

6) Insert a Temporary Location (a GCS bucket) where Cloud Dataflow will leave intermediatte files, product of its processing, e.g: gs://cdc-postres-debezium-bq/temp

7) Leave all the other parameters with the options selected by default and press "RUN JOB".



SELECT * EXCEPT(op, row_num)  
FROM (  
  SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY ts_ms DESC) AS row_num  
  FROM (  
    SELECT after.id, after.first_name, after.last_name, after.email, ts_ms, op  
    FROM `mimetic-might-312320.gentera.customers_delta`  
    UNION ALL  
    SELECT *, 'I'  
    FROM `mimetic-might-312320.gentera.customers_main`))  
WHERE  
  row_num = 1  
  AND op <> 'D'  

SELECT table_schema, table_name  
FROM information_schema.tables  
WHERE table_schema = 'inventory'  
ORDER BY table_name;  

MERGE `mimetic-might-312320.gentera.customers_main` m  
USING  
  (  
  SELECT * EXCEPT(row_num)  
  FROM (  
    SELECT *, ROW_NUMBER() OVER(PARTITION BY COALESCE(after.id,before.id) ORDER BY delta.ts_ms DESC) AS row_num  
    FROM `mimetic-might-312320.gentera.customers_delta` delta )  
  WHERE row_num = 1) d  
ON  m.id = COALESCE(after.id,before.id)  
  WHEN NOT MATCHED  
AND op IN ("c", "u") THEN  
INSERT (id, first_name, last_name, email, ts_ms)  
VALUES (d.after.id, d.after.first_name, d.after.last_name, d.after.email, d.ts_ms)  
  WHEN MATCHED  
  AND d.op = "d" THEN  
DELETE  
  WHEN MATCHED  
  AND d.op = "u"  
  AND (m.ts_ms < d.ts_ms) THEN  
UPDATE  
SET first_name = d.after.first_name, last_name = d.after.last_name, email = d.after.email, ts_ms = d.ts_ms  
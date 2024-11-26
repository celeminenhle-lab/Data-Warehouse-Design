# Data-Warehouse-Design
Real-time Data Warehouse using the following technologies: 







Getting the setup up and running

bash
Copy code
docker compose build
docker compose up -d
docker compose ps
Access the Flink Web UI at http://localhost:8081 and Kibana at http://localhost:5601.

Postgres
Start the Postgres client:

bash
Copy code
docker compose exec postgres env PGOPTIONS="--search_path=claims" bash -c 'psql -U $POSTGRES_USER postgres'
To list tables:

sql
Copy code
SELECT * FROM information_schema.tables WHERE table_schema = 'claims';
Debezium
Start the Postgres connector:

bash
Copy code
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres.json
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres-members.json
Check connector status:

bash
Copy code
curl http://localhost:8083/connectors/claims-connector/status
Test data changes in Kafka:

bash
Copy code
docker compose exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic pg_claims.claims.accident_claims
Data changes
Run these queries to observe change propagation:

sql
Copy code
INSERT INTO accident_claims (claim_total, claim_total_receipt, claim_currency, member_id, accident_date, accident_type, accident_detail, claim_date, claim_status) VALUES (500, 'PharetraMagnaVestibulum.tiff', 'AUD', 321, '2020-08-01 06:43:03', 'Collision', 'Blue Ringed Octopus', '2020-08-10 09:39:31', 'INITIAL');
UPDATE accident_claims SET claim_total_receipt = 'CorrectReceipt.pdf' WHERE claim_id = 1001;
DELETE FROM accident_claims WHERE claim_id = 1001;
Flink connectors
Refer to Flink documentation for available connectors.

Datasource ingestion
Start Flink SQL Client:

bash
Copy code
docker compose exec sql-client ./sql-client.sh
docker compose exec sql-client ./sql-client-submit.sh
Create table example:

sql
Copy code
CREATE TABLE t1(
  uuid VARCHAR(20), 
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = '/data/t1',
  'read.streaming.enabled' = 'true'
);
Postgres catalog registration

sql
Copy code
CREATE CATALOG datasource WITH (
    'type'='jdbc',
    'base-url'='jdbc:postgresql://postgres:5432/',
    'default-database'='postgres',
    'username'='postgres',
    'password'='postgres'
);
Create and populate tables in the datasource layer, DWD layer, and DWB layer as shown.

Continuous queries
Write data to DWD tables:

sql
Copy code
INSERT INTO dwd.accident_claims
SELECT claim_id, claim_total, claim_total_receipt, claim_currency, member_id, 
       CAST (accident_date as DATE), accident_type, accident_detail, 
       CAST (claim_date as DATE), claim_status, 
       CAST (ts_created as TIMESTAMP), CAST (ts_updated as TIMESTAMP), claim_date
FROM datasource.accident_claims;
Query data for verification:

sql
Copy code
SELECT * FROM dwd.accident_claims;
SELECT * FROM dwd.members;

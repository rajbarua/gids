
# GIDS Demo

## Supercharging Kafka Applications with Real-time Contextual Insights
### Setup
1. Run kafka `docker pull apache/kafka:3.7.0` and then `docker run -d -p 9092:9092 --name kafka apache/kafka:3.7.0`
1. Start Hazelcast `hz start`. See [installation](https://docs.hazelcast.com/hazelcast/latest/getting-started/install-hazelcast) guidelines. If you use [EE](https://docs.hazelcast.com/hazelcast/latest/getting-started/install-enterprise) then you need to `export HZ_LICENSEKEY=$(cat ~/hazelcast/demo.license)`.
3. Also run Management Centre as `hz-mc start`
1. Configure CLC by adding a config `clc config add dev cluster.name=dev cluster.address=localhost:5701`
1. Connect to clc using `clc -c dev`
1. Optional: Exec into the container `docker exec -it kafka bash` and `cd opt/kafka/bin`
1. Optional: Follow the same steps in any SQL client, like DBeaver

### Demo
Goal is to read from a Kafka topic which is emitting a json with an e-commerce data stream. Each message contains - id, customer, price, order_ts and amount.
1. First we need to create a Mapping to that incoming messages
```sql
CREATE OR REPLACE MAPPING orders (
    id BIGINT,
    customer VARCHAR,
    price DECIMAL,
    order_ts TIMESTAMP,
    amount BIGINT)
TYPE KAFKA
OPTIONS(
    'valueFormat'='json-flat',
    'bootstrap.servers' = '127.0.0.1:9092'
);
``` 
2. Let's start reading from the topic
```sql
SELECT customer AS customer, ROUND(price,2) AS Price, amount AS "Sold" FROM orders;
```
Nothing shows. The issue is that `topic` has no data as there are no producers. So lets produce some random data and push into the topic.

3. Randomly create data and push into a Kafka topic (only to read again to process that data back inside Hazelcast).
Here is a sample _continuous query_ that will form the base of our data generator. It uses a built [stream generator](https://docs.hazelcast.com/hazelcast/5.3/sql/functions-and-operators#table-valued-functions).
```sql
SELECT v as id,
           RAND(v*v) as userRand,
           TO_TIMESTAMP_TZ(v*10 + 1645484400000) as order_ts,
           ROUND(RAND()*100, 0) as amount
     FROM TABLE(generate_stream(100));
```
We can now create a Hazelcast `JOB` that will generate and push sample data into kafka topic called `orders`.
```sql
CREATE JOB IF NOT EXISTS push_orders
    OPTIONS (
    'processingGuarantee' = 'exactlyOnce',
    'snapshotIntervalMillis' = '5000')
    AS
    SINK INTO orders (
        SELECT id,
        CASE WHEN userRand BETWEEN 0 AND 0.1 THEN 'Elmer Fudd'
                WHEN userRand BETWEEN 0.1 AND 0.2 THEN 'Speedy Gonzales'
                WHEN userRand BETWEEN 0.2 AND 0.3 THEN 'Daffy'
                WHEN userRand BETWEEN 0.3 AND 0.4 THEN 'Sylvester'
                WHEN userRand BETWEEN 0.4 AND 0.5 THEN 'Porky'
                WHEN userRand BETWEEN 0.5 AND 0.6 THEN 'Tweety'
                WHEN userRand BETWEEN 0.6 AND 0.7 THEN 'Bugs Bunny'
                WHEN userRand BETWEEN 0.7 AND 0.8 THEN 'Marvin Martian'
                ELSE 'Wile E. Coyote'
        END as customer,
        CASE WHEN userRand BETWEEN 0 and 0.1 then userRand*50+1
                WHEN userRand BETWEEN 0.1 AND 0.2 THEN userRand*75+.6
                WHEN userRand BETWEEN 0.2 AND 0.3 THEN userRand*60+.2
                WHEN userRand BETWEEN 0.3 AND 0.4 THEN userRand*30+.3
                WHEN userRand BETWEEN 0.4 AND 0.5 THEN userRand*43+.7
                WHEN userRand BETWEEN 0.5 AND 0.6 THEN userRand*100+.4
                WHEN userRand BETWEEN 0.6 AND 0.7 THEN userRand*25+.8
                WHEN userRand BETWEEN 0.6 AND 0.7 THEN userRand*80+.5
                WHEN userRand BETWEEN 0.7 AND 0.8 THEN userRand*10+.1
                ELSE userRand*100+4
        END as price,
        order_ts,
        amount
        FROM
            (SELECT v as id,
                RAND(v*v) as userRand,
                TO_TIMESTAMP_TZ(v*10 + 1645484400000) as order_ts,
                ROUND(RAND()*100, 0) as amount
            FROM TABLE(generate_stream(100)))
    );
```
4. Let's query the topic again
```sql
SELECT customer AS customer, ROUND(price,2) AS Price, amount AS "Sold" FROM orders;
```
Limit the output to one customer.
```sql
SELECT customer AS customer, ROUND(price,2) AS Price, amount AS "Sold"
FROM orders
WHERE customer = 'Wile E. Coyote';
```
Limit the output to one symbol and sales of over 50 shares.
```sql
SELECT customer AS customer, ROUND(price,2), amount AS "Sold"
FROM orders
WHERE customer = 'Daffy' AND amount > 50;
```
3. Create data to enrich the streaming data from kafka topic
In order to demonstrate enrichment of streaming data, We can now use Hazelcast IMap to enrich incoming data. Let us create a mapping to a Hazelcast Map that will store store `long` as `key` and `json` as `value` 
```sql
CREATE or REPLACE MAPPING enrich (
    __key BIGINT,
    customer VARCHAR,
    colour1 VARCHAR,
    colour2 VARCHAR,
    colour3 VARCHAR )
TYPE IMap
OPTIONS (
    'keyFormat'='bigint',
    'valueFormat'='json-flat');
```
Let's add some data into into this Map that we will use for enriching the messages on the topic.
```sql
INSERT INTO enrich VALUES
(1, 'Elmer Fudd', 'blue','red','green'),
(2, 'Speedy Gonzales', 'green', 'blue', 'blue'),
(3, 'Daffy', 'blue','green', 'blue'),
(4, 'Sylvester', 'blue','blue', 'blue'),
(5, 'Porky', 'blue', 'red', 'green'),
(6, 'Tweety', 'blue', 'red', 'blue'),
(7, 'Bugs Bunny', 'green', 'red', 'blue'),
(8, 'Marvin Martian', 'green', 'red', 'blue'),
(9, 'Wile E. Coyote', 'blue','green','red');
```
4. Let's _enrich_ the stream with lookup data inside Hazelcast
Following SQL `joins` the streaming data from kafka and IMap stored in Hazelcast
```sql
SELECT
    orders.customer AS Symbol,
    enrich.colour1 as colour1,
    enrich.colour2 as colour2,
    enrich.colour3 as colour3,
     ROUND(orders.price,2) AS Price,
     orders.amount AS "Sold"
FROM orders
JOIN enrich
ON enrich.customer = orders.customer 
AND enrich.colour2 = 'red';
```
5. While we can run this as an ad-hoc continuous query but we can also run as a job to store that into an IMap
First create a Mapping to this new IMap
```sql
CREATE or REPLACE MAPPING destination_map(
    __key BIGINT,
    customer VARCHAR,
    colour1 VARCHAR,
    colour2 VARCHAR,
    colour3 VARCHAR,
    price DECIMAL,
    amount BIGINT  )
TYPE IMap
OPTIONS (
    'keyFormat'='bigint',
    'valueFormat'='json-flat');
```
Now lets push data into an IMap
```sql
CREATE JOB IF NOT EXISTS push_orders_imap
AS
    SINK INTO destination_map (
        SELECT
            orders.id AS Id,
            orders.customer AS Symbol,
            enrich.colour1 as colour1,
            enrich.colour2 as colour2,
            enrich.colour3 as colour3,
            ROUND(orders.price,2) AS Price,
            orders.amount AS "Sold"
        FROM orders
        JOIN enrich
        ON enrich.customer = orders.customer 
        AND enrich.colour2 = 'red'
    );
```
Cancel job even
```sql
ALTER JOB push_orders_imap SUSPEND;
```
6. We can also join two streams if needed
First we create one more view of the order view 
```sql
CREATE OR REPLACE VIEW orders_ordered AS
SELECT *
FROM TABLE(IMPOSE_ORDER(
    TABLE orders,
    DESCRIPTOR(order_ts),
    INTERVAL '0.5' SECONDS));
```
```sql
CREATE OR REPLACE VIEW high_low AS
SELECT
     window_start,
     window_end,
     customer,
     ROUND(MAX(price),2) AS high,
     ROUND(MIN(price),2) AS low
FROM TABLE(TUMBLE(
     TABLE orders_ordered,
     DESCRIPTOR(order_ts),
     INTERVAL '5' SECONDS
))
GROUP BY 1,2,3;
```
```sql
SELECT
     tro.customer AS Customer,
     tro.price AS Price,
     hl.high AS High,
     hl.low AS Low
FROM orders_ordered AS tro
JOIN high_low AS hl
ON tro.customer= hl.customer
AND hl.window_end BETWEEN tro.order_ts AND tro.order_ts + INTERVAL '0.1' SECONDS;
```

## Similarity Search
We will use a local Hazelcast deployment and MC along with Python virtual environment. We will use [pyenv](https://github.com/pyenv/pyenv) to manage multiple python version on the local machine. For this demo we will use Python 3.11.
### Installations
This guide is written for local installation of Hazelcast but it can be extended for cloud and other installations.
1. Make sure Hazelcast is installed via brew or other ways
1. Install `pyenv` using `brew install pyenv` and `echo 'eval "$(pyenv init -)"' >> ~/.zshrc`
1. Install python 3.11 latest as `pyenv install 3.11`
1. Change into project directory and switch to python 3.7 using `cd ~/src/gids; pyenv local 3.11`
1. Then create a pythin virtual environment and activate it `python -m venv .venv`, `source .venv/bin/activate`
1. Install Python dependencies `pyenv exec pip install -U ipykernel qdrant-client sentence-transformers hazelcast-python-client jupyter`
1. Run qdrant `docker run -p 6333:6333 -v $(pwd)/qdrant_storage:/qdrant/storage qdrant/qdrant`
1. Start Hazelcast `hz start` and optionally Management Centre `hz-mc start`
1. Switch to Notebook `jupyter notebook`


### References
1. [Blog](https://dzone.com/articles/boosting-similarity-search-with-stream-processing)

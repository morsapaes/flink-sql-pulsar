# Flink SQL for Pulsar Folks

> :warning: **Update:** This repository will no longer be actively maintained. Please check the [Ververica fork](https://github.com/ververica).

[Apache Flink](https://flink.apache.org/) and [Apache Pulsar](https://pulsar.apache.org/) share a common vision around unifying batch and streaming: on one side, batch is seen as just a [special case of streaming](https://youtu.be/h5OYmy9Yx7Y), while on the other streams serve as a [unified view on data](https://medium.com/streamnative/apache-pulsar-as-one-storage-455222c59017). The main differentiating use case for Flink + Pulsar is to simplify building a one-stop-shop for storing and processing both real-time and historic data.

<p align="center">
<img width="1008" alt="Screen Shot 2021-05-11 at 15 44 58" src="https://user-images.githubusercontent.com/23521087/117826785-b9e7f580-b270-11eb-85a9-828baf461ffd.png">
</p>

This demo walks you through building an analytics application using Pulsar and Flink SQL.

## Docker

To keep things simple, the demo uses a Docker Compose setup that makes it easier to bundle up all the services you need.

#### Getting the setup up and running

`docker-compose build`

`docker-compose up -d`

#### Is everything really up and running?

`docker-compose ps`

You should be able to access the Flink Web UI (http://localhost:8081).

## Pulsar

You’ll use the [Twitter Firehose built-in connector](https://pulsar.apache.org/docs/en/io-twitter-source) to consume tweets about `gardening` into a Pulsar topic. To create the Twitter source, run:

```bash
docker-compose exec pulsar ./bin/pulsar-admin source create \
  --name twitter \
  --source-type twitter \
  --destinationTopicName tweets \
  --source-config '{"consumerKey":<consumerKey>,"consumerSecret":<consumerSecret>,"token":<token>,"tokenSecret":<tokenSecret>, "terms":"gardening"}'
```

> :information_source: This source requires a valid Twitter authentication token, which you can generate on the [Twitter Developer Portal](https://developer.twitter.com/en/docs/authentication/oauth-1-0a/obtaining-user-access-tokens).

After creating the source, you can check that data is flowing into the `tweets` topic:

```bash
docker-compose exec pulsar ./bin/pulsar-client consume -n 0 -r 0 -s test tweets
```

At any point, you can also [stop](https://pulsar.apache.org/docs/en/io-use/#stop-a-connector) the connector:

```bash
docker-compose exec pulsar ./bin/pulsar-admin sources stop --name twitter
```

## Flink SQL

Next, you can start the Flink SQL Client:

```bash
docker-compose exec sql-client ./sql-client.sh
```

and use a [Pulsar catalog](https://github.com/streamnative/pulsar-flink#catalog) to access the topic directly as a table in Flink. This will make some things a lot easier afterwards, too!

```sql
CREATE CATALOG pulsar WITH (
   'type' = 'pulsar',
   'service-url' = 'pulsar://pulsar:6650',
   'admin-url' = 'http://pulsar:8080',
   'format' = 'json'
);

USE CATALOG pulsar;

SHOW TABLES;
```

You can query the `tweets` table off-the-bat using a simple `SELECT` statement — so, you now have tweets making their way from Twitter Firehose to Pulsar to Flink!

### Using Pulsar Metadata

If you look closely, most events have a null `createdAt` value. What now?

![4_flink_sql_select](https://user-images.githubusercontent.com/23521087/117856935-887d2300-b28c-11eb-913a-edadb1b8a8de.gif)

One way to get a relevant timestamp is to tap into [Pulsar metadata](https://github.com/streamnative/pulsar-flink#metadata-configurations) to get the `publishTime` (i.e. ingestion time). The cool thing about using catalogs is being able to create a table with the exact same schema as the original topic by just using a `CREATE TABLE LIKE` statement:

```sql
CREATE TABLE default_catalog.default_database.pulsar_tweets 
(
  publishTime TIMESTAMP(3) METADATA,
  WATERMARK FOR publishTime AS publishTime - INTERVAL '5' SECOND
) WITH (
  'connector' = 'pulsar',
  'topic' = 'persistent://public/default/tweets',
  'value.format' = 'json',
  'service-url' = 'pulsar://pulsar:6650',
  'admin-url' = 'http://pulsar:8080',
  'scan.startup.mode' = 'earliest'
)
LIKE tweets;
```

In the DDL above, you're using the [Pulsar Flink connector](https://github.com/streamnative/pulsar-flink#pulsar-flink-connector), tapping into the `tweets` topic, and using the `JSON` format to deserialize the events. And because you're fetching the `publishtime` and defining it as a [watermark](https://ci.apache.org/projects/flink/flink-docs-stable/docs/dev/table/sql/create/#watermark), you now also have some notion of time in your application!

### Producing Aggregated Results to Pulsar

To close the loop, you can create a sink table backed by a new `tweets_agg` topic in Pulsar, and insert into it using a simple windowed aggregation:

```sql
CREATE TABLE pulsar_tweets_agg (
tmstmp TIMESTAMP(3),
  tweet_cnt BIGINT
) WITH (
  'connector'='pulsar',
  'topic'='persistent://public/default/tweets_agg',
  'value.format'='json',
  'service-url'='pulsar://pulsar:6650',
  'admin-url'='http://pulsar:8080'
 );

INSERT INTO pulsar_tweets_agg
SELECT TUMBLE_START(publishTime, INTERVAL '10' SECOND) AS wStart,
       COUNT(id) AS tweet_cnt
FROM pulsar_tweets
GROUP BY TUMBLE(publishTime, INTERVAL '10' SECOND);
```

You can see that the query is pretty much something you could run on a regular database — just standard SQL doing some aggregation of the total number of tweets over windows of 10 seconds. Once you submit this query, it will run continuously and continuously sink the results into Pulsar. To monitor the execution of the query or cancel it, use the [Flink Web UI](http://localhost:8081)!

<hr>

**And that's it!**

For an overview of the evolution of the Flink Pulsar integration over time, check out these [slides](https://noti.st/morsapaes/wU2kF1/select-star-flink-sql-for-pulsar-folks), and follow [Apache Flink](https://twitter.com/ApacheFlink) and [Apache Pulsar](https://twitter.com/apache_pulsar) on Twitter for the latest updates.
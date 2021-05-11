# Analyzing Twitch Streams with Flink SQL and Apache Pulsar

## Docker

To keep things simple, this demo uses a Docker Compose setup that makes it easier to bundle up all the services you need:

<p align="center">
<img width="700" alt="demo_overview" src="https://user-images.githubusercontent.com/23521087/90702325-3c67f980-e28b-11ea-8496-9f237ceeae8b.png">
</p>

#### Getting the setup up and running

`docker-compose build`

`docker-compose up -d`

#### Is everything really up and running?

`docker-compose ps`

You should be able to access the Flink Web UI (http://localhost:8081).


```bash
docker-compose exec pulsar ./bin/pulsar-admin source create \
  --name twitter \
  --source-type twitter \
  --destinationTopicName tweets \
  --source-config '{"consumerKey":<consumerKey>,"consumerSecret":<consumerSecret>,"token":<token>,"tokenSecret":<tokenSecret>, "terms":"gardening"}'
```

At any point, you can [stop](https://pulsar.apache.org/docs/en/io-use/#stop-a-connector) the connector using:

```bash
docker-compose exec pulsar ./bin/pulsar-admin sources stop --name twitter
```

```bash
docker-compose exec pulsar ./bin/pulsar-admin schemas get tweets
```

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
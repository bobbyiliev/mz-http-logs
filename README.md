# [Materialize - Log Parsing Demo](https://materialize.com/docs/demos/log-parsing/)

This is a self-contained demo using [Materialize](https://materialize.com) to parse server logs for a mock e-commerce site, and extract some business insights from them.

For more information about this demo and the “what” and “why” of our proposed deployment using Materialize to provide real-time log parsing make sure to check out the official documentation here:

> [Materialize - Log Parsing Demo](https://materialize.com/docs/demos/log-parsing/)

## Prerequisites 

Before you get started, you need to make sure that you have Docker and Docker Compose installed.

You can follow the steps here on how to install Docker:

> [Installing Docker](https://materialize.com/docs/third-party/docker/)

## Running the demo

Once you have Docker and Docker Compose installed, you can follow the steps here in order to get the demo up and running.

### Cloning the repository

First things first, before you could run the demo, you need to clone the repository:

```bash
git clone https://github.com/bobbyiliev/mz-http-logs.git
```

Once that is done, switch to the repository directorly:

```bash
cd mz-http-logs
```

### Starting all serivices

To start all services just execute this single `docker-compose` command:

```bash
docker-compose up -d
```

This will start all of the services specified in the `docker-compose.yaml` file in a detached mode.

### Launch Materialize CLI (`mzcli`)

In order to access the Materialize CLI (`mzcli`) container run the following command:

```bash
docker-compose run mzcli
```

### Create a Source from logs

As soon as you have started all of the services, the demo application would be constantly under load and the logs will be populated with home page views, product searches and requests to the product details pages.

To tap into this log file that is constantly being updated, we can use the `CREATE SOURCE`:

```sql
CREATE SOURCE requests
FROM FILE '/log/requests' WITH (tail = true)
FORMAT REGEX '(?P<ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) - - \[(?P<ts>[^]]+)\] "(?P<path>(?:GET /search/\?kw=(?P<search_kw>[^ ]*) HTTP/\d\.\d)|(?:GET /detail/(?P<product_detail_id>[a-zA-Z0-9]+) HTTP/\d\.\d)|(?:[^"]+))" (?P<code>\d{3}) -';
```

Then to verify that the source was created, you can use the following statement:

```sql
SHOW SOURCES;
```

Output:

```
+-----------+
| SOURCES   |
|-----------|
| requests  |
+-----------+
```

### Check the available columns

We can look at the structure of the `requests` source with `SHOW COLUMNS`:

```sql
SHOW COLUMNS FROM requests;
```

Output:

```
+-------------------+------------+--------+
| name              | nullable   | type   |
|-------------------+------------+--------|
| ip                | true       | text   |
| ts                | true       | text   |
| path              | true       | text   |
| search_kw         | true       | text   |
| product_detail_id | true       | text   |
| code              | true       | text   |
| mz_line_no        | false      | int8   |
+-------------------+------------+--------+
```

### Creating a Materialized view

And then we can create a materialized view that embeds this query:

```sql
CREATE MATERIALIZED VIEW unique_visitors AS
    SELECT count(DISTINCT ip) FROM requests;
```
### Query the Materialized view

To view the results of the query, run the following statement:

```sql
SELECT * FROM unique_visitors;
```

You’ll note that the result should come back pretty quickly.

In case that you needed to check the query used to create the view, you can use `SHOW CREATE VIEW`:

```sql
SHOW CREATE VIEW unique_visitors;
```

## More demos

* [Microservice demo](https://materialize.com/docs/demos/microservice)
* [Business intelligence demo](https://materialize.com/docs/demos/business-intelligence)
* [`CREATE SOURCE`](https://materialize.com/docs/sql/create-source)
* [Functions + Operators](https://materialize.com/docs/sql/functions)
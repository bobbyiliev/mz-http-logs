# [Materialize - Log Parsing Demo](https://materialize.com/docs/demos/log-parsing/)

This is a self-contained demo using [Materialize](https://materialize.com) to parse server logs for a mock e-commerce site, and extract some business insights from them.

![Materialized Log Parssing Demo](https://user-images.githubusercontent.com/21223421/141309644-d80cffe4-39f9-4afa-a211-907f9de7d74e.png)

## Introduction

Servers, especially busy ones, can emit a vast amount of logging data. Given that logs are unstructured strings, it’s challenging to draw inferences from them. This inaccessibility often means that even though logs contain a lot of potential, teams don’t capitalize on them.

Materialize, though, offers the chance to extract and query data from your logs. An instance of `materialized` can continually read a log file, impose a structure on it, and then let you define views you want to maintain on that data—just as you would with any SQL table. This opens up new opportunities for both real-time business and operational analysis.

In this demo, we’ll look at parsing server logs for a mock e-commerce site, and extracting some business insights from them.

## Prerequisites

Before you get started, you need to make sure that you have Docker and Docker Compose installed.

You can follow the steps here on how to install Docker:

> [Installing Docker](https://materialize.com/docs/third-party/docker/)

## Diagram

![Materialize Log Parsing demo Diagram](https://materialize.com/docs/images/demos/log_parsing_architecture_diagram.png)

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

#### Understand the logs' structure and the regex

Log files are often just strings of text delimited by newlines—this makes them difficult to use in a relational model. However, given that they’re formatted consistently, it’s possible to impose structure on the logs with regular expressions–which is exactly what we’ll do.

Below is an example of a few lines from our web server’s log file:

```
248.87.122.109 - - [28/Jan/2020 17:08:19] "GET /search/?kw=K8oN HTTP/1.1" 200 -
203.153.87.134 - - [28/Jan/2020 17:08:19] "GET /search HTTP/1.1" 308 -
12.234.172.170 - - [28/Jan/2020 17:08:20] "GET /detail/HGwL HTTP/1.1" 200 -
```

We can see that some fields we might be interested in include:

| Field                | Example                          |
| -------------------- | -------------------------------- |
| IP addresses         | `248.87.122.109`                 |
| Timestamps           | `28/Jan/2020 17:08:19`           |
| Full page paths      | `/search/?kw=K8oN`               |
| Search terms         | `K8oN` in `GET /search/?kw=K8oN` |
| Viewed product pages | `HGwL` in `GET /detail/HGwL`     |
| HTTP status code     | `200`                            |

#### Impose a structure with regex

Once we understand our logs' structure, we can formalize it with a regular expression. In this example, we’ll use named capture groups to generate columns (e.g. `(?P<ip>...)` creates a capture group named ip).

While you don’t necessarily need to understand the following regex, here’s an example of how we can structure the above logs:

```
(?P<ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) - - \[(?P<ts>[^]]+)\] "(?P<path>(?:GET /search/\?kw=(?P<search_kw>[^ ]*) HTTP/\d\.\d)|(?:GET /detail/(?P<product_detail_id>[a-zA-Z0-9]+) HTTP/\d\.\d)|(?:[^"]+))" (?P<code>\d{3}) -
```

In this regex, we’ve created the following columns:

| Column              | Expresses |
| ------------------- | --------- |
| `ip`                | Users' IP address |
| `ts`                | Events' timestamp |
| `path`              | Paths where the event occurred |
| `search_kw`         | Keywords a user searched for |
| `product_detail_id` | IDs used to differentiate each product |
| `code`              | HTTP codes |

In Materialize, if a capture group isn’t filled by the input string, the row simply has a NULL value in the attendant column.

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

---

Let's create one more Materialized view and aggregate the logs:

```sql
CREATE MATERIALIZED VIEW aggregated_logs AS
  SELECT
    ip,
    path,
    code::int,
    COUNT(*) as count
  FROM requests GROUP BY 1,2,3;
```

The important things to note are:

* The moment you execute the statement, Materialize creates a dataflow to match the SQL
* Then Materialize processes each line of the log through the dataflow, and keeps listening for new lines. This is incredibly powerful for dashboards that rely on real-time data.

A quick rundown of the statement itself:

* First we start with the `CREATE MATERIALIZED VIEW aggregated_logs` which identifies that we want to create a new Materialized view. The `aggregated_logs` part is the name of our Materialized view.
* Then we specify the `SELECT` statement used to build the output. In this case we are aggregating by `ip`, `path` and `statuscode`, and we are counting the total instances of each combo with a `COUNT(*)`

Let's run a `SELECT` query to check out the results

```sql
SELECT * FROM unique_visitors ORDER BY count DESC LIMIT 100;
// Output:
       ip       |      path      | code | count 
----------------+----------------+------+-------
 18.120.103.2   | GET / HTTP/1.1 |  200 |    15
 2.65.37.39     | GET / HTTP/1.1 |  200 |    13
 127.23.43.9    | GET / HTTP/1.1 |  200 |    13
 29.120.64.86   | GET / HTTP/1.1 |  200 |    13
 82.27.85.125   | GET / HTTP/1.1 |  200 |    13
 112.69.118.96  | GET / HTTP/1.1 |  200 |    13
 115.118.92.80  | GET / HTTP/1.1 |  200 |    13
 60.104.117.114 | GET / HTTP/1.1 |  200 |    13
 0.67.28.9      | GET / HTTP/1.1 |  200 |    12
```

When creating a Materialized View, it could be based on multiple sources like your Kafka Stream, a raw data file that you have on an S3 bucket, and your PostgreSQL database. This single view will give you the power to analyze your data in real-time.

# Recap
In this demo, we saw:

* How to create a source from dynamic file
* How Materialize can structure log files
* How to define sources and views within Materialize
* How to query views to extract data from your logs

## Related pages

* [Microservice demo](https://materialize.com/docs/demos/microservice)
* [Business intelligence demo](https://materialize.com/docs/demos/business-intelligence)
* [`CREATE SOURCE`](https://materialize.com/docs/sql/create-source)
* [Functions + Operators](https://materialize.com/docs/sql/functions)
---
title: Get started
pcx_content_type: how-to
weight: 1
meta:
  title: Get started with Workers Analytics Engine
---

# Get started with Workers Analytics Engine

Workers Analytics Engine collects custom events from your Workers and lets you query them with SQL. The fastest way to try it is to complete the following workflow.

## 1. Enable Analytics Engine

1. Log into the [Cloudflare dashboard](https://dash.cloudflare.com) and select your account.
2. Go to **Workers & Pages** ▸ **Overview**.
3. In the right sidebar, find **Analytics Engine** and select **Set up** ▸ **Enable Analytics Engine**.

## 2. Configure a dataset binding in Wrangler

Workers Analytics Engine stores data in datasets (similar to SQL tables). Create a dataset binding in [Wrangler](/workers/wrangler/configuration/) so that your Worker can write to it. The dataset is created automatically the first time you write data.

{{<Aside type="note">}}
Use Wrangler `2.6.0` or later to define Analytics Engine bindings. You can [install or update Wrangler here](/workers/wrangler/install-and-update/).
{{</Aside>}}

Add the binding to your `wrangler.toml` file:

```toml
---
filename: wrangler.toml
---
[[analytics_engine_datasets]]
binding = "<BINDING_NAME>"
```

By default the dataset name matches the binding name. Make sure the binding value is [a valid JavaScript identifier](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Grammar_and_types#variables); the binding is exposed inside your Worker as `env.<BINDING_NAME>` with the `writeDataPoint()` method.

Optionally give the dataset a different name:

```toml
---
filename: wrangler.toml
---
[[analytics_engine_datasets]]
binding = "<BINDING_NAME>"
dataset = "<DATASET_NAME>"
```

Run `npx wrangler deploy` to redeploy your Worker with the new binding.

## 3. Write data from your Worker

Use the binding’s `writeDataPoint()` method to send events. Each data point contains:

* `blobs`: string fields for grouping/filtering (for example, city or sensor ID).
* `doubles`: numeric fields for aggregation (for example, temperature or latency).
* `indexes`: string fields used for [sampling](/analytics/analytics-engine/sql-api/#sampling).

Example Worker that records sensor data:

```js
  async fetch(request, env) {
    env.WEATHER.writeDataPoint({
      'blobs': ["Seattle", "USA", "pro_sensor_9000"],
      'doubles': [25, 0.5],
      'indexes': ["a3cd45"] // Sensor ID
    });
    return new Response("OK!");
  }
```

You can also log metadata from incoming requests by reusing existing [runtime variables](/workers/runtime-apis/request/). The following Worker (based on the [Geolocation Hello World example](/workers/examples/geolocation-hello-world/)) writes geographic information directly to Analytics Engine without redefining the fields:

```js
env.<EXAMPLE_DATASET>.writeDataPoint({
  'blobs': [ 
    request.cf.colo, 
    request.cf.country, 
    request.cf.city, 
    request.cf.region, 
    request.cf.timezone
  ],
  'doubles': [
    request.cf.metroCode, 
    request.cf.longitude, 
    request.cf.latitude
  ],
  'indexes': [
    request.cf.postalCode
  ] 
});
```

When writing data, keep blobs and doubles in a consistent order so you can reference them later when querying.

## 4. Query data with GraphQL or SQL

Query Workers Analytics Engine data through:

* [GraphQL](/analytics/graphql-api/) – best for powering dashboards with a simplified schema.
* [SQL API](/analytics/analytics-engine/sql-api/) – best for ad hoc queries, ClickHouse-compatible tools, or Grafana.

The SQL API is an HTTP endpoint at `https://api.cloudflare.com/client/v4/accounts/YOUR_ACCOUNT_ID/analytics_engine/sql` (use `POST` or `GET`). Authenticate with a Cloudflare [API Token](https://dash.cloudflare.com/profile/api-tokens) that has the **Account Analytics Read** permission. Tools like [Postman](https://www.postman.com/) can send the request if you prefer a GUI.

### Example of querying data with the SQL API

In the following example, we use the SQL API to query the top 10 cities that had the highest average humidity readings when the temperature was above zero.

Here is how we represent that as SQL. We are using a custom averaging function to take into account [sampling](/analytics/analytics-engine/sql-api/#sampling):

```sql
SELECT 
  blob1 AS city,
  SUM(_sample_interval * double2) / SUM(_sample_interval) AS avg_humidity
FROM WEATHER 
WHERE double1 > 0 
GROUP BY city 
ORDER BY avg_humidity DESC
LIMIT 10
```

Execute the query with any HTTP client, for example cURL:

```curl
curl -X POST "https://api.cloudflare.com/client/v4/accounts/YOUR_ACCOUNT_ID/analytics_engine/sql" -H "Authorization: Bearer YOUR_API_TOKEN" -d "SELECT blob1 AS city, SUM(_sample_interval * double2) / SUM(_sample_interval) AS avg_humidity FROM WEATHER WHERE double1 > 0 GROUP BY city ORDER BY avg_humidity DESC LIMIT 10"
```

Blobs and doubles currently use 1-based indexing (`blob1`, `double1`, etc.). Refer to the [SQL API docs](/analytics/analytics-engine/sql-api/) and the [SQL reference](/analytics/analytics-engine/sql-reference/) for the supported syntax.

### Working with time series

Workers Analytics Engine automatically adds a `timestamp` field to every event so that you can build time series views. Most queries round and `GROUP BY` the timestamp. For example:

```sql
SELECT
  intDiv(toUInt32(timestamp), 300) * 300 AS t, 
  blob1 AS city, 
  SUM(_sample_interval * double2) / SUM(_sample_interval) AS avg_humidity
FROM WEATHER
WHERE
  timestamp >= NOW() - INTERVAL '1' DAY
  AND double1 > 0
GROUP BY t, city
ORDER BY t, avg_humidity DESC
```

This query first rounds the `timestamp` field to the nearest five minutes. Then, it groups by that field and city and calculates the average humidity in each city for a five minute period.

Refer to [Querying Workers Analytics Engine from Grafana](/analytics/analytics-engine/grafana/) for more details on creating efficient Grafana queries.

## Limits

The following limits apply to Analytics Engine:

* Up to 20 blobs, 20 doubles, and 1 index per request.
* All blobs combined must be ≤ 5120 bytes.
* An index can be ≤ 96 bytes.
* At most 25 `writeDataPoint()` calls per incoming HTTP request.

## Data retention

* Data will be stored in Workers Analytics Engine for three months. In the future, we plan to offer longer retention periods.

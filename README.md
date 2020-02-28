# api-express-exporter

api-express-exporter is a prometheus exporter that helps you track express api requests. Plug it in and start monitoring express api requests!

```js
const app = express();

...

// Before all routes
app.use(require("api-express-exporter")()); // That's it!

// Apply your routes
app.get('hello', (req, res) => {
  res.json({ 'hello': 'world!'})
});
```

## Difference between api-express-exporter and other prometheus-api-exporters

api-express-exporter has three goals:
1. Allow you to aggregate metrics by the actual pattern across nested routers.

| request path           | maps to path            |   
|------------------------|-------------------------|
| /api/sub/2/more/3      | /api/sub/:id/more/:id2  |   
| /api/sub/30/more/50/   | /api/sub/:id/more/:id2  |
| /api/error             | /api/error              |
| /api/users/1           | /api/users/:userId      |
| /api/users/100         | /api/users/:userId      |
| /api/users/badId       | /api/users/:userId      |
| /api/users/uuid        | /api/users/:userId      |

`api-express-exporter` will not guess or replace your route params.
It will map them to the original pattern.

It does this by extracting your actual routes with `express-list-endpoints`, and then create `url-pattern` for each route. When a request comes in, `api-express-exporter` will use the UrlPattern objects to retrieve the original route pattern.

This is essential for large applications, if you have tens of thousands of products in a database collection, you do not want separate time series for each product, you want to see them all show up under `/api/products/:productId`.

Losing the original path param names is also error-prone, some frameworks will map both `/api/products/500` and `/api/users/500` to `/api/products/:id` and `/api/users/:id`, making it hard to read at a glance. We want `/api/users/:userId` and `/api/products/:productId`

2. Zero configuration to start with. Use reasonable defaults.
2. Keep things simple. 100 lines total in one file. Easy to fork and update.

## Installation

This is a [Node.js](https://nodejs.org/en/) module available through the
[npm registry](https://www.npmjs.com/).

Before installing, [download and install Node.js](https://nodejs.org/en/download/).
Node.js 0.10 or higher is required.

Installation is done using the
[`npm install` command](https://docs.npmjs.com/getting-started/installing-npm-packages-locally):

```bash
$ npm install api-express-exporter
```

This package depends on `express`, `prom-client`, `express-list-endpoints`, `express-prom-bundle`, and `url-pattern`

## Configuration

| Option Name      | Description  |
|------------------|--------------|
| host             | host string for the metrics server. Defaults to `127.0.0.1`  |
| port             | port that metrics server listens on. Defaults to 9991        |
| urlPatternMaker  | function to create the url pattern matcher, defaults to `(path) => new UrlPattern(path, { segmentNameCharset: "a-zA-Z0-9_-" })`  |
| normalizePath    | boolean. Set this to false to use the original url instead of cleaned up ones. |
| createServer     | boolean. Set this to false to not create the exporter server endpoint |

## Sample Output

When program starts, you will see the following:

```text
Metrics server listening on 127.0.0.1:9991
```

Assuming you have a couple of routes defined like this:

```$js
// maps to /api/sub/:id/more/:id2
app.use("/api/sub", require("./sub_module"));

app.get("/api", (req, res) => {
  res.status(200).send("Api Works.");
});
app.get("/api/fast/", (req, res) => {
  res.status(200).send("Fast response!");
});
app.get("/api/slow", (req, res) => {
  setTimeout(() => {
    res.status(200).send("Slow response...");
  }, 1000);
});

app.get("/api/error", (req, res, next) => {
  try {
    throw new Error("Something broke...");
  } catch (error) {
    res.status(500).send(error.message);
  }
});
```

And you hit the corresponding routes in Postman or Curl. Check console again, you should see:

```$js
$ node src/main.js
Server is running on port 4000
Metrics server listening on 127.0.0.1:9991
Route found: /api/sub/:id
Route found: /api/sub/:id/more/:id2
Route found: /api
Route found: /api/fast
Route found: /api/slow
Route found: /api/error
Route found: /api/list/:listId
```

Navigate to `127.0.0.1:9991/metrics`, you will see:

```text

# HELP http_request_duration_seconds duration histogram of http responses labeled with: status_code, method, path
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.03",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="0.3",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="1",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="1.5",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="3",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="5",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="10",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="+Inf",status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_sum{status_code="500",method="GET",path="/api/error"} 0.008067119
http_request_duration_seconds_count{status_code="500",method="GET",path="/api/error"} 1
http_request_duration_seconds_bucket{le="0.03",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="0.3",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="1",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="1.5",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="3",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="5",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="10",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_bucket{le="+Inf",status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1
http_request_duration_seconds_sum{status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 0.001186646
http_request_duration_seconds_count{status_code="200",method="GET",path="/api/sub/:id/more/:id2"} 1

# HELP up 1 = up, 0 = not up
# TYPE up gauge
up 1
```

To see how to visualize the data in prometheus+grafana, you can check out [Node.js Monitoring with Prometheus+Grafana](https://medium.com/teamzerolabs/node-js-monitoring-with-prometheus-grafana-3056362ccb80)

Happy monitoring!

## License

  [MIT](LICENSE)
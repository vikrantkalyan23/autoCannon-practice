# Autocannon practice

Autocannon generates HTTP/1.1 traffic and measures how a server behaves under load. It runs from the terminal or inside a Node.js program and supports HTTPS and HTTP pipelining. The server being tested can use any language.

Use it to establish an API performance baseline, compare implementations, investigate slow endpoints, or check whether a change reduces throughput. It does not render a page or execute browser JavaScript, so API latency is different from page loading speed. See the [official package documentation](https://github.com/mcollina/autocannon#readme).

## Installation

Install Node.js and npm first. Use a supported Node.js version compatible with the package you install.

```bash
node --version
npm --version

# Run without a global installation (npm may download the package).
npx autocannon --help

# Or install a terminal command globally.
npm install --global autocannon
autocannon --version

# For a project's benchmark scripts; run inside an npm project.
npm install --save-dev autocannon
```

The examples below use `npx autocannon`. With a global installation, you can use `autocannon` instead. For repeatable results, keep the project dependency and lockfile under version control and record the package version.

## Start a local practice server

Save this as `server.cjs`:

```javascript
const http = require('node:http');

http.createServer((req, res) => {
  // Consume incoming bodies so POST examples can reuse connections.
  req.resume();
  req.on('end', () => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ ok: true }));
  });
}).listen(3000, '127.0.0.1', () => {
  console.log('Listening on http://127.0.0.1:3000');
});
```

Run it in one terminal:

```bash
node server.cjs
```

Run benchmarks in another terminal. This server returns the same response for every path; the examples demonstrate traffic generation rather than a complete application.

## Basic commands

```bash
# Default run: 10 connections, 10 seconds, GET, pipelining 1.
npx autocannon http://127.0.0.1:3000

# One connection: start with a small workload.
npx autocannon -c 1 -d 5 http://127.0.0.1:3000

# 50 connections for 30 seconds.
npx autocannon -c 50 -d 30 http://127.0.0.1:3000

# Complete a request count instead of a timed run.
npx autocannon -c 10 -a 1000 http://127.0.0.1:3000
```

Connections are open client connections, not simulated human users. Without a rate limit, the client tries to send requests as quickly as the configuration permits.

## CLI attributes / options

Option names are case sensitive: `-r` and `-R` have different scopes. This is a practical reference, not an exhaustive list. Check `npx autocannon --help` for your installed release. The [CLI implementation](https://github.com/mcollina/autocannon/blob/master/autocannon.js) defines aliases and defaults.

| Option | Meaning | Example value |
| --- | --- | --- |
| `-c`, `--connections` | Concurrent connections; default 10 | `-c 50` |
| `-d`, `--duration` | Runtime in seconds; default 10 | `-d 30` |
| `-a`, `--amount` | Request count; overrides duration | `-a 1000` |
| `-p`, `--pipelining` | Outstanding requests per connection; default 1 | `-p 5` |
| `-m`, `--method` | HTTP method; default GET | `-m POST` |
| `-H`, `--headers` | Request header; repeat for multiple headers | `-H 'Accept=application/json'` |
| `-b`, `--body` | Inline request payload | `-b '{"name":"Ada"}'` |
| `-i`, `--input` | Read payload from a file | `-i payload.json` |
| `-t`, `--timeout` | Connection timeout in seconds; default 10 | `-t 5` |
| `-r`, `--connectionRate` | Requests/second limit per connection | `-r 10` |
| `-R`, `--overallRate` | Requests/second limit across connections in one worker | `-R 200` |
| `-w`, `--workers` | Load generator worker threads | `-w 2` |
| `-W`, `--warmup` | Warmup configuration | `-W [ -c 2 -d 5 ]` |
| `-l`, `--latency` | Extra latency percentiles | `-l` |
| `-j`, `--json` | Machine readable JSON output | `-j` |
| `-T`, `--title` | Label the result | `-T baseline` |
| `--renderStatusCodes` | Show response status breakdown | `--renderStatusCodes` |
| `--debug` | Print connection errors | `--debug` |
| `-h`, `--help` | Show available flags | `--help` |

## Intermediate examples

### Headers and authentication

```bash
# Replace the token with a test credential for your own API.
npx autocannon -c 10 -d 15 \
  -H 'Authorization=Bearer YOUR_TEST_TOKEN' \
  -H 'Accept=application/json' \
  http://127.0.0.1:3000/api/profile
```

### Send JSON with POST

```bash
npx autocannon -c 10 -d 15 -m POST \
  -H 'Content-Type=application/json' \
  -b '{"name":"Ada","role":"tester"}' \
  http://127.0.0.1:3000/api/users
```

For a larger payload, save JSON to `payload.json` and replace `-b` with `-i`:

```bash
npx autocannon -c 10 -a 500 -m POST \
  -H 'Content-Type=application/json' \
  -i payload.json http://127.0.0.1:3000/api/users
```

### Limit traffic and inspect responses

```bash
npx autocannon -c 20 -d 30 -R 200 -l --renderStatusCodes \
  http://127.0.0.1:3000
```

Treat 200 requests/second as the requested limit, not a promise of achieved throughput. Check what the run actually delivered. If both rate options are present, `connectionRate` takes precedence. Rate limited runs apply coordinated omission correction by default: this accounts for latency samples missed when the system cannot keep up. Keep that correction enabled when comparing such runs. See the [rate options documentation](https://github.com/mcollina/autocannon#usage).

### Save a report

```bash
npx autocannon -c 20 -d 15 -T baseline -j \
  http://127.0.0.1:3000 > baseline.json
```

JSON output is suitable for later analysis; it does not itself enforce performance thresholds.

## Advanced examples

### Warm up before measuring

```bash
npx autocannon -c 40 -d 30 --warmup [ -c 5 -d 5 ] \
  http://127.0.0.1:3000
```

Warmup helps reduce startup effects such as just-in-time compilation and cold caches. Decide separately whether cold startup performance matters for your application.

### HTTP/1.1 pipelining

```bash
npx autocannon -c 20 -p 5 -d 20 http://127.0.0.1:3000
```

This configuration permits roughly 100 requests in flight across 20 connections. Pipelining differs from HTTP/2 multiplexing; use it only when it represents the clients and infrastructure you want to measure.

### Use multiple CPU cores to generate load

```bash
npx autocannon -w 2 -c 40 -R 100 -d 30 http://127.0.0.1:3000
```

Here, 40 connections are divided between two workers, while the rate limit is 100 per worker: approximately 200 requests/second total if achievable. `amount` is also divided between workers; choose values divisible by the worker count to avoid rounding surprises. `overallRate` and `maxOverallRequests` apply separately to each worker. See [worker semantics](https://github.com/mcollina/autocannon#workers).

### Increase load in stages

```bash
for connections in 5 10 25 50; do
  npx autocannon -c "$connections" -d 20 -j \
    http://127.0.0.1:3000 > "result-${connections}.json"
done
```

These are separate runs. Compare their latency and errors alongside throughput to find where added concurrency stops helping.

### HTTPS with a client certificate

For an existing test server configured for mutual TLS:

```bash
npx autocannon -c 10 -d 15 \
  --cert ./client-cert.pem --key ./client-key.pem --ca ./ca.pem \
  https://localhost:3443
```

The local practice server above serves plain HTTP and cannot run this example. The [CLI TLS handling](https://github.com/mcollina/autocannon/blob/master/autocannon.js) loads certificate files from these paths.

## Use Autocannon from Node.js

After installing the project dependency, save this as `benchmark.cjs`. The `.cjs` extension allows CommonJS syntax even in a project configured for ES modules.

```javascript
const autocannon = require('autocannon');

async function main() {
  const result = await autocannon({
    url: 'http://127.0.0.1:3000',
    connections: 10,
    duration: 10,
    pipelining: 1,
    headers: { accept: 'application/json' }
  });

  console.log(JSON.stringify({
    requestsPerSecond: result.requests.average,
    p99Milliseconds: result.latency.p99,
    errors: result.errors,
    timeouts: result.timeouts,
    non2xx: result.non2xx
  }, null, 2));

  // Illustrative CI policy: adjust these limits for your environment.
  if (result.errors > 0 || result.timeouts > 0 ||
      result.non2xx > 0 || result.latency.p99 > 200) {
    process.exitCode = 1;
  }
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

```bash
node benchmark.cjs
```

CLI names generally map to JavaScript attributes, such as `-c` → `connections`, `-d` → `duration`, and `-R` → `overallRate`. Headers become an object and the payload uses `body`. Use `requests` for request sequences and hooks for dynamic request setup; see the [API reference](https://github.com/mcollina/autocannon#api). The [result aggregation implementation](https://github.com/mcollina/autocannon/blob/master/lib/aggregateResult.js) shows the returned metric fields.

## Understand the results

| Metric | How to read it |
| --- | --- |
| Latency | Response time in milliseconds; lower is usually better at equivalent load |
| p50 | Median response time |
| p99 | Approximately 99% of recorded latency observations are at or below this value |
| Req/Sec | Request throughput; judge it together with response status and errors |
| Bytes/Sec | Data throughput, influenced by response sizes |
| Errors / timeouts | Transport problems or requests that exceeded the timeout |
| Non-2xx / status codes | Responses such as redirects, authentication failures, or server failures |

These fields come from the [run counters](https://github.com/mcollina/autocannon/blob/master/lib/run.js) and [aggregated results](https://github.com/mcollina/autocannon/blob/master/lib/aggregateResult.js). A fast stream of HTTP 500 responses is not a successful capacity result. A low average can also hide slow tail responses; inspect p99.

For useful comparisons, keep the endpoint, payload, dataset, rate, connections, duration, and software versions consistent. Repeat runs to assess normal variation. Monitor server CPU, memory, database activity, and the load generator itself. Running both processes on one laptop is convenient for practice, but they compete for resources; use separate machines for capacity measurements.

## Similar packages and software

The recommendations below are practical choices based on each tool's documented features, not a ranking of benchmark speed.

| Tool | Interface / ecosystem | Useful when | Tradeoff compared with Autocannon |
| --- | --- | --- | --- |
| [Autocannon](https://github.com/mcollina/autocannon) | Node.js API and CLI | You want quick HTTP/1.1 endpoint benchmarks in a Node project | Complex user journeys need custom scripting |
| [wrk](https://github.com/wg/wrk) | Native CLI, Lua scripting | You want a multithreaded HTTP benchmark with Lua customization | Less direct integration with Node.js code |
| [wrk2](https://github.com/giltene/wrk2) | A wrk variant with constant throughput measurement | You want to study latency at a configured request rate | More specialized than a scenario testing framework |
| [Vegeta](https://github.com/tsenart/vegeta) | Go library and CLI | You want rate driven HTTP tests and command line reporting pipelines | Uses a different scripting/library ecosystem |
| [k6](https://github.com/grafana/k6) | Go engine with JavaScript test scripts | You want scripted workloads and [pass/fail thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/) | Runs its own runtime rather than acting as an npm library |
| [Artillery](https://www.artillery.io/docs) | YAML/JavaScript scenarios | You need phased scenarios, WebSockets, or Playwright based workloads | More configuration than a single endpoint benchmark |
| [Locust](https://docs.locust.io/en/stable/what-is-locust.html) | Python scenarios, web UI and CLI | You prefer Python and distributed simulated users | Requires Python test code and its execution environment |
| [Apache JMeter](https://jmeter.apache.org/) | Java, graphical test plan editor and CLI | You need broad protocol support such as JDBC or JMS and HTML reports | Larger setup; create/debug plans in the GUI and generate load in CLI mode |

Start with Autocannon for this repository's endpoint practice. Consider k6 or Artillery for multi-step workload suites, Locust for Python-based user behavior, and JMeter for broader protocol coverage. Do not compare headline request rates from different tools without matching workload and measurement settings.

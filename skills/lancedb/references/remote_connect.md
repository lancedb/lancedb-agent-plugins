# Connecting to a LanceDB remote server

LanceDB Enterprise/Cloud deployments are served by a server implementing the
lance-namespace OpenAPI spec
(<https://github.com/lance-format/lance-namespace/blob/main/docs/src/spec.yaml>).
Every remote (`db://...`) connection talks to such a server, and some operations
exist only there. In particular, all operations around jobs (listing, inspecting,
creating, or canceling jobs) run server-side — there is no local/OSS equivalent, so
resolve a server connection before attempting any job work. The job REST methods
themselves are documented in `references/remote_jobs.md`.

Every request needs two things:

1. **Base URL** — the server endpoint
2. **Credentials** — an API key (`x-api-key` header over REST), and usually a database name (`x-lancedb-database` header)

## Resolution steps

1. If the user already gave a URL and API key (or said which environment they're working against), use that.
2. Otherwise, look for credentials already available in the environment:
   - Env vars like `LANCEDB_URI` / `LANCEDB_HOST` / `LANCEDB_API_KEY`
   - A server endpoint already running or port-forwarded locally (the REST default port is 2333, i.e. `http://localhost:2333`)
3. If you didn't find both pieces, ask the user directly: **"What's your LanceDB endpoint's URL, and what's your API key?"** Also ask which database to use if it isn't obvious. Don't guess or probe further — the user knows their deployment.

## Validating the connection

Make a cheap authenticated request and check the status before starting real work:

```bash
curl -s -w "\n%{http_code}" "{base_url}/v1/table/?limit=1" \
  -H "x-api-key: <key>" \
  -H "x-lancedb-database: <database>"
```

- `200` — connection, key, and database header all good
- `401` — API key missing or wrong
- `400` mentioning a database header — this deployment expects `x-lancedb-database`

## Non-REST equivalents

The same credentials work through the SDKs and CLI:

- Python SDK: `lancedb.connect("db://<database>", api_key="<key>", host_override="<base_url>")`
- TypeScript SDK: `await lancedb.connect("db://<database>", { apiKey: "<key>", hostOverride: "<base_url>" })`
- `lancedb` CLI: a `[profiles.<name>]` entry in `~/.lancedb/config.toml` with `http_server_url`, `api_key`, `database`

## `host_override` / `hostOverride` must be a full URL: scheme *and* port, no defaults

The SDK client uses `host_override` verbatim as the request prefix — it does not
inject a scheme or a port. A bare hostname, or a URL missing the port, fails
silently at the network layer rather than raising a clear config error:

- **Missing scheme** (e.g. `host_override="my-service.svc.cluster.local"`): the
  client concatenates this directly in front of each request path
  (`{host_override}/v1/table/...`), producing an unparseable URL. This fails
  immediately on the *first* real call (e.g. `list_tables()`, not `connect()`,
  which is lazy) with a low-level URL-parse/builder error — not a connection or
  auth error.
- **Missing port** (e.g. `host_override="http://my-service.svc.cluster.local"`
  with no `:<port>`): the client falls back to the scheme's standard port (80
  for `http://`, 443 for `https://`), which is almost never where a LanceDB
  server listens. This surfaces as a `RetryError` with all failures counted as
  `connect_failures` (`request_failures=0`, `read_failures=0`) — the client
  never gets far enough to send a request or read a response. A health-check
  `curl` to the *correct* port succeeding while the SDK still hits
  `connect_failures=N/N` is the signature of this — the service and DNS are
  fine, only the client's target port is wrong.
- **Don't assume the local dev server's port generalizes.** The `lancedb
  server` local dev default is port `2333` (see "Resolution steps" above), but
  Enterprise/Cloud query nodes commonly default to a different port (`10024`).
  Always get the actual port from the deployment (Helm values, `kubectl get
  svc`, or the person who runs it) rather than assuming either default.

Always pass `host_override` as a complete `scheme://host:port` value, e.g.
`http://lancedb-query-node:10024`, and validate it with the `curl` command
above (same host, port, and headers the SDK will use) before debugging further
up the stack.

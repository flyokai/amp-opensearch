# flyokai/amp-opensearch

> User docs → [`README.md`](README.md) · Agent quick-ref → [`CLAUDE.md`](CLAUDE.md) · Agent deep dive → [`AGENTS.md`](AGENTS.md)

> Async OpenSearch client for PHP — bridges the official `opensearch-project/opensearch-php` SDK with AMPHP's non-blocking HTTP client.

Drops in as a custom request handler so you can keep using the official SDK API while every call is non-blocking under Revolt.

## Features

- `AmpHandler` — callable handler for `OpenSearch\ClientBuilder::setHandler()`
- `HttpClientBuilder` — configured AMPHP `HttpClient` with connection pooling
- Configurable retry (limit + delay)
- Returns `Guzzle\CompletedFutureArray` (the format the SDK expects)

## Installation

```bash
composer require flyokai/amp-opensearch
```

## Quick start

```php
use Flyokai\AmpOpensearch\AmpHandler;
use Flyokai\AmpOpensearch\HttpClientBuilder;
use OpenSearch\ClientBuilder;

$handler = new AmpHandler(
    (new HttpClientBuilder())->build(),
    retryLimit: 3,
    retryDelay: 0.1,
);

$client = (new ClientBuilder())
    ->setHandler($handler)
    ->setHosts(['https://localhost:9200'])
    ->setBasicAuthentication('admin', 'admin')
    ->setSSLVerification(false)
    ->build();

$client->index([
    'index' => 'products',
    'id'    => 1,
    'body'  => ['name' => 'Foo', 'price' => 9.99],
]);

$result = $client->search([
    'index' => 'products',
    'body'  => ['query' => ['match' => ['name' => 'Foo']]],
]);
```

Every call inside fibers is fully non-blocking — the SDK doesn't know.

## Architecture

`AmpHandler::__invoke(array $request)` is invoked by the OpenSearch SDK for every operation. It:

1. Translates the SDK request array into an AMPHP `Request`
2. Executes via the configured `HttpClient`
3. Wraps the response as `CompletedFutureArray` with: `status`, `reason`, `headers`, `body` (`php://memory`), `effective_url`, `transfer_stats`, `error`
4. Retries up to `retryLimit` times with `retryDelay` between attempts on failure

`HttpClientBuilder::build()` returns an AMPHP `HttpClient` with:

- TLS without peer verification (development default)
- `UnlimitedConnectionPool`
- `DefaultConnectionFactory` + `ConnectContext`

## Gotchas

- **SSL verification disabled by default.** `HttpClientBuilder` calls `withoutPeerVerification()`. Review for production.
- **Hard-coded JSON headers.** `Content-Type` and `Accept` are always `application/json`. Not customizable per request.
- **Memory stream for responses** — `php://memory`. No streaming.
- **`transfer_stats` is incomplete** — `total_time` is hardcoded to 0.
- **Requires Revolt event loop context** — retry delays use `EventLoop::delay()` + suspension. Will fail outside fibers.

## See also

- [`flyokai/indexer`](../indexer/README.md) — uses this for indexing operations.

## License

MIT

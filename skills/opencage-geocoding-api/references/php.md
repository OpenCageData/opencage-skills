# OpenCage Geocoding API — PHP

This skill covers PHP-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api/SKILL.md** first.

This reference documents the **`opencage/geocode` PHP SDK v4.0.0**.

## Installation

```bash
composer require opencage/geocode
```

The `opencage/geocode` package is available on Packagist and is the official PHP client maintained by OpenCage. It requires **PHP 8.2 or newer** and uses [Guzzle](https://docs.guzzlephp.org/) for HTTP, which Composer pulls in as a dependency — no extra configuration is required.

```php
require 'vendor/autoload.php';
```

## Basic Usage

```php
<?php
require 'vendor/autoload.php';

$geocoder = new \OpenCage\Geocoder\Geocoder('YOUR-API-KEY');

// Forward geocoding (address → coordinates)
$result = $geocoder->geocode('82 Clerkenwell Road, London');
if ($result['total_results'] > 0) {
    $first = $result['results'][0];
    echo $first['geometry']['lat'] . ', ' . $first['geometry']['lng'] . PHP_EOL;
    echo $first['formatted'] . PHP_EOL;
}
```

`geocode()` returns the raw API response decoded into a PHP associative array. The top-level keys match those described in **opencage-geocoding-api/SKILL.md** (`status`, `total_results`, `results`, `rate`, etc.). Always check `$result['total_results'] > 0` before accessing `$result['results'][0]`.

## Reverse Geocoding (coordinates → address)

Use the dedicated `geocodeReverse()` method, which accepts the latitude and longitude as separate arguments (each may be a `float` or a `string`):

```php
$result = $geocoder->geocodeReverse(43.831, 4.360);  // latitude, longitude
if ($result['total_results'] > 0) {
    echo $result['results'][0]['formatted'] . PHP_EOL;
    // 3 Rue de Rivarol, 30020 Nîmes, France
}
```

Internally this simply forms the `"lat,lng"` query for you, so passing a coordinate string to `geocode()` still works, but `geocodeReverse()` is clearer and is the recommended approach in 4.0.0.

## Passing Optional Parameters

Optional API parameters (documented in opencage-geocoding-api/SKILL.md) are passed as an associative array. The array is typed `array<string, string>`, so use string values:

```php
$result = $geocoder->geocode('Brandenburg Gate', [
    'countrycode'    => 'de',
    'language'       => 'de',
    'limit'          => '1',
    'no_annotations' => '1',
]);
```

The same optional-parameters array is accepted as the final argument of `geocodeReverse()`, `geocodeAsync()`, and `geocodeReverseAsync()`.

**Always pass `'no_annotations' => '1'`** when you only need coordinates or the `formatted` address — it reduces response size and latency.

## Asynchronous Geocoding

New in 4.0.0: `geocodeAsync()` and `geocodeReverseAsync()` return a Guzzle `PromiseInterface` instead of a result array. This lets you issue several requests concurrently.

```php
// Single async request — call wait() to resolve it to the result array
$promise = $geocoder->geocodeAsync('82 Clerkenwell Road, London');
$result  = $promise->wait();
print_r($result);

// Concurrent requests — fire them all, then unwrap together
$promises = [
    'london' => $geocoder->geocodeAsync('London'),
    'paris'  => $geocoder->geocodeAsync('Paris'),
    'tokyo'  => $geocoder->geocodeAsync('Tokyo'),
];
$results = \GuzzleHttp\Promise\Utils::unwrap($promises);
echo $results['london']['results'][0]['formatted'] . PHP_EOL;
echo $results['paris']['results'][0]['formatted'] . PHP_EOL;
echo $results['tokyo']['results'][0]['formatted'] . PHP_EOL;

// Async reverse geocoding
$promise = $geocoder->geocodeReverseAsync(43.831, 4.360);
$result  = $promise->wait();
echo $result['results'][0]['formatted'] . PHP_EOL;
```

Each resolved promise yields the same associative-array structure as the synchronous methods. Concurrency is convenient on **paid** plans; on a free-trial account (1 req/s) prefer the rate-limited sequential pattern shown under *Batch Geocoding*.

## Handling the Response

In 4.0.0 `geocode()` (and `geocodeReverse()`) **always returns an array** when a response is received, and **throws an `\Exception`** for a missing API key or an undecodable response — it never returns `null`. Network problems (DNS, TLS, unreachable host) are not thrown; they come back as a normal array with `status.code` of `498`.

```php
try {
    $result = $geocoder->geocode('Berlin, Germany');
} catch (\Exception $e) {
    // e.g. "Missing API key" or "Failed to decode API response"
    echo 'Request error: ' . $e->getMessage() . PHP_EOL;
    return;
}

$code = $result['status']['code'];

if ($code === 498) {
    echo 'Network issue: ' . $result['status']['message'] . PHP_EOL;
} elseif ($code !== 200) {
    echo 'API error: ' . $result['status']['message'] . PHP_EOL;
} elseif ($result['total_results'] === 0) {
    echo 'No results found' . PHP_EOL;
} else {
    echo $result['results'][0]['formatted'] . PHP_EOL;
}
```

Common status codes are documented in **opencage-geocoding-api/SKILL.md**. Key ones to handle:

- `200` — OK
- `401` — Invalid or missing API key
- `402` — Quota exceeded
- `403` — Key suspended, disabled, or IP rejected
- `429` — Too many requests (per-second rate limit hit)
- `498` — Network issue reaching the API (SDK-generated, not from the server)

Note that `geocodeAsync()`/`geocodeReverseAsync()` also throw the "Missing API key" exception synchronously, before the promise is returned, so wrap the call (not just the `wait()`) in your `try`/`catch`.

## Accessing Components Safely

`components` fields are not guaranteed to be present for every location (see opencage-geocoding-api/SKILL.md — "Results Reflect the Real World"). Use `isset()` or the null coalescing operator:

```php
$components = $result['results'][0]['components'] ?? [];

$country  = $components['country'] ?? null;
$city     = $components['city'] ?? $components['town'] ?? $components['village'] ?? null;
$postcode = $components['postcode'] ?? null;
$type     = $components['_type'] ?? null;
```

## Batch Geocoding

For processing a list of addresses on a free-trial account, add a short `sleep` between requests to respect the 1 req/s rate limit:

```php
<?php
require 'vendor/autoload.php';

$geocoder = new \OpenCage\Geocoder\Geocoder(getenv('OPENCAGE_API_KEY'));

$addresses = ['Berlin, Germany', 'Paris, France', 'London, UK'];
$output = [];

foreach ($addresses as $address) {
    try {
        $result = $geocoder->geocode($address, ['no_annotations' => '1', 'limit' => '1']);
    } catch (\Exception $e) {
        $output[] = ['input' => $address, 'error' => $e->getMessage()];
        continue;
    }

    $code = $result['status']['code'];

    if ($code === 200 && $result['total_results'] > 0) {
        $first = $result['results'][0];
        $output[] = [
            'input'     => $address,
            'lat'       => $first['geometry']['lat'],
            'lng'       => $first['geometry']['lng'],
            'formatted' => $first['formatted'],
        ];
    } elseif ($code === 429) {
        echo "Rate limit hit — pausing\n";
        sleep(2);
    } else {
        $output[] = ['input' => $address, 'error' => "status $code"];
    }

    sleep(1);  // 1 req/s for free-trial accounts; remove for paid subscriptions
}

print_r($output);
```

On a paid plan you can instead issue the batch concurrently with `geocodeAsync()` and `\GuzzleHttp\Promise\Utils::unwrap()` (see *Asynchronous Geocoding*).

## Configuration

The constructor takes the API key; the following setters tune the underlying Guzzle client. Calling any of them rebuilds the client on the next request.

### Timeout

```php
$geocoder->setTimeout(5);  // seconds; default is 10
```

### Proxy

The proxy URL must include a scheme (`http`, `https`, or `socks5`) and a host, otherwise `setProxy()` throws an `\Exception`:

```php
$geocoder->setProxy('https://proxy.example.com:1234');
```

### Host

`setHost()` accepts only `localhost` (optionally with a port) or an `opencagedata.com` subdomain; anything else throws an `\Exception`. This is mainly useful for testing.

```php
$geocoder->setHost('localhost:8080');
```

## Environment Variables

Never hard-code your API key. Read it from an environment variable at runtime:

```php
$geocoder = new \OpenCage\Geocoder\Geocoder(getenv('OPENCAGE_API_KEY'));
```

In development, use a `.env` file with the `vlucas/phpdotenv` package:

```php
$dotenv = Dotenv\Dotenv::createImmutable(__DIR__);
$dotenv->load();

$geocoder = new \OpenCage\Geocoder\Geocoder($_ENV['OPENCAGE_API_KEY']);
```

## Further Reading

- OpenCage PHP tutorial: https://opencagedata.com/tutorials/geocode-in-php
- `opencage/geocode` on Packagist: https://packagist.org/packages/opencage/geocode
- Source code (v4.0.0): https://github.com/OpenCageData/php-opencage-geocode
- Guzzle documentation: https://docs.guzzlephp.org/
- General API reference: **opencage-geocoding-api/SKILL.md**



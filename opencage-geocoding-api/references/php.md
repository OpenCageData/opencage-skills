# OpenCage Geocoding API — PHP

This skill covers PHP-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api.md** first.

## Installation

```bash
composer require opencage/geocode
```

The `opencage/geocode` package is available on Packagist and is the official PHP client maintained by OpenCage. It requires PHP 8+ and uses the cURL extension (falls back to `fopen` wrappers if cURL is unavailable — ensure `allow_url_fopen` is enabled in `php.ini` if you rely on the fallback).

## Basic Usage

```php
<?php
require 'vendor/autoload.php';

$geocoder = new \OpenCage\Geocoder\Geocoder('YOUR-API-KEY');

// Forward geocoding (address → coordinates)
$result = $geocoder->geocode('82 Clerkenwell Road, London');
if ($result && $result['total_results'] > 0) {
    $first = $result['results'][0];
    echo $first['geometry']['lat'] . ', ' . $first['geometry']['lng'] . PHP_EOL;
    echo $first['formatted'] . PHP_EOL;
}

// Reverse geocoding (coordinates → address)
$result = $geocoder->geocode('43.831,4.360');  // lat,lng as a string
if ($result && $result['total_results'] > 0) {
    echo $result['results'][0]['formatted'] . PHP_EOL;
}
```

The library returns the raw API response as a PHP associative array. The top-level keys match those described in **opencage-geocoding-api.md** (`status`, `total_results`, `results`, `rate`, etc.). Always check `$result['total_results'] > 0` before accessing `$result['results'][0]`.

## Passing Optional Parameters

Optional API parameters (documented in opencage-geocoding-api.md) are passed as an associative array in the second argument:

```php
$result = $geocoder->geocode('Brandenburg Gate', [
    'countrycode' => 'de',
    'language'    => 'de',
    'limit'       => 1,
    'no_annotations' => 1,
]);
```

**Always pass `'no_annotations' => 1`** when you only need coordinates or the `formatted` address — it reduces response size and latency.

## Checking the Response Status

The response always contains a `status` key with a `code` and `message`. Check these before trusting the results:

```php
$result = $geocoder->geocode('Berlin, Germany');

if ($result === null) {
    echo 'Request failed (network or configuration error)' . PHP_EOL;
} elseif ($result['status']['code'] !== 200) {
    echo 'API error: ' . $result['status']['message'] . PHP_EOL;
} elseif ($result['total_results'] === 0) {
    echo 'No results found' . PHP_EOL;
} else {
    $first = $result['results'][0];
    echo $first['formatted'] . PHP_EOL;
}
```

Common status codes and their meanings are documented in **opencage-geocoding-api.md**. Key ones to handle:

- `200` — OK
- `401` — Invalid or missing API key
- `402` — Quota exceeded
- `403` — Key suspended
- `429` — Too many requests (per-second rate limit hit)

## Accessing Components Safely

`components` fields are not guaranteed to be present for every location (see opencage-geocoding-api.md — "Results Reflect the Real World"). Use `isset()` or the null coalescing operator:

```php
$components = $result['results'][0]['components'] ?? [];

$country  = $components['country'] ?? null;
$city     = $components['city'] ?? $components['town'] ?? $components['village'] ?? null;
$postcode = $components['postcode'] ?? null;
$type     = $components['_type'] ?? null;
```

## Batch Geocoding

For processing a list of addresses, add a short `sleep` between requests to respect the 1 req/s rate limit on free-trial accounts:

```php
<?php
require 'vendor/autoload.php';

$geocoder = new \OpenCage\Geocoder\Geocoder(getenv('OPENCAGE_API_KEY'));

$addresses = ['Berlin, Germany', 'Paris, France', 'London, UK'];
$output = [];

foreach ($addresses as $address) {
    $result = $geocoder->geocode($address, ['no_annotations' => 1, 'limit' => 1]);

    if ($result && $result['status']['code'] === 200 && $result['total_results'] > 0) {
        $first = $result['results'][0];
        $output[] = [
            'input'     => $address,
            'lat'       => $first['geometry']['lat'],
            'lng'       => $first['geometry']['lng'],
            'formatted' => $first['formatted'],
        ];
    } elseif ($result && $result['status']['code'] === 429) {
        echo "Rate limit hit — pausing\n";
        sleep(2);
    } else {
        $code = $result['status']['code'] ?? 'null';
        $output[] = ['input' => $address, 'error' => "status $code"];
    }

    sleep(1);  // 1 req/s for free-trial accounts; remove for paid subscriptions
}

print_r($output);
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
- Source code: https://github.com/OpenCageData/php-opencage-geocode
- General API reference: **opencage-geocoding-api.md**

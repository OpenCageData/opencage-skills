# OpenCage Geocoding API — Parameters, Annotations, Confidence & Rate Limits

This file contains detailed reference tables for the OpenCage Geocoding API. For general concepts, response structure, error handling, and common mistakes, refer to **opencage-geocoding-api/SKILL.md** first.

## Optional API Parameters

The OpenCage Geocoding API can be called with various optional parameters:

| Parameter | Brief Description | Example Usage |
|-----------|-------------------|---------------|
| `abbrv` | Abbreviates and shortens the `formatted` string in the response | `abbrv=1` |
| `address_only` | Excludes POI names from the `formatted` string, returning only the address | `address_only=1` |
| `add_request` | Includes the original request parameters in the response (useful for debugging) | `add_request=1` |
| `bounds` | Forward geocoding only. Restricts results to a bounding box defined by SW and NE corners (min lon, min lat, max lon, max lat) | `bounds=-0.563160,51.280430,0.278970,51.683979` |
| `countrycode` | Forward geocoding only. Restricts results to one or more countries via ISO 3166-1 Alpha-2 codes | `countrycode=de` or `countrycode=ca,us` |
| `jsonp` | Wraps the returned JSON with a function name (JSONP callback) | `jsonp=myfuncname` |
| `language` | Specifies preferred response language using an IETF language tag, or `native` for local language | `language=de` |
| `limit` | Forward geocoding only. Sets the maximum number of results returned (default: 10, max: 100) | `limit=1` |
| `no_annotations` | Suppresses the `annotations` block from results to reduce response size and improve speed | `no_annotations=1` |
| `no_dedupe` | Disables deduplication of results | `no_dedupe=1` |
| `no_record` | Prevents the query from being logged (for privacy-sensitive use cases) | `no_record=1` |
| `pretty` | Pretty-prints the JSON response for easier reading and debugging | `pretty=1` |
| `proximity` | Forward geocoding only. Biases results toward the specified lat/lon coordinate | `proximity=52.5432379,13.4142133` |
| `roadinfo` | Changes geocoder behaviour to snap to the nearest road and returns extended road/driving info in annotations | `roadinfo=1` |

## Annotations

By default every result includes an `annotations` object with supplementary data derived from the result's coordinates (unless `no_annotations=1` is passed). Annotations are **not** properties of the input query — they describe what is true at the returned location.

| Annotation | Description |
|------------|-------------|
| `timezone` | Timezone name, UTC offset |
| `currency` | Local currency name, symbol, ISO code |
| `callingcode` | International dialling code |
| `flag` | Country emoji flag |
| `geohash` | Geohash of the result centre point |
| `DMS` | Coordinates in degree/minute/second format |
| `Mercator` | Mercator projection coordinates |
| `MGRS` | Military Grid Reference System code |
| `OSM` | OpenStreetMap view/edit URLs |
| `wikidata` | Wikidata item identifier |
| `what3words` | Three-word address |
| `sun` | Sunrise and sunset times |
| `qibla` | Direction to Mecca |
| `FIPS` | US Federal Information Processing Standards codes |
| `NUTS` | EU territorial unit codes |
| `UN_M49` | UN regional and statistical codes |

Some annotations are only available to paying customers and must be enabled on request (e.g. `H3`, `UN/LOCODE`). Road-specific data requires passing `roadinfo=1`.

**Annotations vary by location.** Not every annotation is available everywhere. `FIPS` codes only exist in the United States. `NUTS` codes only exist in the EU. `qibla` is meaningless at the poles. `roadinfo` requires road data to exist for that area.

## Confidence Score

The confidence score (0–10) measures the **size of the bounding box** of the matched place — it is **not** a measure of correctness or relevance. A high score means the result is geographically small (e.g. a building); a low score means it is geographically large (e.g. a country). The API can be completely certain it found the right place and still return a low confidence score.

For example, geocoding `"Berlin, Germany"` returns a confidence of **2**, because Berlin is a large city — its bounding box spans many kilometres. That low score doesn't mean the result is wrong; it means the matched place is large. By contrast, geocoding a specific street address in Berlin might return a confidence of **9** or **10**.

| Score | Implied bounding box size |
|-------|--------------------------|
| 10 | < 0.25 km (building/exact address) |
| 9 | < 0.5 km (street level) |
| 7–8 | < 1–5 km (neighborhood/district) |
| 4–6 | City or region level |
| 1–3 | Country or large area |
| 0 | No bounding box available |

Do not use the confidence score for ranking — results are already ordered by relevance. Use it to filter out results that are too coarse for your use case.

To understand *what type of place* was matched, check `components["_type"]` instead of relying on the confidence score. Common values include `"city"`, `"town"`, `"village"`, `"road"`, `"building"`, `"postcode"`, `"country"`, `"state"`, and `"neighbourhood"`.

## API Rate Limits

| Plan | Daily quota | `rate` in response |
|------|-------------|-------------------|
| Free trial | 2,500 req/day | Yes |
| One-time purchase | Fixed quota | Yes |
| Subscription | No hard limit | No |

The `rate` object is only present in responses for free-trial and one-time purchase customers, whose quotas are enforced. Subscription customers have no hard limits, so their responses do not include a `rate` field — do not assume it will always be present.

```json
{
  "rate": {
    "limit": 2500,
    "remaining": 2498,
    "reset": 1605312000
  },
  "results": [...],
  "status": {
    "code": 200,
    "message": "OK"
  },
  "total_results": 1
}
```

For `402` errors, free-trial and one-time purchase customers can check `rate.remaining` in the response to monitor quota before hitting the limit.

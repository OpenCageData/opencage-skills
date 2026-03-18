# OpenCage Geocoding API — Java

This skill covers Java-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api.md** first.

## Installation

The official Java client is **jopencage**, maintained by OpenCage at `com.opencagedata`.

### Maven

```xml
<dependency>
    <groupId>com.opencagedata</groupId>
    <artifactId>jopencage</artifactId>
    <version>2.1.0</version>
</dependency>
```

### Gradle

```groovy
implementation 'com.opencagedata:jopencage:2.1.0'
```

Check the [Maven Central repository](https://search.maven.org/artifact/com.opencagedata/jopencage) for the latest version. The older `com.byteowls` groupId is no longer maintained — use `com.opencagedata` for all new projects.

## Basic Usage

```java
import com.opencagedata.jopencage.JOpenCageGeocoder;
import com.opencagedata.jopencage.model.JOpenCageForwardRequest;
import com.opencagedata.jopencage.model.JOpenCageReverseRequest;
import com.opencagedata.jopencage.model.JOpenCageResponse;
import com.opencagedata.jopencage.model.JOpenCageResult;

// Forward geocoding (address → coordinates)
JOpenCageGeocoder geocoder = new JOpenCageGeocoder("YOUR-API-KEY");

JOpenCageForwardRequest request = new JOpenCageForwardRequest("Berlin, Germany");
request.setNoAnnotations(true);
request.setLimit(1);

JOpenCageResponse response = geocoder.forward(request);

if (response != null && response.getResults() != null && !response.getResults().isEmpty()) {
    JOpenCageResult first = response.getResults().get(0);
    Double lat = first.getGeometry().getLat();
    Double lng = first.getGeometry().getLng();
    String formatted = first.getFormatted();
    System.out.println(lat + ", " + lng + " — " + formatted);
}

// Reverse geocoding (coordinates → address)
JOpenCageReverseRequest reverseRequest = new JOpenCageReverseRequest(-22.6792, 14.5272);
reverseRequest.setNoAnnotations(true);

JOpenCageResponse reverseResponse = geocoder.reverse(reverseRequest);
if (reverseResponse != null && reverseResponse.getResults() != null
        && !reverseResponse.getResults().isEmpty()) {
    System.out.println(reverseResponse.getResults().get(0).getFormatted());
}
```

Always null-check the response and the results list — the API may return zero results for valid but unrecognised queries.

## Configuring Requests

Request parameters map to setter methods on the request object. Common ones:

```java
JOpenCageForwardRequest request = new JOpenCageForwardRequest("Museumsinsel, Berlin");
request.setLanguage("de");           // BCP 47 language tag for returned text
request.setCountrycode("de");        // Restrict results to a country (ISO 3166-1 alpha-2)
request.setLimit(5);                 // Maximum number of results (default 10)
request.setMinConfidence(3);         // Only return results with confidence >= this value
request.setNoAnnotations(true);      // Skip annotations — smaller, faster response
request.setNoDedupe(false);          // Allow duplicate results (default false = dedupe on)
```

**Always call `setNoAnnotations(true)`** when you only need coordinates or the formatted address — it reduces response size and latency.

For reverse requests:

```java
JOpenCageReverseRequest request = new JOpenCageReverseRequest(51.5074, -0.1278);
request.setLanguage("en");
request.setNoAnnotations(true);
```

## Checking the Response Status

```java
JOpenCageResponse response = geocoder.forward(request);

if (response == null) {
    System.err.println("Request failed — check network connectivity");
} else if (response.getStatus().getCode() != 200) {
    System.err.println("API error " + response.getStatus().getCode()
        + ": " + response.getStatus().getMessage());
} else if (response.getResults().isEmpty()) {
    System.out.println("No results found");
} else {
    System.out.println(response.getResults().get(0).getFormatted());
}
```

Common status codes and their meanings are documented in **opencage-geocoding-api.md**. Key ones to handle:

- `200` — OK
- `401` — Invalid or missing API key
- `402` — Quota exceeded
- `403` — Key suspended
- `429` — Too many requests (per-second rate limit hit)

## Accessing Components

`components` fields are returned as a `Map<String, String>` and are not guaranteed to be present for every result. Use `getOrDefault`:

```java
import com.opencagedata.jopencage.model.JOpenCageResult;
import java.util.Map;

JOpenCageResult result = response.getResults().get(0);
Map<String, String> components = result.getComponents();

String country  = components.getOrDefault("country", "");
String city     = components.getOrDefault("city",
                    components.getOrDefault("town",
                      components.getOrDefault("village", "")));
String postcode = components.getOrDefault("postcode", "");
String type     = components.getOrDefault("_type", "");
```

## Batch Geocoding

For processing a list of addresses, pause between requests to respect the 1 req/s rate limit on free-trial accounts:

```java
import java.util.List;
import java.util.ArrayList;
import java.util.Map;
import java.util.HashMap;

JOpenCageGeocoder geocoder = new JOpenCageGeocoder(System.getenv("OPENCAGE_API_KEY"));
List<String> addresses = List.of("Berlin, Germany", "Paris, France", "London, UK");
List<Map<String, Object>> results = new ArrayList<>();

for (String address : addresses) {
    JOpenCageForwardRequest req = new JOpenCageForwardRequest(address);
    req.setNoAnnotations(true);
    req.setLimit(1);

    JOpenCageResponse resp = geocoder.forward(req);
    Map<String, Object> row = new HashMap<>();
    row.put("input", address);

    if (resp != null && resp.getStatus().getCode() == 200 && !resp.getResults().isEmpty()) {
        JOpenCageResult first = resp.getResults().get(0);
        row.put("lat", first.getGeometry().getLat());
        row.put("lng", first.getGeometry().getLng());
        row.put("formatted", first.getFormatted());
    } else if (resp != null && resp.getStatus().getCode() == 429) {
        System.err.println("Rate limit hit — pausing");
        Thread.sleep(2000);
    } else {
        row.put("error", resp != null ? resp.getStatus().getMessage() : "null response");
    }

    results.add(row);
    Thread.sleep(1000);  // 1 req/s for free-trial accounts; remove for paid subscriptions
}
```

## Environment Variables

Never hard-code your API key. Read it from an environment variable at runtime:

```java
JOpenCageGeocoder geocoder = new JOpenCageGeocoder(System.getenv("OPENCAGE_API_KEY"));
```

For Spring Boot applications, bind the key from `application.properties`:

```properties
opencage.api-key=${OPENCAGE_API_KEY}
```

```java
@Value("${opencage.api-key}")
private String apiKey;

// Then:
JOpenCageGeocoder geocoder = new JOpenCageGeocoder(apiKey);
```

## Further Reading

- OpenCage Java tutorial: https://opencagedata.com/tutorials/geocode-in-java
- jopencage on Maven Central: https://search.maven.org/artifact/com.opencagedata/jopencage
- Source code: https://github.com/OpenCageData/jopencage
- General API reference: **opencage-geocoding-api.md**

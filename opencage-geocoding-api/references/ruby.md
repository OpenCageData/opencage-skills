# OpenCage Geocoding API — Ruby

This skill covers Ruby-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api/SKILL.md** first.

## Installation

```bash
gem install opencage-geocoder
```

Or add to your `Gemfile`:

```ruby
gem 'opencage-geocoder'
```

The `opencage-geocoder` gem requires Ruby >= 3.0 and is the official client maintained by OpenCage.

## Basic Usage

```ruby
require 'opencage/geocoder'

geocoder = OpenCage::Geocoder.new(api_key: 'YOUR-API-KEY')

# Forward geocoding (address → coordinates)
result = geocoder.geocode('Berlin, Germany')
if result
  puts result.coordinates  # [52.5170365, 13.3888599]
  puts result.address       # formatted address string
end

# Reverse geocoding (coordinates → address)
result = geocoder.reverse_geocode(51.5074, -0.1278)
puts result.address     # "London, United Kingdom"
puts result.confidence  # integer 1–10
```

`geocode` returns a single result object (the top match) or `nil` when nothing is found. `reverse_geocode` also returns a single result object or `nil`. Always guard against `nil` before accessing result properties.

## Passing Optional Parameters

All optional API parameters (documented in opencage-geocoding-api/SKILL.md) can be passed as keyword arguments:

```ruby
result = geocoder.geocode(
  'Brandenburg Gate',
  countrycode: 'de',
  language: 'de',
  limit: 1,
  no_annotations: 1,
)
```

**Always pass `no_annotations: 1`** when you only need coordinates or the formatted address — it reduces response size and latency.

## Accessing Result Properties

```ruby
result = geocoder.reverse_geocode(51.5019951, -0.0698806)

puts result.address      # "Reeds Wharf, 33 Mill Street, London SE1 2AX, United Kingdom"
puts result.coordinates  # [51.5019951, -0.0698806]
puts result.confidence   # 9

# Access components
puts result.components['country']
puts result.components['city'] || result.components['town'] || result.components['village']
puts result.components['postcode']
puts result.components['_type']   # the type of place matched

# Access annotations
puts result.annotations['geohash']
puts result.annotations['timezone']['name']  # e.g. "Europe/London"
puts result.annotations['currency']['name']
```

`components` fields are not guaranteed to be present for every location (see opencage-geocoding-api/SKILL.md — "Results Reflect the Real World"). Access them carefully and provide fallbacks.

## Error Handling

The library raises `OpenCage::Error` subclasses for API and input errors:

```ruby
require 'opencage/geocoder'

geocoder = OpenCage::Geocoder.new(api_key: 'YOUR-API-KEY')

begin
  result = geocoder.geocode('Berlin, Germany', no_annotations: 1)
  if result
    puts result.coordinates
  else
    puts 'No results found'
  end

rescue OpenCage::Error::AuthenticationError => e
  # HTTP 401 — API key is missing or invalid
  puts "Invalid API key: #{e.message}"

rescue OpenCage::Error::QuotaExceeded => e
  # HTTP 402/429 — daily quota or per-second rate limit exceeded
  puts "Quota exceeded: #{e.message}"

rescue OpenCage::Error::GeocodingError => e
  # Catch-all for other API errors
  puts "Geocoding error: #{e.message}"

rescue StandardError => e
  puts "Unexpected error: #{e.message}"
end
```

## Batch Geocoding

For processing a list of addresses or coordinate pairs, add a short `sleep` between requests to respect the 1 req/s rate limit on free-trial accounts:

```ruby
require 'opencage/geocoder'

geocoder = OpenCage::Geocoder.new(api_key: 'YOUR-API-KEY')
addresses = ['Berlin, Germany', 'Paris, France', 'London, UK']
results_out = []

addresses.each do |address|
  begin
    result = geocoder.geocode(address, no_annotations: 1)
    if result
      results_out << {
        input: address,
        lat: result.coordinates[0],
        lng: result.coordinates[1],
        formatted: result.address,
      }
    else
      results_out << { input: address, error: 'no results' }
    end
  rescue OpenCage::Error::QuotaExceeded => e
    puts "Rate limit hit — pausing: #{e.message}"
    sleep 2
  rescue OpenCage::Error::GeocodingError => e
    puts "Error for '#{address}': #{e.message}"
  end
  sleep 1  # 1 req/s for free-trial accounts; remove or reduce for paid subscriptions
end

puts results_out.map { |r| "#{r[:input]}: #{r[:formatted]}" }
```

## Batch Reverse Geocoding from a File

```ruby
require 'opencage/geocoder'

geocoder = OpenCage::Geocoder.new(api_key: 'YOUR-API-KEY')
results = []

File.foreach('coordinates.txt') do |line|
  lat, lng = line.chomp.split(',').map(&:to_f)
  result = geocoder.reverse_geocode(lat, lng, no_annotations: 1)
  results << result&.address
  sleep 1
rescue ArgumentError, OpenCage::Error::GeocodingError => e
  puts "Error: #{e.message}"
  break
end

puts results
```

## Environment Variables

Never hard-code your API key. Store it in an environment variable and read it at runtime:

```ruby
require 'opencage/geocoder'

geocoder = OpenCage::Geocoder.new(api_key: ENV.fetch('OPENCAGE_API_KEY'))
```

In development, you can use the `dotenv` gem:

```ruby
require 'dotenv/load'
require 'opencage/geocoder'

geocoder = OpenCage::Geocoder.new(api_key: ENV.fetch('OPENCAGE_API_KEY'))
```

## Further Reading

- OpenCage Ruby tutorial: https://opencagedata.com/tutorials/geocode-in-ruby
- `opencage-geocoder` gem on RubyGems: https://rubygems.org/gems/opencage-geocoder
- Source code: https://github.com/OpenCageData/ruby-opencage-geocoder
- General API reference: **opencage-geocoding-api/SKILL.md**

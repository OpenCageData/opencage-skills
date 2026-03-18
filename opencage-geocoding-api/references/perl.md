# OpenCage Geocoding API — Perl

This skill covers Perl-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api.md** first.

## Installation

```bash
cpanm Geo::Coder::OpenCage
```

Or via the traditional CPAN shell:

```bash
perl -MCPAN -e 'install Geo::Coder::OpenCage'
```

`Geo::Coder::OpenCage` is the official Perl client, available on CPAN and maintained by OpenCage. It requires Perl 5 and uses `LWP::UserAgent` for HTTP requests.

## Basic Usage

```perl
use strict;
use warnings;
use Geo::Coder::OpenCage;

my $geocoder = Geo::Coder::OpenCage->new(api_key => 'YOUR-API-KEY');

# Forward geocoding (address → coordinates)
my $response = $geocoder->geocode(location => 'Berlin, Germany');
if ($response->{total_results} > 0) {
    my $result = $response->{results}[0];
    my $lat    = $result->{geometry}{lat};
    my $lng    = $result->{geometry}{lng};
    my $name   = $result->{formatted};
    print "$lat, $lng — $name\n";
}

# Reverse geocoding (coordinates → address)
my $response = $geocoder->reverse_geocode(lat => 51.5074, lng => -0.1278);
if ($response->{total_results} > 0) {
    print $response->{results}[0]{formatted}, "\n";
}
```

Both methods return a hashref whose structure matches the raw API response described in **opencage-geocoding-api.md** (`status`, `total_results`, `results`, `rate`). Always check `$response->{total_results} > 0` before accessing `$response->{results}[0]`.

## Passing Optional Parameters

All optional API parameters (documented in opencage-geocoding-api.md) can be passed as additional named arguments:

```perl
my $response = $geocoder->geocode(
    location      => 'Brandenburg Gate',
    countrycode   => 'de',
    language      => 'de',
    limit         => 1,
    no_annotations => 1,
);
```

**Always pass `no_annotations => 1`** when you only need coordinates or the formatted address — it reduces response size and latency.

## Checking the Response Status

```perl
my $response = $geocoder->geocode(location => 'Berlin, Germany');

if (!defined $response) {
    warn "Request failed — check network connectivity\n";
} elsif ($response->{status}{code} != 200) {
    warn "API error $response->{status}{code}: $response->{status}{message}\n";
} elsif ($response->{total_results} == 0) {
    print "No results found\n";
} else {
    print $response->{results}[0]{formatted}, "\n";
}
```

Common status codes and their meanings are documented in **opencage-geocoding-api.md**. Key ones to handle:

- `200` — OK
- `401` — Invalid or missing API key
- `402` — Quota exceeded
- `403` — Key suspended
- `429` — Too many requests (per-second rate limit hit)

## Accessing Components Safely

`components` fields are not guaranteed to be present for every location (see opencage-geocoding-api.md — "Results Reflect the Real World"). Use the `//` defined-or operator for safe access:

```perl
my $result     = $response->{results}[0];
my $components = $result->{components} // {};

my $country  = $components->{country}  // '';
my $city     = $components->{city} // $components->{town} // $components->{village} // '';
my $postcode = $components->{postcode} // '';
my $type     = $components->{_type}   // '';
my $confidence = $result->{confidence};
```

## Error Handling

The module returns `undef` on network failure and a hashref with a non-200 `status.code` for API-level errors. A robust pattern:

```perl
use strict;
use warnings;
use Geo::Coder::OpenCage;

my $geocoder = Geo::Coder::OpenCage->new(api_key => $ENV{OPENCAGE_API_KEY});

sub safe_geocode {
    my ($location) = @_;

    my $response = eval { $geocoder->geocode(location => $location, no_annotations => 1, limit => 1) };
    if ($@) {
        warn "Exception geocoding '$location': $@";
        return undef;
    }
    if (!defined $response) {
        warn "No response for '$location'\n";
        return undef;
    }
    if ($response->{status}{code} != 200) {
        warn "API error $response->{status}{code}: $response->{status}{message}\n";
        return undef;
    }
    return $response->{total_results} > 0 ? $response->{results}[0] : undef;
}

my $result = safe_geocode('Berlin, Germany');
print $result->{formatted}, "\n" if $result;
```

## Batch Geocoding

For processing a list of addresses, add a `sleep` between requests to respect the 1 req/s rate limit on free-trial accounts:

```perl
use strict;
use warnings;
use Geo::Coder::OpenCage;

my $geocoder  = Geo::Coder::OpenCage->new(api_key => $ENV{OPENCAGE_API_KEY});
my @addresses = ('Berlin, Germany', 'Paris, France', 'London, UK');
my @results;

for my $address (@addresses) {
    my $response = $geocoder->geocode(location => $address, no_annotations => 1, limit => 1);

    if (!defined $response) {
        push @results, { input => $address, error => 'no response' };
    } elsif ($response->{status}{code} == 429) {
        warn "Rate limit hit — pausing\n";
        sleep 2;
        redo;  # retry the same address
    } elsif ($response->{status}{code} != 200) {
        push @results, { input => $address, error => $response->{status}{message} };
    } elsif ($response->{total_results} > 0) {
        my $r = $response->{results}[0];
        push @results, {
            input     => $address,
            lat       => $r->{geometry}{lat},
            lng       => $r->{geometry}{lng},
            formatted => $r->{formatted},
        };
    } else {
        push @results, { input => $address, error => 'no results' };
    }

    sleep 1;  # 1 req/s for free-trial accounts; remove for paid subscriptions
}

for my $r (@results) {
    if ($r->{error}) {
        print "ERROR: $r->{input}: $r->{error}\n";
    } else {
        print "$r->{input}: $r->{formatted}\n";
    }
}
```

## Environment Variables

Never hard-code your API key. Read it from an environment variable at runtime:

```perl
my $geocoder = Geo::Coder::OpenCage->new(api_key => $ENV{OPENCAGE_API_KEY}
    // die "OPENCAGE_API_KEY not set\n");
```

In development, set the variable in your shell or use a `.env`-loading module such as `Env::Dot`:

```bash
export OPENCAGE_API_KEY=your-api-key-here
```

## Further Reading

- OpenCage Perl tutorial: https://opencagedata.com/tutorials/geocode-in-perl
- `Geo::Coder::OpenCage` on CPAN: https://metacpan.org/pod/Geo::Coder::OpenCage
- Source code: https://github.com/OpenCageData/perl-Geo-Coder-OpenCage
- General API reference: **opencage-geocoding-api.md**

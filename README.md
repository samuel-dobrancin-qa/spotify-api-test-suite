[![Spotify API Tests](https://github.com/samuel-dobrancin-qa/spotify-api-test-suite/actions/workflows/newman.yml/badge.svg)](https://github.com/samuel-dobrancin-qa/spotify-api-test-suite/actions/workflows/newman.yml)

# Spotify API — QA Test Suite

Structured API testing of the Spotify Web API across 
authentication, search, and track data endpoints.
12 test cases covering happy path, negative testing, 
boundary conditions, and real edge cases that matter 
to developers building on this API.

---

## Why a real API

Most QA portfolios test practice APIs built for learning.
This collection tests a production API used by thousands
of developers worldwide — which means the edge cases are
real, the findings are relevant, and the behaviour
sometimes surprises you.

---

## What's covered

| Category | Tests | Focus |
|---|---|---|
| Authentication | 4 | OAuth flow, error specificity, token handling |
| Search | 4 | Pagination, URL encoding, fuzzy search behaviour |
| Track Data | 4 | ID validation, market restrictions, schema consistency |

**12/12 tests passing**

---

## Key findings

**Spotify differentiates error codes precisely** — invalid
credentials, missing parameters, invalid ID format, and
non-existent resources all return different error codes
with specific messages. Good API design that makes
debugging straightforward.

**Two services, two error schemas** — the accounts endpoint
and the API endpoint return errors in different JSON
structures. Applications consuming both need to handle
each format separately.

**`preview_url` is silently deprecated** — present in
responses but consistently null or absent. No warning,
no deprecation notice in the response. Silent breakage
for apps that depend on it.

**Fuzzy search always returns results** — a 20-character
random string returned 841 tracks. You cannot rely on
empty results to trigger "no results found" UI states.

**Market restrictions return 200, not an error** — region-
locked tracks return full data with `is_playable: false`
and `restrictions.reason: "market"`. Checking only for
200 and assuming playability will break for users in
restricted markets.

---

## How to run

### Prerequisites
- Postman installed
- Free Spotify developer account at developer.spotify.com
- A Spotify app created with Client ID and Client Secret

### Setup
1. Import `spotify-api-test-suite.json` into Postman
2. Create a Postman environment called `Spotify QA`
3. Add your `client_id` and `client_secret` as variables
4. Set `base_url` to `https://api.spotify.com/v1`

### Running
Run `AUTH-01` first — it automatically saves the access
token to the environment. Then run the full collection
in sequence. Token expires after 60 minutes — re-run
AUTH-01 if you see 401 errors mid-run.

---

## Files

- `spotify-api-test-suite.json` — Postman collection
- `SPOTIFY-API-TEST-REPORT.md` — Full test report with
  findings and observations per test case

---

## About

Samuel Dobrančin — Quality Engineer  
4+ years enterprise QA, currently deepening API testing
skills through Postman Academy and real API evaluation.

[GitHub](https://github.com/samuel-dobrancin-qa) ·
[LinkedIn](https://linkedin.com/in/samuel-dobrancin-8a203a273)

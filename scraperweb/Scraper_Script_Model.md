# Scraper Script Model

Typed C++ facades + per-provider scripts. Prefer this over the legacy `DIOSCRAPERWEB` XML engine.

## Roles

| Layer | Responsibility |
|-------|----------------|
| **C++ facade** (`DIOSCRAPERWEBPUBLICIP`, `DIOSCRAPERWEBGEOLOCATIONIP`, `DIOSCRAPERWEBWEATHER`, `DIOSCRAPERWEBTRANSLATION`, `DIOSCRAPERWEBMACMANUFACTURER`, `DIOSCRAPERWEBUSERAGENTID`, …) | Stable typed API, cache (`ask` → result), resolve/run script, map results |
| **`DIOSCRAPERSCRIPT`** | Arg/result maps, run script under scripts root, attach `SCRIPT_LIB_SCRAPER` |
| **Script** (`assets/scripts/scrapers/*.{g,js,lua}`) | Choose URL(s), `WebClient_*` download, `ExtractBetween` / String helpers, `Scraper_SetResult` |

Scripts are **not** auto-registered via `AddInternalLibraries`. The runner adds `SCRIPT_LIB_SCRAPER` with a live context. Under `SCRIPT_LIB_SANDBOX`, scrapers do not run (same idea as WebClient). Language is chosen by file extension (`.g` / `.js` / `.lua`). Lua scrapers that define `function main()` must end with `return main()` so the chunk actually invokes it.

## Contract: PublicIP

**Script path (default):** `scrapers/publicip.g`

Providers: **api.ipify.org** (primary) → **ifconfig.me/ip** (fallback). No API key.

| Direction | Name | Type | Notes |
|-----------|------|------|--------|
| arg | `timeout` | int | Seconds; default 10 in script if ≤ 0 |
| arg | `localIP` | string | Optional |
| result | `ok` | `"1"` / `"0"` | Success flag |
| result | `ip` | string | Public IPv4/IPv6 text |

**C++ API:** `DIOSCRAPERWEBPUBLICIP::Get(DIOIP&, timeout, localIP*, usecache)`

## Contract: Geolocation IP

**Script path (default):** `scrapers/geolocationip.g`

Providers: **ip-api.com/json** (primary) → **ipwho.is** (fallback). No API key.

| Direction | Name | Type |
|-----------|------|------|
| arg | `ip` | string (required) |
| arg | `timeout` | int |
| arg | `localIP` | string (optional) |
| result | `ok` | `"1"` / `"0"` |
| result | `country`, `state`, `city`, `isp`, `organization` | string |
| result | `latitude`, `longitude` | string (parsed as float in C++) |

**C++ API:** `DIOSCRAPERWEBGEOLOCATIONIP::Get(…, DIOGEOLOCATIONIP_RESULT&, …)`

## Contract: Weather

**Script path (default):** `scrapers/weather.g`

Providers: **Open-Meteo** geocoding + forecast (primary) → **Nominatim** geocode + Open-Meteo forecast → **wttr.in/?format=j1** (last resort). No API key. Legacy OpenWeatherMap XML is not used.

| Direction | Name | Type | Notes |
|-----------|------|------|--------|
| arg | `location` | string | City name (required) |
| arg | `celsius` | int | `1` metric, `0` Fahrenheit |
| arg | `timeout` | int | Seconds; default 10 if ≤ 0 |
| arg | `localIP` | string | Optional |
| result | `ok` | `"1"` / `"0"` | Success flag |
| result | `condition` | string | WMO code mapped to short text when known |
| result | `temperature` | string | Parsed as float in C++ |
| result | `humidity` | string | Parsed as float in C++ |

**C++ API:** `DIOSCRAPERWEBWEATHER::Get(location, iscelsius, DIOWEATHER_RESULT&, …)` (also legacy out-param overload). Cache key: `location|C` / `location|F`.

## Contract: Translation

**Script path (default):** `scrapers/translation.g`

Providers: **Google Translate free** `client=gtx` (primary) → **MyMemory** `api.mymemory.translated.net` (fallback). No API key. Legacy XML Google URL is not used. Scripts percent-encode the query (space/newline/tab/`&`/`#`/`+`/`?`/`=`/quotes/braces, and `%` via a `__PCT__` placeholder then `%25` — direct `%`→`%25` would infinite-loop in `XSTRING::Replace`) so reserved URL characters and TranslateScan-style `__PFn__` tokens stay intact.

| Direction | Name | Type | Notes |
|-----------|------|------|--------|
| arg | `text` | string | Phrase to translate (required) |
| arg | `tl` | string | Target language ISO 639-1 (required), e.g. `es`, `fr` |
| arg | `sl` | string | Source language; default `auto` |
| arg | `timeout` | int | Seconds; default 10 if ≤ 0 |
| arg | `localIP` | string | Optional |
| result | `ok` | `"1"` / `"0"` | Success flag |
| result | `translation` | string | Translated text |
| result | `detected` | string | Detected source language when `sl=auto` |

**C++ API:** `DIOSCRAPERWEBTRANSLATION::Get(text, sl, tl, DIOTRANSLATION_RESULT&, …)` (also out-param overload). Cache key: `text|sl|tl`.

Language codes: Google / MyMemory expect **ISO 639-1** (`en`, `es`, `zh`, …) — the same values stored in `XLANGUAGE_ISO_639_3_ENTRY::codealpha2`. `Get` normalizes input via `XLANGUAGE_ISO_639_3::CodeAlpha2_Resolve` (accepts alpha2, GEN alpha3 such as `spa`/`eng`, and legacy Google `iw`/`jw`). BCP-47 tags (`zh-CN`, `zh-TW`) are passed through unchanged. `sl` may be `auto`.

Before caching, `Get` runs `DecodeTranslationText`: URL percent-escapes (`%20`, `%3A`, …) are decoded via `DIOURL::DecodeUnsafeCharsToString` into readable text. C/C++ printf masks (`%s`, `%d`, `%2d`, `%08X`, `%%`, …) are preserved.

## Contract: MAC Manufacturer

**Script path (default):** `scrapers/macmanufacturer.g`

Providers: **api.maclookup.app/v2/macs** (primary) → **api.macvendors.com** plain-text (fallback). No API key. Legacy coffer.com HTML scrape is not used.

| Direction | Name | Type | Notes |
|-----------|------|------|--------|
| arg | `mac` | string | MAC address (required), e.g. `00:1B:63:84:45:E6` |
| arg | `timeout` | int | Seconds; default 10 if ≤ 0 |
| arg | `localIP` | string | Optional |
| result | `ok` | `"1"` / `"0"` | Success flag |
| result | `manufacturer` | string | Vendor / company name |

**C++ API:** `DIOSCRAPERWEBMACMANUFACTURER::Get(DIOMAC&, DIOMACMANUFACTURED_RESULT&, …)` (also `XSTRING& manufactured` overload). Cache key: MAC string.

## Contract: User Agent ID

**Script path (default):** `scrapers/useragentid.g` (facade default file name `useragentid.g` under the SCRAPERS section).

Providers: **www.useragentstring.com** `getJSON=all` (primary) → **useragentstring.com** HTTP (fallback). No API key. The C++ facade URL-encodes the User-Agent (`DIOURL::EncodeUnsafeCharsFromString`) and passes it as arg `ua`.

| Direction | Name | Type | Notes |
|-----------|------|------|--------|
| arg | `ua` | string | URL-encoded User-Agent (required) |
| arg | `timeout` | int | Seconds; default 10 if ≤ 0 |
| arg | `localIP` | string | Optional |
| result | `ok` | `"1"` / `"0"` | Success flag |
| result | `browser` | string | `agent_name` |
| result | `browser_version` | string | `agent_version` |
| result | `browser_type` | string | `agent_type` (Browser, Robot, …) |
| result | `os` | string | `os_name` (legacy SO field) |
| result | `os_type` | string | `os_type` |
| result | `os_version` | string | `os_versionNumber` or `os_versionName` |
| result | `language` | string | `agent_language` |
| result | `language_tag` | string | `agent_languageTag` |

**C++ API:** `DIOSCRAPERWEBUSERAGENTID::Get(…, DIOUSERAGENTID_RESULT&, …)` (also legacy `browser`/`systemoperative` out-params). Cache key: raw User-Agent string.

## Script helpers

```
Scraper_GetArg(name) → string
Scraper_GetArgInt(name) → int
Scraper_SetResult(name, value)
Scraper_GetResult(name) → string
```

Plus standard `WebClient_*` and String libs (`ExtractBetween`, `TrimString`, `AddString`, …).

## Layout

```
Resources/scraperweb/scripts/     # canonical sources
  publicip.g / publicip.js / publicip.lua
  geolocationip.g / geolocationip.js / geolocationip.lua
  weather.g / weather.js / weather.lua (+ weather_mock.g)
  translation.g / translation.js / translation.lua (+ translation_mock.g)
  macmanufacturer.g / macmanufacturer.js / macmanufacturer.lua (+ macmanufacturer_mock.g)
  useragentid.g / useragentid.js / useragentid.lua (+ useragentid_mock.g)

<app>/assets/scripts/scrapers/    # deployed copies (UnitTests, ScriptsExample, DevTests, …)
```

Default facade paths remain `.g`. Override with `SetScriptPath(L"scrapers/publicip.js")` or `.lua` when those languages are compiled in (`SCRIPT_JAVASCRIPT_ACTIVE` / `SCRIPT_LUA_ACTIVE`). Same `Scraper_*` / `WebClient_*` / String helper names in all three languages.

## Legacy XML

`DIOSCRAPERWEB` + `scraperweb.xml` remain for unmigrated types (SexName). **No new features** on the XML engine. Migrated facades (PublicIP, GeolocationIP, Weather, Translation, MAC Manufacturer, UserAgentID) no longer inherit `DIOSCRAPERWEB` or load XML.

## Feature flags

In the app `CMakeLists.txt`, enable scrapers individually:

```cmake
option(DIO_SCRAPERWEB_PUBLICIP_FEATURE         "Scraper Web Public IP"         ON )
option(DIO_SCRAPERWEB_GEOLOCATIONIP_FEATURE    "Scraper Web Geolocation IP"    ON )
option(DIO_SCRAPERWEB_WEATHER_FEATURE          "Scraper Web Weather"           ON )
option(DIO_SCRAPERWEB_TRANSLATION_FEATURE      "Scraper Web Translation"       ON )
option(DIO_SCRAPERWEB_MACMANUFACTURER_FEATURE  "Scraper Web MAC Manufacturer"  ON )
option(DIO_SCRAPERWEB_USERAGENTID_FEATURE      "Scraper Web User Agent ID"     ON )
```

Any child option turns on `DIO_SCRAPERWEB_FEATURE` in `GEN_Main_Dependencies.cmake` (common cache + script runner + Script libs). C++ defines:

- `DIO_SCRAPERWEB_ACTIVE` — common
- `DIO_SCRAPERWEB_PUBLICIP_ACTIVE`
- `DIO_SCRAPERWEB_GEOLOCATIONIP_ACTIVE`
- `DIO_SCRAPERWEB_WEATHER_ACTIVE`
- `DIO_SCRAPERWEB_TRANSLATION_ACTIVE`
- `DIO_SCRAPERWEB_MACMANUFACTURER_ACTIVE`
- `DIO_SCRAPERWEB_USERAGENTID_ACTIVE`

The legacy XML engine (`DIOScraperWeb.cpp`) and unused scrapers (SexName) are **not** compiled.

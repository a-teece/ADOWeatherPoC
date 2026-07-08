**Context references:**
- `Context.MD`
- `Technical-Context.MD`
- `PRD.md`
- `docs/adr/0001-units-are-per-measurement-not-a-metric-imperial-bundle.md`
- `docs/adr/0002-canonical-condition-set-mapped-at-the-edge.md`
- `docs/adr/0003-cross-platform-desktop-windows-first-mac-later.md`
- `docs/adr/0004-keyless-free-tier-services-only.md`

(No `Roadmap.md` — the PRD covers a single Feature, confirmed with the user during `/roadmap`.)

# Weather PoC — Design

## Summary

Weather PoC is a single Feature covering the whole PoC experience end-to-end, per the PRD. This
Spec designs the three-screen app (Weather / Search overlay / Settings), the interaction flows
around location detection and switching, and the module wiring that composes the PRD's five deep
modules plus `ThemePreference` into MVVM ViewModels.

## Screens

- **Weather Page** (landing screen) — Current Conditions at top, Hourly Forecast as a horizontal
  scroll strip beneath it, Daily Forecast as a vertical list beneath that. Carries a search icon
  (opens the Search overlay), a settings icon (navigates to the Settings Page), a pull-to-refresh
  gesture, and a visible refresh button (both trigger the same re-fetch).
- **Search overlay** (modal, not a navigation destination) — a text field with debounced
  as-you-type queries against `PlaceSearchService`. Each result row shows name + region + country
  (Requirement 11). Selecting a row closes the overlay and hands the `Place` back to the Weather
  Page.
- **Settings Page** (separate navigation destination) — exactly three controls: Temperature unit
  (°C/°F), Wind-speed unit (mph/km/h), Theme (System/Light/Dark). Nothing else.

## Modules and ViewModel composition

The PRD's five deep modules (`WeatherClient`, `LocationResolver`, `PlaceSearchService`,
`UnitConventionProvider`, `ActiveLocationStore`) plus `ThemePreference` are each injected via DI
into exactly one ViewModel that owns them:

- **`WeatherPageViewModel`** — owns `WeatherClient`, `LocationResolver`, `ActiveLocationStore`,
  `UnitConventionProvider` (read-only for rendering). Drives the Weather Page's entire display
  state and the location-switch banner.
- **`PlaceSearchViewModel`** — owns `PlaceSearchService` only.
- **`SettingsPageViewModel`** — owns `UnitConventionProvider` (read/write) and `ThemePreference`.

No module is called from more than one ViewModel. Cross-page composition happens via
`CommunityToolkit.Mvvm`'s `WeakReferenceMessenger`, carrying exactly two message types:

1. **Settings-changed** (unit or theme) — published by `SettingsPageViewModel`, subscribed to by
   `WeatherPageViewModel` (re-renders already-fetched data in the new units, no re-fetch) and the
   app shell (applies the new theme live).
2. **Place-selected** — published by `PlaceSearchViewModel` on selection, subscribed to by
   `WeatherPageViewModel` (persists the new Active Location via `ActiveLocationStore`, re-fetches).

The in-process messenger itself is plain in-memory pub/sub within a single process — it crosses no
seam boundary from the taxonomy and is not enumerated below; it's covered by the ViewModel-level
behavioural tests instead (see Testing).

## Interaction flows

**Launch sequence:**
1. `ActiveLocationStore` restores the last Active Location (if any). `WeatherPageViewModel`
   immediately starts fetching Current/Hourly/Daily for it, showing a skeleton loading state
   (placeholder shapes for each section) until the first response lands.
2. In parallel, `LocationResolver` runs its cascade: device geolocation (MAUI `Geolocation`) → IP
   geolocation (BigDataCloud `reverse-geocode-client`) → none.
3. If a Detected Location resolves and differs from the restored Active Location, a dismissible
   inline banner appears at the top of the Weather Page ("You're now in **X** — switch?") with a
   Switch action and a × dismiss. Switching re-points the Active Location (persisted) and
   re-fetches; dismissing or ignoring makes the banner disappear with no change. This check re-runs
   on every launch — there is no "don't ask again" tracking.
4. If there is no restored Active Location at all (true first-ever launch) and detection resolves
   something, that becomes the Active Location directly (no banner — nothing to compare against).
   If detection resolves nothing either (both device and IP legs fail), the app opens straight into
   the Search overlay.

**Search:** debounced as-you-type queries to `PlaceSearchService`. A response with no `results` key
(the provider's zero-match shape — see Seam 2) renders as an empty result list, not an error.
Selecting a row closes the overlay and publishes the place-selected message.

**Refresh:** pull-to-refresh and the refresh button both call `WeatherPageViewModel`'s single
re-fetch method; the same method also runs automatically when the app returns to the foreground.

**Errors:** every fetch failure in this PoC is transient (no hard-failure/blocking-modal case
exists — confirmed with the user). `WeatherPageViewModel` retains whatever data it last
successfully loaded and surfaces an inline status message plus a non-blocking toast
(`CommunityToolkit.Maui`). No previously-loaded data is ever cleared solely because a refresh
failed.

**Units/theme changes:** `SettingsPageViewModel` writes through `UnitConventionProvider` /
`ThemePreference` immediately (no separate save step) and publishes the settings-changed message.

## Seam inventory

### Seam 1: WeatherClient ↔ Open-Meteo Forecast API
- **(a) class:** network-protocol, external
- **(b) sides:** `WeatherClient` ↔ Open-Meteo Forecast API (`api.open-meteo.com/v1/forecast`)
- **(c) contract:** `GET /v1/forecast?latitude={lat}&longitude={lon}&current=...&hourly=...&daily=...&timezone=auto`. **No API key or auth header required** for non-commercial/keyless use (first contact with this system pins the auth method: none — confirmed live, matches ADR-0004). Response is JSON with a `current` object (present because `WeatherClient` always requests it — contains scalar values including `weather_code` [integer WMO code] and `is_day` [integer 0 or 1, used for day/night graphic selection]), an `hourly` object (a `time` array of ISO8601 strings plus one parallel array per requested variable, including `weather_code`), and a `daily` object (same shape, daily granularity). No null elements were observed in successful-response arrays; since Open-Meteo's docs don't explicitly guarantee no per-element gaps at the extremes of `forecast_days`, `WeatherClient` must treat a missing/null array element defensively (map it to "condition unavailable for that slot" rather than throwing).
- **(d) proof:** live GET issued 2026-07-08 to `api.open-meteo.com/v1/forecast` with `current=temperature_2m,weather_code,is_day`, `hourly=temperature_2m,weather_code`, `daily=weather_code,temperature_2m_max,temperature_2m_min` — observed 200 OK with the shape described in (c) (see session transcript). Tier 1 regression: a recorded-replay fixture test built from this captured payload, asserting `WeatherClient` parses it into the domain's Current/Hourly/Daily objects including `weather_code`→`Condition` mapping and `is_day`→day/night selection. Tier 2 (per the PRD's existing testing decision) exercises the real endpoint on a bounded, scheduled basis.
- **(e) authority:** https://open-meteo.com/en/docs (fetched 2026-07-08) plus the live call above (observed 2026-07-08).

### Seam 2: PlaceSearchService ↔ Open-Meteo Geocoding API
- **(a) class:** network-protocol, external
- **(b) sides:** `PlaceSearchService` ↔ Open-Meteo Geocoding API (`geocoding-api.open-meteo.com/v1/search`)
- **(c) contract:** `GET /v1/search?name={query}&count={n}&language=en`. No auth (inherits the keyless decision from Seam 1 — same provider). On ≥1 match, the response body has a top-level `results` array of objects carrying `name`, `country`, `admin1` (region), `latitude`, `longitude` (plus other fields `PlaceSearchService` ignores). **On zero matches, the `results` key is entirely ABSENT from the response body** — not an empty array, not null. `PlaceSearchService` MUST treat a missing `results` key as "zero `Place` candidates," not as a parse error.
- **(d) proof:** live GET issued 2026-07-08 with query `name=zzzzznonexistentplacexyz123` — observed response `{"generationtime_ms":0.5147457}` with no `results` key present (see session transcript). Tier 1 regression: a recorded-replay fixture covering both the populated-`results` shape and this exact absent-key zero-match shape.
- **(e) authority:** https://open-meteo.com/en/docs/geocoding-api (fetched 2026-07-08) plus the live zero-match call above (observed 2026-07-08).

### Seam 3: LocationResolver ↔ BigDataCloud reverse-geocode-client API
- **(a) class:** network-protocol, external — **first contact with this system**
- **(b) sides:** `LocationResolver` ↔ BigDataCloud `reverse-geocode-client` endpoint (`api.bigdatacloud.net`)
- **(c) contract:** `GET https://api.bigdatacloud.net/data/reverse-geocode-client?localityLanguage=en` with `latitude`/`longitude` omitted (the IP-geolocation fallback path). **No API key required** — first contact with this new external system pins the auth method as none (public keyless GET, confirmed live), satisfying ADR-0004. Response JSON includes `lookupSource` (string; observed value `"ip geolocation"` when lat/long are omitted), `latitude`/`longitude` (numbers — the resolved location), `city`, `principalSubdivision`, `countryName`, `countryCode` (strings). `postcode`, `plusCode`, `fips`, `csdCode`, and `localityInfo` are **omitted from the response entirely** (not null, not empty-string) when unavailable for that location — `LocationResolver` must not assume their presence and must not read them (it only needs coordinates + a display name).
- **(d) proof:** live GET issued 2026-07-08 with no `latitude`/`longitude` params — observed 200 OK with `lookupSource: "ip geolocation"` and a fully populated location resolved from the caller's IP (see session transcript; a second live call with explicit `latitude`/`longitude` confirmed `lookupSource: "coordinates"` for comparison). Tier 1 regression: a recorded-replay fixture from the captured IP-geolocation payload. Tier 2 (per the PRD's existing `LocationResolver` testing decision) exercises the real endpoint on a scheduled basis.
- **(e) authority:** https://www.bigdatacloud.com/free-api/free-reverse-geocode-to-city-api (fetched 2026-07-08) plus the two live calls above (observed 2026-07-08).

### Seam 4: LocationResolver ↔ device geolocation (MAUI Geolocation API)
- **(a) class:** host-OS/runtime, external (the OS's location service is outside our control; MAUI is the cross-platform accessor)
- **(b) sides:** `LocationResolver` ↔ `Microsoft.Maui.Devices.Sensors.Geolocation.GetLocationAsync()`
- **(c) contract:** `GetLocationAsync()` returns `Task<Location?>`. It **can return `null`** — per Microsoft's docs, this "indicates that the underlying platform is unable to obtain the current location" (denied/unavailable/etc. are collapsed into this single null outcome rather than distinct exceptions for that case). Permission is requested automatically by the API at call time; no separate pre-check call is required. Within a non-null `Location`, `Latitude`/`Longitude` are always present, but `Altitude`, `Speed`, and `Course` may themselves be null/absent depending on device/platform — `LocationResolver` only reads `Latitude`/`Longitude` so this doesn't affect its contract. `LocationResolver`'s cascade treats a `null` result as "device geolocation failed" and falls through to Seam 3 (IP geolocation), never throwing on this path.
- **(d) proof:** no owning symbol yet (first build). Tier 1 test: a faked `IGeolocation` returning `null` (asserts cascade falls through to the IP leg) and returning a populated `Location` (asserts it's used directly, no fallback call made). Tier 2/manual: verified against the real Windows location permission prompt and service on the actual target OS, per the PRD's "platform matrix is non-negotiable" testing rule.
- **(e) authority:** https://learn.microsoft.com/dotnet/maui/platform-integration/device/geolocation (net-maui-10.0) and https://learn.microsoft.com/dotnet/api/microsoft.maui.devices.sensors.geolocation.getlocationasync (net-maui-10.0), both fetched 2026-07-08.

### Seam 5: ActiveLocationStore / UnitConventionProvider / ThemePreference ↔ MAUI Preferences
- **(a) class:** persistent-on-disk-state, internal (both sides are first-party — our code and the MAUI framework's own storage abstraction already fixed in `Technical-Context.MD`, not a third-party service)
- **(b) sides:** `ActiveLocationStore` / `UnitConventionProvider` / `ThemePreference` ↔ `Microsoft.Maui.Storage.Preferences.Default`
- **(c) contract:** `Preferences.Default.Get<T>(key, defaultValue)` returns `defaultValue` whenever `key` does not exist — it never throws or returns `null` for a missing key. A stored value can legitimately equal the default, so `ContainsKey(key)` — not the returned value — is the only reliable "has this been set before?" check. Each store therefore has an explicit "nothing stored yet" contract: `ActiveLocationStore` returns an explicit no-active-location result (never a bogus `0,0` coordinate) when `ContainsKey` is false for its key; unit/theme overrides fall back to their locale-derived/System default the same way.
- **(d) proof:** no owning symbol yet (first build). Tier 1 round-trip test against the real `Preferences` API (write, read back, assert equality) plus an explicit test of the no-key-yet path for each store, asserting the documented absent-state result — this meets the real-I/O bar since it's the actual local-storage implementation, not a hand-rolled fake of it.
- **(e) n/a** — internal seam.

### Seam 6: ThemePreference ↔ OS theme signal
- **(a) class:** host-OS/runtime, external
- **(b) sides:** `ThemePreference` ↔ `Microsoft.Maui.ApplicationModel.AppInfo.Current.RequestedTheme` (plus live updates consumed via `AppThemeBinding`)
- **(c) contract:** `RequestedTheme` returns one of `AppTheme.Light`, `AppTheme.Dark`, or `AppTheme.Unspecified`. `Unspecified` is documented as returned "when the operating system doesn't have a specific user interface style" (a case Microsoft documents for pre-13 iOS; Windows 10+ is expected to always report Light or Dark, but the contract does not guarantee it never returns `Unspecified` on Windows). `ThemePreference`'s "System" mode MUST define a fallback for `Unspecified` — **Light** — rather than crash or render an undefined state. Live theme changes while the app runs are consumed via `AppThemeBinding`/`SetAppThemeColor`, which MAUI documents as automatically updating bound values when the system theme changes — `ThemePreference` does not need to poll.
- **(d) proof:** no owning symbol yet (first build). Tier 1 test driving all three enum values (including `Unspecified`) through `ThemePreference`'s "System" resolution logic, asserting the Light fallback specifically for `Unspecified`.
- **(e) authority:** https://learn.microsoft.com/dotnet/maui/user-interface/system-theme-changes and https://learn.microsoft.com/dotnet/maui/platform-integration/appmodel/app-information (net-maui-10.0), both fetched 2026-07-08.

## Testing

Confirms and extends the PRD's Testing Decisions with the concrete ViewModel- and seam-level
behavior from this Spec:

- **`WeatherClient`, `LocationResolver`, `UnitConventionProvider`, `ActiveLocationStore`** — scope
  unchanged from the PRD (real-IO Tier 1/2 coverage on the two external seams; deterministic Tier 1
  coverage on the two local-persistence modules), now backed by the concrete seam contracts above.
- **`PlaceSearchService`** — Tier 1 coverage per the PRD's ratchet stance, now additionally required
  to cover the absent-`results`-key zero-match shape (Seam 2) as a named case, not just the
  populated-results happy path.
- **`WeatherPageViewModel`** — owns real observable behavior worth pinning: given a detected
  location differing from the restored active one, the switch-banner state becomes visible; given a
  fetch failure, previously-loaded data is retained and an error/status state is set rather than
  cleared; given a settings-changed message, displayed values re-render in new units without a
  re-fetch call. Asserted as the ViewModel's own bindable state, with injected modules faked
  (`NSubstitute`) since seam-level real-IO testing already lives in the modules themselves.
- **`ThemePreference`** — the `Unspecified`→Light fallback (Seam 6) is a named test case, not left
  implicit.
- **`SettingsPageViewModel`, `PlaceSearchViewModel`** — remain thin pass-throughs to their modules;
  not called out with dedicated tests beyond what the underlying modules already get.
- **Prior art:** none yet (first Feature) — this Spec is the first to add ViewModel- and
  seam-contract-level test scope on top of the PRD's module-level pattern.

## Out of scope

Unchanged from the PRD's Out of Scope section — this Spec designs the PRD's single Feature and
introduces no new scope beyond it. Notably still out of scope: any hard-failure/blocking-modal UX
(confirmed with the user — no such case exists in this PoC), a "don't ask again" mechanism for the
location-switch banner, and any specific icon-pack selection for Condition graphics (deferred to
implementation time, bounded by: free/open license, day+night variant per canonical Condition).

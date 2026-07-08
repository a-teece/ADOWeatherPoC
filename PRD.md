## Problem Statement

Most weather apps present numbers first — a temperature, a percentage, a text label — and treat
graphics as decoration. A viewer wants to glance at a screen and immediately grasp what the sky is
doing right now, this afternoon, and over the coming days, at a place of their choosing, without
parsing rows of figures. Existing weather clients also tend to assume the viewer's units match the
place being viewed, and often demand an account or API key before they'll show anything.

## Solution

Weather PoC is a Windows desktop application (cross-platform-capable, with macOS a planned later
target) that shows the weather for a single **Active Location** at a time, led by picture-based
**Condition** graphics rather than a data-only readout. On launch it resolves the user's
**Detected Location** automatically (device geolocation, falling back to IP-based geolocation),
and shows that place's **Current Conditions**, **Hourly Forecast** (to local midnight), and
**Daily Forecast** (a soft seven-day target). The user can search for and switch to any other
**Place** at any time. Display units follow the *viewer* (locale-defaulted, per-measurement
overridable), never the place being viewed. The app uses only keyless, free-tier data services
(Open-Meteo), so it ships no secret and needs no backend.

## Requirements

1. As a first-time user, I want the app to automatically detect my location on launch, so that I see relevant weather without any setup.
2. As a user whose device geolocation is unavailable or denied, I want the app to fall back to IP-based geolocation, so that I still get a useful default location.
3. As a user whose location can't be resolved by any method, I want to be taken straight to location search, so that I can pick a place myself instead of seeing a wrong default.
4. As a user, I want to see the Current Conditions for my Active Location — temperature, feels-like, a Condition label and graphic, humidity, wind, and chance of precipitation — so that I know what it's like right now.
5. As a user, I want to see an Hourly Forecast from now until local midnight at the Active Location, so that I can plan the rest of my day.
6. As a user, I want to see a multi-day Daily Forecast (today plus the next several days, up to a seven-day target) with a high/low and a Condition per day, so that I can plan ahead.
7. As a user, I want each Condition to be shown as a picture-led graphic (not just a text label), so that I can read the weather at a glance.
8. As a user, I want Condition graphics to reflect day or night at the relevant local time, so that an evening hour doesn't show a daytime sun icon.
9. As a user, I want Daily Forecast tiles to always show the day graphic for a Condition, so that day-level summaries stay visually consistent regardless of when I check.
10. As a user, I want to search for a place by name, so that I can view weather somewhere other than my detected location.
11. As a user, I want search results that are ambiguous by name (e.g. Paris, FR vs Paris, TX) to show region and country, so that I can tell them apart before selecting.
12. As a user, I want to explicitly select a Place from search results, so that the app never silently guesses which same-named place I meant.
13. As a user, selecting a Place should change what's on screen without altering my original Detected Location, so that returning to "my location" still means where I actually am.
14. As a user, I want the app to remember my last Active Location across sessions, so that reopening the app shows relevant weather immediately rather than a blank state or a fresh permission prompt.
15. As a user, I want all displayed times (current conditions, hourly, daily) to reflect the Active Location's own local timezone, so that checking a far-away place shows that place's actual day/night and hours, not my device's.
16. As a user, I want my temperature unit (°C/°F) and wind-speed unit (mph/km/h) to default from my device's locale, so that I see sensible units without configuring anything.
17. As a user in a region where temperature and speed units don't pair the "obvious" way (e.g. the UK: °C with mph), I want the app to get the default right out of the box, so that I don't have to manually correct it.
18. As a user, I want to override my temperature unit and my wind-speed unit independently, so that I can set exactly the combination I prefer regardless of locale defaults.
19. As a user, I want my unit choices to persist across sessions, so that I don't have to reset them every time I open the app.
20. As a user, I want my chosen units to apply no matter which place I'm viewing, so that viewing a foreign city doesn't switch my units to that place's local convention.
21. As a user, I want a System/Light/Dark theme preference for the app's chrome, so that the app matches my OS setting by default but I can pin a choice if I prefer.
22. As a user who has pinned a Light or Dark theme, I want that pin to persist and stop following the OS setting, so that my explicit choice sticks.
23. As a user, when the weather service call fails or I have no network connection, I want a friendly inline status message plus a non-blocking toast, so that I understand something went wrong without technical jargon or a blocked screen.
24. ~~As a user facing a hard failure (not a transient one), I want a blocking dialog only in that case, so that minor hiccups don't interrupt me but serious problems are clearly flagged.~~ **Superseded for the Weather PoC Feature** (confirmed with the user during that Feature's Spec, 2026-07-08): every fetch failure in this PoC's scope — Open-Meteo/BigDataCloud unreachable, no network — is transient, so no hard-failure case exists to trigger this dialog. Requirement kept here, not implemented, in case a future Feature introduces a genuine hard-failure case (e.g. corrupted local persistence).
25. As a user, I never want to see error codes, stack traces, or raw exception text, so that failures stay understandable.
26. As a developer, I want the weather provider's raw condition codes mapped to a small canonical Condition set at a single integration edge, so that the rest of the app never depends on provider-specific vocabulary.
27. As a developer, I want the app to use only keyless, free-tier external services, so that the installed client never has to protect an embedded secret.
28. As a developer, I want platform-specific capabilities (geolocation, OS theme signal, local persistence) reached through abstractions rather than baked into the UI layer, so that a future macOS build doesn't require a rewrite.
29. As a developer, I want every Open-Meteo API call and all unhandled exceptions logged via `Microsoft.Extensions.Logging` to a local rolling log file, so that issues can be diagnosed from the installed app without a remote log sink.
30. As a developer, I want the Open-Meteo integration seam covered by a real-IO test on at least one side, so that defects at that integration boundary are caught rather than hidden behind mocks on both sides.

## Implementation Decisions

- **Modules to be built:**
  - **`WeatherClient`** — the Open-Meteo weather edge module. Exposes a simple interface (`GetCurrentConditions`, `GetHourlyForecast`, `GetDailyForecast` for a given coordinate). Internally owns the HTTP call (via the registered `IHttpClientFactory` client), JSON parsing (`System.Text.Json`), and the mapping of Open-Meteo's raw condition codes into the app's canonical `Condition` set (ADR-0002). This is the module's deep interface: callers never see provider-specific shapes.
  - **`LocationResolver`** — resolves the Detected Location via an ordered cascade: device geolocation → IP-based geolocation → none. Exposes a single resolve operation returning either a resolved location or an explicit "unresolved" result (never a silent default). Platform-specific geolocation access sits behind this module's interface (ADR-0003), so a macOS implementation can be swapped in later without touching callers.
  - **`PlaceSearchService`** — geocoding search against Open-Meteo's keyless geocoding endpoint (ADR-0004), returning zero or more `Place` candidates (name + region + country) for a text query. Shares the same provider as `WeatherClient` but is a distinct responsibility (search vs. forecast data) and a distinct module.
  - **`UnitConventionProvider`** — resolves the default `Unit Convention` from the viewer's device locale via an owned locale→units lookup (seeded from a 3rd-party library per ADR-0001, but held as a small owned table so regional quirks like the UK can be corrected directly), and layers in any per-measurement user override. Temperature and wind-speed units are independent settings; precipitation is always a percentage.
  - **`ActiveLocationStore`** — persists and restores the Active Location across app sessions using local device storage, so a returning user sees their last place immediately, before any fresh detection runs.
  - **`ThemePreference`** — tri-state (System/Light/Dark) preference; when set to System, follows the OS light/dark signal live; when pinned, holds that choice across sessions.
  - **ViewModels** (`CommunityToolkit.Mvvm`, source-generated `ObservableObject`/`RelayCommand`) — a Weather view-model composing `WeatherClient` + `ActiveLocationStore` + `UnitConventionProvider` output into Current Conditions / Hourly / Daily display state, and a Search view-model composing `PlaceSearchService`. These are intentionally thin orchestration, not deep modules.

- **Condition → graphic mapping**: each canonical `Condition` owns exactly one day graphic and one night graphic. The day/night selection is a pure function of Condition × local time-of-day at the relevant moment (current time for Current Conditions and each Hourly entry; always "day" for Daily Forecast tiles per the domain glossary's relationships section).

- **Provider boundary**: `WeatherClient` and `PlaceSearchService` are the only modules permitted to call Open-Meteo directly; this keeps the keyless-provider constraint (ADR-0004) and the condition-mapping edge (ADR-0002) each enforced in one place.

- **Persistence**: `ActiveLocationStore`, unit overrides, and the pinned theme choice are all local-device persistence (no server, no account) — consistent with the PoC having no backend and no deployed environment.

- **Error handling surface**: transient failures (failed Open-Meteo call, no network) are surfaced by the Weather view-model as an inline status message plus a non-blocking toast (`CommunityToolkit.Maui`); a blocking modal is reserved for hard failures only, but the Weather PoC Feature has none in its confirmed scope (Requirement 24 superseded for this Feature — see above), so no blocking-modal code path is implemented by it. All user-facing copy is plain-language, with no error codes or raw exception text (Technical-Context.MD, User Feedback Approach).

- **Logging**: every Open-Meteo call (from both `WeatherClient` and `PlaceSearchService`) and all unhandled exceptions are logged via `Microsoft.Extensions.Logging` to a local rolling log file under `AppDataDirectory`, per Technical-Context.MD.

## Testing Decisions

A good test in this codebase asserts on **external behaviour** — the shape of a module's output, an observable state transition, or a side effect a caller/operator could see — never on internal implementation details or on generated/free-text content. This follows the "Test the contract, not the LLM" and "assert on observable behaviour" standards already recorded in `Technical-Context.MD`.

- **`WeatherClient`** — the headline external seam (Technical-Context.MD names it explicitly). Gets a **real-IO test on at least one side**: Tier 1 recorded-replay against real fixture responses captured from Open-Meteo, exercising the actual HTTP + JSON + condition-mapping path, on every commit. A Tier 2 live-dependency test against the real Open-Meteo endpoint runs on a bounded, scheduled basis. No mock-on-both-sides for this seam.
- **`LocationResolver`** — a platform-touching seam (device geolocation is OS-specific). Gets Tier 1 coverage for the cascade logic (device → IP → none) with fakes at the OS boundary, plus real-IO coverage of the IP-geolocation call specifically, since that leg is an external HTTP dependency like `WeatherClient`. Per the "platform matrix is non-negotiable" rule, this module's tests run on every OS target it supports (Windows now; macOS when that target lands).
- **`UnitConventionProvider` & `ActiveLocationStore`** — lower-risk than the two external seams above, but each owns real logic worth pinning: the locale→units lookup's regional-override table (`UnitConventionProvider`), and the persist/restore round-trip (`ActiveLocationStore`), including the "restore before detection re-runs" ordering from the domain glossary. Covered with Tier 1 tests against the deterministic contract (given a locale, the expected Unit Convention; given a stored location, what's restored on next launch) — no real I/O needed since there's no external service in these seams, only local persistence.
- **`PlaceSearchService`** shares its provider (and therefore its class of integration risk) with `WeatherClient`; it is not singled out with dedicated Tier 2 live-dependency coverage in this PRD, but per Technical-Context.MD's "opt-out, not opt-in" ratchet stance it still gets standard Tier 1 coverage and is not exempted outright — a deliberate trim would need to be recorded, and none has been made here.
- **ViewModels** are thin composition and are exercised indirectly through the modules above; they are not treated as an independent testing surface in this PRD.
- **Prior art**: none yet — this is the first Feature in the product, so these decisions establish the testing pattern (`xUnit` + `FluentAssertions` + `NSubstitute`, per Technical-Context.MD) that later Features will follow.

## Out of Scope

- macOS (and any other platform beyond Windows) — planned, per ADR-0003, but not this PoC's delivery target.
- Saved/favourite locations or any multi-location list — only one Active Location exists at a time (domain glossary).
- Any weather data beyond Current Conditions, Hourly Forecast, and Daily Forecast (e.g. radar maps, weather alerts/warnings, historical data, air quality, pollen).
- Any provider requiring an API key, token, or account secret (ADR-0004) — the keyless constraint is fixed for the PoC's lifetime, not just its first release.
- A backend or proxy service of any kind — the app calls Open-Meteo directly from the client.
- Push notifications, background refresh, or any weather delivery when the app isn't the foreground/open window.
- Accounts, sign-in, or any user identity — location, unit, and theme preferences are local-device only.
- Localization/translation of the app's UI text (unit and locale handling here is about measurement units, not display language).
- Automated store distribution/installer polish — this is a PoC, not a shipping product.

## Further Notes

- This PRD covers the PoC's single Feature scope end-to-end rather than being split by a Roadmap, since the product is deliberately narrow (per `Context.MD`'s framing of Weather PoC as "a proof of concept — deliberately narrow in scope"). If the product grows beyond this scope, a `Roadmap.md` breakout becomes appropriate at that point.
- The four existing ADRs (units-per-measurement, canonical-Condition-at-the-edge, cross-platform-desktop, keyless-services-only) are treated as fixed constraints throughout this PRD, per the documentation fabric's authority order in `CLAUDE.md` (ADR outranks PRD).
- No stack-level Dev commands exist yet in `CLAUDE.md`; those get filled in once implementation begins, consistent with `Technical-Context.MD` already having the frameworks (.NET 10, MAUI, MVVM) fixed.

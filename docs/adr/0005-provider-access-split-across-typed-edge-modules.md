# Provider access is split across one typed edge module per external system

**Context.** ADR-0004 decided the PoC uses only keyless, free-tier external services, and its
original text said provider access is "funnelled through the single edge module" — a sentence
that in fact conflated ADR-0004's own keyless-only decision with ADR-0002's separate decision (the
boundary that maps provider *weather* codes to canonical Conditions). As the Weather PoC Feature's
design solidified during `/feature-doc-gauntlet`'s first fix pass (2026-07-09), the Feature ended up
with **three** typed edge modules, not one: `WeatherClient` (Open-Meteo forecast), `PlaceSearchService`
(Open-Meteo geocoding), and `LocationResolver` (ip-api.com IP geolocation + BigDataCloud
reverse-geocode). That is a real, deliberate architectural decision distinct from ADR-0004's actual
subject (keyless-only), so it is recorded here as its own ADR per Technical-Context.MD Overriding
Principle #2 ("no breaking changes without an ADR"), rather than as an in-place amendment to ADR-0004.

**Decision.** Provider access is split across **one typed edge module per external system**, not
funnelled through a single shared edge module:

- `WeatherClient` ↔ Open-Meteo Forecast API (Seam 1)
- `PlaceSearchService` ↔ Open-Meteo Geocoding API (Seam 2)
- `LocationResolver` ↔ ip-api.com IP geolocation + BigDataCloud reverse-geocode-client (Seams 3, 4, 8)

Each module independently satisfies ADR-0004's keyless-only constraint — none of the three
introduces a key, token, or account secret. ADR-0002's Condition-mapping edge remains a separate,
narrower boundary that lives inside `WeatherClient` alone; it was never meant to describe every
provider integration in the app, and this ADR does not change it.

**Consequences.** Three modules to maintain instead of one, in exchange for each module having a
narrower, single-provider contract (matching the module boundaries already established in
`PRD.md`'s Implementation Decisions) instead of one module juggling three unrelated wire formats.
No change to ADR-0004's keyless-only decision or ADR-0002's Condition-mapping boundary — this ADR
only corrects and formalizes how many edge modules realize ADR-0004's decision.

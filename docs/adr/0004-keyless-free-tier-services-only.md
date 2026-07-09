# Keyless, free-tier services only — the client ships no secret

**Context.** Weather PoC is an installed desktop app (ADR-0003) calling external services
directly from the client (no proxy backend). An installed client cannot hide an API key — anything
shipped in the binary is extractable. The strongest way to remove that risk is to remove the key.

**Decision.** The PoC uses **only keyless, free-tier** external services for everything it needs —
weather (current, hourly, daily, condition codes, sunrise/sunset), place-name geocoding, and IP
geolocation. No service that requires an API key, token, or account secret may be introduced, because
the client has nowhere safe to keep one. Provider access is funnelled through the single edge module
(the same boundary that maps provider codes to Conditions, ADR-0002).

**Consequences.** Provider choice is constrained to the keyless field (e.g. Open-Meteo for weather +
geocoding), which is smaller and may offer less than key-gated vendors — an accepted trade-off for a
PoC that needs no secret management, no billing exposure, and no proxy infrastructure. If the product
ever needs a key-gated provider, that is the moment a thin proxy backend (rejected here) must be
revisited — it cannot simply be bolted onto the client. The vendor chosen to satisfy this —
**Open-Meteo** (keyless weather + geocoding) — is recorded in `Technical-Context.MD`.

---

**Amendment (2026-07-09, during the Weather PoC Feature's doc-gauntlet fix pass).** The "single
edge module" sentence above conflated this ADR's keyless-only decision with ADR-0002's separate
decision (the boundary that maps provider *weather* codes to canonical Conditions). As the Feature's
design solidified, the keyless-only decision was satisfied with **one typed edge module per external
system** — `WeatherClient` (Open-Meteo forecast), `PlaceSearchService` (Open-Meteo geocoding), and
`LocationResolver` (ip-api.com IP geolocation + BigDataCloud reverse-geocode) — matching the module
boundaries already established in `PRD.md`'s Implementation Decisions. This is consistent with, not
a breach of, this ADR's actual decision: every one of those modules calls only keyless, free-tier
services, and none introduces a key/token/secret. The superseded text is struck through rather than
deleted, so the original decision stays visible:

> ~~Provider access is funnelled through the single edge module (the same boundary that maps
> provider codes to Conditions, ADR-0002).~~ **Superseded:** provider access is split across one
> typed edge module per external system (`WeatherClient`, `PlaceSearchService`, `LocationResolver`),
> each independently satisfying the keyless-only constraint this ADR actually decides. ADR-0002's
> Condition-mapping edge remains a separate, narrower boundary inside `WeatherClient` alone — it was
> never meant to describe every provider integration in the app.

The vendor list in `Technical-Context.MD`'s "3rd-party tech" section is updated in the same pass to
add ip-api.com and BigDataCloud alongside Open-Meteo, per the Consequences section's existing
"vendor chosen ... is recorded in `Technical-Context.MD`" rule.

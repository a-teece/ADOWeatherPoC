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

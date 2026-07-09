# Weather PoC — Domain Glossary

The shared language of **Weather PoC**. This file defines *what the product's terms mean* —
not how they are built. Implementation details, specs, and decisions live elsewhere (see the
documentation fabric in `CLAUDE.md`; the stack is in `Technical-Context.MD`).

## Weather PoC
The product: a Windows desktop application (cross-platform-capable, Mac a planned later target)
that shows the weather for a Location and represents it with picture-led graphics as well as
readings. It shows the **Current Conditions** now, an **Hourly Forecast** for the rest of today,
and a multi-day **Daily Forecast**. Its defining trait is the graphical, picture-led presentation
of weather rather than a data-only readout. A proof of concept — deliberately narrow in scope.

## Language

**Detected Location**:
The app's best-effort guess at where the user physically is, resolved once when the app
opens. Resolved by trying, in order: precise device geolocation → approximate IP-based
geolocation → (if both fail) none, and the user is sent to search. May be absent.
_Avoid_: GPS location, current location (ambiguous — see Active Location)

**Active Location**:
The single place whose weather is currently shown on screen. Defaults to the Detected
Location on launch; choosing a different place re-points it. Only one exists at a time.
_Avoid_: selected location, chosen location, current location

**Place**:
A candidate location returned by a name search, shown with name + region + country so the
user can tell same-named places apart (Paris, FR vs Paris, Texas). Selecting a Place makes
it the Active Location. A Place is only a candidate until selected.
_Avoid_: result, city, search hit

**Current Conditions**:
The single right-now weather snapshot for the Active Location: temperature, feels-like,
a Condition label + graphic, humidity, wind, and chance of precipitation.
_Avoid_: now, observation, current weather (use the full term)

**Hourly Forecast**:
The hour-by-hour outlook from now until local midnight at the Active Location; each hour
carries time, temperature, a Condition graphic, and chance of precipitation.
_Avoid_: today, next 24 hours (the bound is local midnight, not a rolling 24h)

**Daily Forecast**:
The day-by-day outlook — a soft target of seven days (today + next six), but whatever the
keyless data source returns cleanly (5, 7, …) rather than contorting to hit exactly seven;
each day carries the day, a high and low temperature, and a Condition label + graphic.
_Avoid_: weekly forecast, extended forecast, 7-day forecast (the count is a target, not a promise)

**Last Refreshed**:
The timestamp of the most recent successful weather fetch (Current Conditions, Hourly Forecast,
and Daily Forecast update together as one fetch) for the Active Location, shown to the user so
stale data is never mistaken for live. Unaffected by a failed fetch — a failure leaves the
previous Last Refreshed timestamp in place rather than clearing or updating it, which is what
makes it meaningful as a staleness signal.
_Avoid_: last updated, synced, refreshed at (use the full term)

**Condition**:
The app's own canonical weather descriptor — one of a small fixed set (Clear, Partly Cloudy,
Cloudy, Fog, Drizzle, Rain, Snow, Sleet, Thunderstorm, Windy). The data source's raw codes are
mapped into this set at the edge; the rest of the app speaks only Conditions. Each Condition has
a day graphic and a night graphic (the picture-led "Weather Visual" of an earlier draft is these
graphics — not a separate concept).
_Avoid_: weather type, weather condition, weather visual, status, the provider's raw condition strings

**Unit Convention**:
The per-measurement set of display units in force — a temperature unit (°C or °F) and a
wind-speed unit (mph or km/h), which vary independently. Defaulted from the viewer's device
locale, user-overridable per measurement. Precipitation is always a percentage chance.
_Avoid_: metric, imperial (a Unit Convention is not one or the other — the UK is °C + mph)

**Theme**:
The light or dark visual styling of the app's own chrome — a three-state preference (System /
Light / Dark) that defaults to following the OS light/dark setting live, until the user pins
Light or Dark. It is a presentation preference of the app, not driven by the Condition or by
day/night, and carries no weather meaning.
_Avoid_: dark mode (as if binary — the default "System" state is the third option)

## Relationships

- On launch, the **Active Location** defaults to the **Detected Location**; if none could be
  resolved (device + IP both failed), the user is sent to search to pick one.
- A name search returns zero or more **Place** candidates; selecting one re-points the
  **Active Location**. The **Detected Location** is unchanged. Selection is always explicit —
  the app never auto-picks when more than one **Place** matches.
- Exactly one **Active Location** is shown at a time (no favourites or saved list in the PoC).
- The **Active Location** persists across sessions: on return it is restored and shown before
  detection re-runs, so a returning user never waits on a fresh permission prompt.
- The **Active Location** has exactly one **Current Conditions**, one **Hourly Forecast**, and
  one **Daily Forecast** at a time.
- All times shown (**Current Conditions**, **Hourly Forecast**, **Daily Forecast**) read in the
  **Active Location's** local timezone — never the user's device timezone — so a far-away place
  shows its own clock and its own midnight.
- The **Unit Convention** follows the viewer, not the **Active Location**: viewing Tokyo from the
  UK still renders in the viewer's chosen units.
- A displayed graphic is a function of **Condition × day-or-night** at the relevant time in the
  **Active Location**. **Daily Forecast** day-tiles always use the day graphic; **Current
  Conditions** and **Hourly Forecast** entries pick day/night from their actual local time.
- **Last Refreshed** advances only on a successful fetch. A failed fetch (surfaced per the
  Current-Conditions-area error state) leaves the prior **Last Refreshed** timestamp and the
  last-known data on screen unchanged, so the user can judge how stale what they're looking at is.

## Example dialogue

> **Dev:** "I open the app on my Bristol PC, then search 'Paris' and tap Paris, FR. Is Paris now my location?"
> **Domain expert:** "No. Bristol is still your **Detected Location** — that never changed. Paris just became the **Active Location**, the place on screen. The **Place** you tapped re-pointed it."
>
> **Dev:** "And the temperature — Paris is in France, do I see °C?"
> **Domain expert:** "You see whatever *your* **Unit Convention** says, not Paris's. You're a UK viewer, so °C and mph — the convention follows the reader, not the place. The clock and the day/night graphics, though, follow Paris."
>
> **Dev:** "What if my PC couldn't work out where I am at all?"
> **Domain expert:** "Then there's no **Detected Location** — device location failed, IP fallback failed — and we drop you straight into search to pick a **Place**. No silent default city."

## Flagged ambiguities

- "location" was used for both the device's physical place and the place on screen —
  resolved: **Detected Location** (physical, resolved once) vs **Active Location** (displayed, re-pointable).
- An earlier glossary draft scoped the product to **current conditions only** ("never a prediction
  of future weather"). Resolved (human decision, 2026-06-29): forecasting **is** in scope — the
  **Hourly Forecast** and **Daily Forecast** are core, alongside **Current Conditions**.
- An earlier draft used **Weather Condition** and **Weather Visual** as separate terms. Resolved:
  the canonical term is **Condition**, and its picture is the Condition's day/night **graphic** —
  not a standalone concept.

# Units are per-measurement, locale-defaulted with regional overrides — not a metric/imperial bundle

**Context.** Weather PoC renders many figures (temperature, feels-like, highs/lows, wind) for a
viewer who can look at places anywhere in the world. The obvious design is a single
"Metric vs Imperial" toggle. That is wrong: the UK uses °C for temperature but mph for speed,
so temperature unit and speed unit do not move together. Bundling them produces incorrect
defaults for whole countries.

**Decision.** Model display units as **independent per-measurement dimensions** — a temperature
unit (°C / °F) and a wind-speed unit (mph / km/h) that vary separately (precipitation is always a
% chance). The default is derived from the viewer's device locale via a **Unit Convention**: a
locale→units mapping, seeded from a 3rd-party library but held as a small owned lookup so regional
quirks (UK = °C + mph, and others) can be corrected in one place rather than trusting the library
blindly. The user can override **each measurement independently** (not via a bundled switch), and
the choice persists across sessions. Units follow the viewer, not the place being viewed.

**Consequences.** Anything that renders a number must take its unit from the relevant per-measurement
setting, never from a single "system" flag — a future reader seeing °C paired with mph for a UK user
should recognise it as deliberate, not a bug. The specific locale-detection library is deferred to
`/init-tech-context` / the Spec, since the stack is not yet chosen; this ADR fixes only the model and
the regional-override requirement.

# App owns a canonical Condition set; provider codes are mapped at the edge

**Context.** Weather data sources each describe the sky in their own vocabulary — free-text
strings, numeric codes, differing granularity. If the UI and forecast logic render directly off a
provider's raw codes, the app is welded to that provider: every icon, label, and day/night rule
breaks the day we change source, and there is no clean internal language for "the weather."

**Decision.** The app defines its own small canonical **Condition** set (Clear, Partly Cloudy,
Cloudy, Fog, Drizzle, Rain, Snow, Sleet, Thunderstorm, Windy). The data source's raw codes are
translated into this set at the integration edge (an anti-corruption boundary); the rest of the
app speaks only Conditions. Each Condition owns a day graphic and a night graphic.

**Consequences.** Swapping or adding a weather provider is a one-table remapping job — nothing
downstream moves. A future reader should not "simplify" by rendering provider strings directly:
the indirection is deliberate insulation, not accidental ceremony. The canonical set is small by
design; provider conditions outside it map to the nearest member.

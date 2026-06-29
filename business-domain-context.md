# Domain Glossary

The shared language of **Weather PoC**. This file defines *what the product's terms mean*
— not how they are built. Implementation details, specs, and decisions live elsewhere
(see the documentation fabric in `CLAUDE.md`).

## Weather PoC
The product: a Windows desktop application that shows a user the Current Conditions for a
Location and represents them with a Weather Visual. Its defining trait is the graphical,
picture-led presentation of weather rather than a data-only readout. A proof of concept —
deliberately narrow in scope.

## Location
A geographic place that Current Conditions are shown for. The user's own location is the
default Location the app opens on; viewing other Locations (places the user chooses to look
up) is an additional, optional capability rather than the core flow. A Location is the
*subject* of the weather shown, not the weather itself.

## Current Conditions
The present weather at a Location at the moment it is shown — the prevailing Weather
Condition together with its associated readings (such as temperature). Not a forecast:
Weather PoC describes the weather *now*, never a prediction of future weather.

## Weather Condition
The categorical descriptor of the weather at a Location — for example sunny, cloudy, rain,
or snow. It is the *category* of weather, distinct from the numeric readings (like
temperature) that accompany it in the Current Conditions, and distinct from the Weather
Visual that depicts it. Each Weather Condition is what a Weather Visual is chosen to
represent.

## Weather Visual
The graphical, picture-led representation of a Weather Condition — the "pretty picture" a
user sees instead of, or alongside, raw numbers. A Weather Visual depicts a Weather
Condition; it is the presentation of the weather, not the weather data itself.

## Theme
The light or dark visual styling of the application, following the Windows system light/dark
setting. A Theme is a presentation preference of the app's own chrome; it is not driven by
the Weather Condition or by time of day, and carries no day/night weather meaning.

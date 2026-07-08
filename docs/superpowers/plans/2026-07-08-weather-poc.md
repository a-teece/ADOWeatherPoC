# Weather PoC Implementation Plan

> **For agentic workers:** Do NOT implement this plan directly. It must first pass `/feature-doc-gauntlet` in a clean session, then be broken into stories by `/enate-to-stories`; AFK implementation happens per-story from there. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Weather PoC .NET MAUI app end-to-end: location detection/search, Current/Hourly/Daily forecasts with picture-led Condition graphics, per-measurement units, and a System/Light/Dark theme — all from keyless, free-tier services.

**Architecture:** A `WeatherPoc.Core` class library holds all deep modules and domain types behind small interfaces (no MAUI dependency), so they're testable with plain xUnit. A `WeatherPoc` MAUI app project (Windows-first TFM, per ADR-0003) supplies MAUI-backed implementations of the platform-touching interfaces (device geolocation, local preferences, OS theme signal) and hosts three thin ViewModels (`WeatherPageViewModel`, `PlaceSearchViewModel`, `SettingsPageViewModel`) composed via DI, communicating cross-page state through `CommunityToolkit.Mvvm`'s `WeakReferenceMessenger`.

**Tech Stack:** .NET 10, .NET MAUI (Windows TFM `net10.0-windows10.0.19041.0`), `CommunityToolkit.Mvvm`, `CommunityToolkit.Maui`, `System.Text.Json`, `Microsoft.Extensions.DependencyInjection`/`Logging` (via the MAUI host builder), `Microsoft.Maui.Storage.Preferences`, `Microsoft.Maui.Devices.Sensors.Geolocation`; tests via `xUnit`, `FluentAssertions`, `NSubstitute` — all per `Technical-Context.MD` → Packages in use.

**Context references:**
- Spec: `docs/superpowers/specs/2026-07-08-weather-poc-design.md`
- `Context.MD`
- `Technical-Context.MD` (Overriding Principles that apply: #3 every-seam-gets-a-real-IO-test; #4 pinned dependencies)
- ADRs: `docs/adr/0001-units-are-per-measurement-not-a-metric-imperial-bundle.md`, `docs/adr/0002-canonical-condition-set-mapped-at-the-edge.md`, `docs/adr/0003-cross-platform-desktop-windows-first-mac-later.md`, `docs/adr/0004-keyless-free-tier-services-only.md`

> An AFK Developer Agent picking up this plan MUST load every file in the Context references block before writing code.

---

## File Structure

```
src/
  WeatherPoc.Core/                          # class library, net10.0, no MAUI dependency
    WeatherPoc.Core.csproj
    Domain/
      Condition.cs                          # enum + WMO-code mapping table (ADR-0002)
      Coordinates.cs
      Place.cs
      TemperatureUnit.cs / WindSpeedUnit.cs / UnitConvention.cs   (ADR-0001)
      CurrentConditions.cs / HourlyForecastEntry.cs / DailyForecastEntry.cs
      DetectedLocation.cs / ActiveLocation.cs
      AppThemeChoice.cs                     # System/Light/Dark
    Weather/
      IWeatherClient.cs / WeatherClient.cs               # Seam 1
    Search/
      IPlaceSearchService.cs / PlaceSearchService.cs     # Seam 2
    Location/
      IIpGeolocationClient.cs / BigDataCloudIpGeolocationClient.cs  # Seam 3
      IDeviceGeolocation.cs                              # Seam 4 (impl lives in app project)
      LocationResolver.cs
      IActiveLocationStore.cs / ActiveLocationStore.cs   # Seam 5
    Settings/
      IPreferencesStore.cs                               # Seam 5 (impl lives in app project)
      UnitConventionProvider.cs
      IThemeSignal.cs                                    # Seam 6 (impl lives in app project)
      ThemePreference.cs
  WeatherPoc/                                # MAUI app project (Windows TFM)
    WeatherPoc.csproj
    MauiProgram.cs
    Platform/
      MauiDeviceGeolocation.cs               # IDeviceGeolocation over Microsoft.Maui.Devices.Sensors.Geolocation
      MauiPreferencesStore.cs                # IPreferencesStore over Microsoft.Maui.Storage.Preferences
      MauiThemeSignal.cs                     # IThemeSignal over AppInfo.Current.RequestedTheme
    Messaging/
      SettingsChangedMessage.cs / PlaceSelectedMessage.cs
    ViewModels/
      WeatherPageViewModel.cs
      PlaceSearchViewModel.cs
      SettingsPageViewModel.cs
    Views/
      WeatherPage.xaml(.cs)
      SearchOverlay.xaml(.cs)
      SettingsPage.xaml(.cs)
    App.xaml(.cs) / AppShell.xaml(.cs)
tests/
  WeatherPoc.Core.Tests/                     # xUnit, references WeatherPoc.Core
    WeatherPoc.Core.Tests.csproj
    Fixtures/
      open-meteo-forecast-response.json      # captured Seam 1 payload
      open-meteo-geocoding-response.json      # captured Seam 2 populated-results payload
      open-meteo-geocoding-zero-match-response.json  # captured Seam 2 zero-match payload
      bigdatacloud-ip-geolocation-response.json      # captured Seam 3 payload
    Weather/WeatherClientTests.cs
    Search/PlaceSearchServiceTests.cs
    Location/BigDataCloudIpGeolocationClientTests.cs
    Location/LocationResolverTests.cs
    Location/ActiveLocationStoreTests.cs
    Settings/UnitConventionProviderTests.cs
    Settings/ThemePreferenceTests.cs
  WeatherPoc.Tests/                          # xUnit, references WeatherPoc (ViewModels)
    WeatherPoc.Tests.csproj
    ViewModels/WeatherPageViewModelTests.cs
```

Rationale: every deep module and its interface lives in `WeatherPoc.Core`, which has **zero MAUI package reference** — this is what makes Tasks 3–10 pure `dotnet test`-able without a Windows/MAUI workload installed, and is the concrete mechanism behind ADR-0003's "platform-specific capabilities reached through abstractions" rule. The three `Platform/*` classes in the app project are the only code that touches a MAUI platform API directly.

---

### Task 1: Solution & project scaffolding

**Files:**
- Create: `WeatherPoc.sln`
- Create: `src/WeatherPoc.Core/WeatherPoc.Core.csproj`
- Create: `src/WeatherPoc/WeatherPoc.csproj`
- Create: `tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj`
- Create: `tests/WeatherPoc.Tests/WeatherPoc.Tests.csproj`

- [ ] **Step 1: Create the class library**

Run:
```bash
dotnet new classlib -n WeatherPoc.Core -o src/WeatherPoc.Core -f net10.0
```

- [ ] **Step 2: Create the MAUI app project**

Run:
```bash
dotnet new maui -n WeatherPoc -o src/WeatherPoc
```

Edit `src/WeatherPoc/WeatherPoc.csproj` so `<TargetFrameworks>` contains only the Windows TFM (Windows-first per ADR-0003; other TFMs are added when that target is scheduled):

```xml
<TargetFrameworks>net10.0-windows10.0.19041.0</TargetFrameworks>
<TargetPlatformMinVersion>10.0.17763.0</TargetPlatformMinVersion>
```

- [ ] **Step 3: Create the two test projects**

Run:
```bash
dotnet new xunit -n WeatherPoc.Core.Tests -o tests/WeatherPoc.Core.Tests -f net10.0
dotnet new xunit -n WeatherPoc.Tests -o tests/WeatherPoc.Tests -f net10.0
```

- [ ] **Step 4: Add project references**

Run:
```bash
dotnet add src/WeatherPoc/WeatherPoc.csproj reference src/WeatherPoc.Core/WeatherPoc.Core.csproj
dotnet add tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj reference src/WeatherPoc.Core/WeatherPoc.Core.csproj
dotnet add tests/WeatherPoc.Tests/WeatherPoc.Tests.csproj reference src/WeatherPoc/WeatherPoc.csproj
```

- [ ] **Step 5: Add pinned NuGet packages (Technical-Context.MD Overriding Principle #4 — exact versions, never floating)**

Run:
```bash
dotnet add src/WeatherPoc/WeatherPoc.csproj package CommunityToolkit.Mvvm --version 8.4.0
dotnet add src/WeatherPoc/WeatherPoc.csproj package CommunityToolkit.Maui --version 9.1.0

dotnet add tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj package FluentAssertions --version 6.12.1
dotnet add tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj package NSubstitute --version 5.3.0

dotnet add tests/WeatherPoc.Tests/WeatherPoc.Tests.csproj package FluentAssertions --version 6.12.1
dotnet add tests/WeatherPoc.Tests/WeatherPoc.Tests.csproj package NSubstitute --version 5.3.0
```

- [ ] **Step 6: Create the solution file and add every project**

Run:
```bash
dotnet new sln -n WeatherPoc
dotnet sln add src/WeatherPoc.Core/WeatherPoc.Core.csproj
dotnet sln add src/WeatherPoc/WeatherPoc.csproj
dotnet sln add tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj
dotnet sln add tests/WeatherPoc.Tests/WeatherPoc.Tests.csproj
```

- [ ] **Step 7: Verify the empty solution builds and tests run**

Run: `dotnet build WeatherPoc.sln`
Expected: `Build succeeded.`

Run: `dotnet test tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj`
Expected: the template's one sample test passes (delete it in the next task).

- [ ] **Step 8: Commit**

```bash
git add WeatherPoc.sln src/ tests/
git commit -m "chore: scaffold WeatherPoc solution (Core lib, MAUI app, two test projects)"
```

---

### Task 2: Domain types (`Condition`, `UnitConvention`, forecast records)

**Files:**
- Create: `src/WeatherPoc.Core/Domain/Condition.cs`
- Create: `src/WeatherPoc.Core/Domain/Coordinates.cs`
- Create: `src/WeatherPoc.Core/Domain/Place.cs`
- Create: `src/WeatherPoc.Core/Domain/TemperatureUnit.cs`
- Create: `src/WeatherPoc.Core/Domain/WindSpeedUnit.cs`
- Create: `src/WeatherPoc.Core/Domain/UnitConvention.cs`
- Create: `src/WeatherPoc.Core/Domain/CurrentConditions.cs`
- Create: `src/WeatherPoc.Core/Domain/HourlyForecastEntry.cs`
- Create: `src/WeatherPoc.Core/Domain/DailyForecastEntry.cs`
- Create: `src/WeatherPoc.Core/Domain/DetectedLocation.cs`
- Create: `src/WeatherPoc.Core/Domain/ActiveLocation.cs`
- Create: `src/WeatherPoc.Core/Domain/AppThemeChoice.cs`
- Test: `tests/WeatherPoc.Core.Tests/Domain/ConditionTests.cs`

The canonical `Condition` set and its WMO-code mapping (ADR-0002) is the one piece of real logic in this task, so it gets a test; the rest are plain records with no branching logic.

- [ ] **Step 1: Write the failing test for WMO → Condition mapping**

```csharp
// tests/WeatherPoc.Core.Tests/Domain/ConditionTests.cs
using FluentAssertions;
using WeatherPoc.Core.Domain;
using Xunit;

namespace WeatherPoc.Core.Tests.Domain;

public class ConditionTests
{
    [Theory]
    [InlineData(0, Condition.Clear)]
    [InlineData(1, Condition.PartlyCloudy)]
    [InlineData(2, Condition.PartlyCloudy)]
    [InlineData(3, Condition.Cloudy)]
    [InlineData(45, Condition.Fog)]
    [InlineData(48, Condition.Fog)]
    [InlineData(51, Condition.Drizzle)]
    [InlineData(55, Condition.Drizzle)]
    [InlineData(61, Condition.Rain)]
    [InlineData(65, Condition.Rain)]
    [InlineData(71, Condition.Snow)]
    [InlineData(75, Condition.Snow)]
    [InlineData(66, Condition.Sleet)]
    [InlineData(67, Condition.Sleet)]
    [InlineData(95, Condition.Thunderstorm)]
    [InlineData(99, Condition.Thunderstorm)]
    public void FromWmoCode_maps_known_codes_to_the_canonical_condition(int wmoCode, Condition expected)
    {
        Condition.FromWmoCode(wmoCode).Should().Be(expected);
    }

    [Fact]
    public void FromWmoCode_maps_an_unrecognised_code_to_the_nearest_member_Cloudy()
    {
        // ADR-0002: "provider conditions outside it map to the nearest member" —
        // an unknown/future WMO code defaults to Cloudy rather than throwing.
        Condition.FromWmoCode(9999).Should().Be(Condition.Cloudy);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter ConditionTests`
Expected: FAIL — `Condition` does not exist yet.

- [ ] **Step 3: Implement `Condition` and the remaining domain types**

```csharp
// src/WeatherPoc.Core/Domain/Condition.cs
namespace WeatherPoc.Core.Domain;

/// <summary>
/// The app's own canonical weather descriptor (ADR-0002). Provider codes are
/// mapped into this set at the edge — nothing outside <see cref="WeatherClient"/>
/// should see a raw WMO code.
/// </summary>
public enum Condition
{
    Clear,
    PartlyCloudy,
    Cloudy,
    Fog,
    Drizzle,
    Rain,
    Snow,
    Sleet,
    Thunderstorm,
    Windy
}

public static class ConditionMapping
{
    public static Condition FromWmoCode(int wmoCode) => wmoCode switch
    {
        0 => Condition.Clear,
        1 or 2 => Condition.PartlyCloudy,
        3 => Condition.Cloudy,
        45 or 48 => Condition.Fog,
        51 or 53 or 55 or 56 or 57 => Condition.Drizzle,
        61 or 63 or 65 or 80 or 81 or 82 => Condition.Rain,
        71 or 73 or 75 or 77 or 85 or 86 => Condition.Snow,
        66 or 67 => Condition.Sleet,
        95 or 96 or 99 => Condition.Thunderstorm,
        _ => Condition.Cloudy // ADR-0002: unrecognised codes map to the nearest member
    };
}
```

Add the static method onto the enum's companion type by extending the test call site — since C# enums can't carry static methods directly, expose it as `Condition.FromWmoCode` via a partial-static pattern: replace the enum test calls with `ConditionMapping.FromWmoCode`. Update Step 1's test file to call `ConditionMapping.FromWmoCode(...)` instead of `Condition.FromWmoCode(...)` before proceeding.

```csharp
// src/WeatherPoc.Core/Domain/Coordinates.cs
namespace WeatherPoc.Core.Domain;

public sealed record Coordinates(double Latitude, double Longitude);
```

```csharp
// src/WeatherPoc.Core/Domain/Place.cs
namespace WeatherPoc.Core.Domain;

public sealed record Place(string Name, string? Region, string Country, Coordinates Coordinates);
```

```csharp
// src/WeatherPoc.Core/Domain/TemperatureUnit.cs
namespace WeatherPoc.Core.Domain;

public enum TemperatureUnit { Celsius, Fahrenheit }
```

```csharp
// src/WeatherPoc.Core/Domain/WindSpeedUnit.cs
namespace WeatherPoc.Core.Domain;

public enum WindSpeedUnit { Mph, Kmh }
```

```csharp
// src/WeatherPoc.Core/Domain/UnitConvention.cs
namespace WeatherPoc.Core.Domain;

// ADR-0001: temperature and wind-speed units vary independently — never a single "system" flag.
public sealed record UnitConvention(TemperatureUnit Temperature, WindSpeedUnit WindSpeed);
```

```csharp
// src/WeatherPoc.Core/Domain/CurrentConditions.cs
namespace WeatherPoc.Core.Domain;

public sealed record CurrentConditions(
    DateTimeOffset LocalTime,
    double TemperatureCelsius,
    double FeelsLikeCelsius,
    Condition Condition,
    bool IsDay,
    int HumidityPercent,
    double WindSpeedKmh,
    int PrecipitationChancePercent);
```

```csharp
// src/WeatherPoc.Core/Domain/HourlyForecastEntry.cs
namespace WeatherPoc.Core.Domain;

public sealed record HourlyForecastEntry(
    DateTimeOffset LocalTime,
    double TemperatureCelsius,
    Condition Condition,
    bool IsDay,
    int PrecipitationChancePercent);
```

```csharp
// src/WeatherPoc.Core/Domain/DailyForecastEntry.cs
namespace WeatherPoc.Core.Domain;

public sealed record DailyForecastEntry(
    DateOnly LocalDate,
    double HighCelsius,
    double LowCelsius,
    Condition Condition);
```

```csharp
// src/WeatherPoc.Core/Domain/DetectedLocation.cs
namespace WeatherPoc.Core.Domain;

/// <summary>Result of a <see cref="LocationResolver"/> cascade. Null <see cref="Place"/> means unresolved.</summary>
public sealed record DetectedLocation(Place? Place);
```

```csharp
// src/WeatherPoc.Core/Domain/ActiveLocation.cs
namespace WeatherPoc.Core.Domain;

public sealed record ActiveLocation(string Name, string? Region, string Country, Coordinates Coordinates, string IanaTimeZoneId);
```

```csharp
// src/WeatherPoc.Core/Domain/AppThemeChoice.cs
namespace WeatherPoc.Core.Domain;

public enum AppThemeChoice { System, Light, Dark }
```

- [ ] **Step 4: Update the test to call `ConditionMapping.FromWmoCode` and run it**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter ConditionTests`
Expected: PASS (all 15 theory cases + the unknown-code case).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherPoc.Core/Domain tests/WeatherPoc.Core.Tests/Domain
git commit -m "feat: add domain types and canonical Condition WMO mapping (ADR-0002)"
```

---

### Task 3: `WeatherClient` — Seam 1 (Open-Meteo Forecast API)

**Files:**
- Create: `src/WeatherPoc.Core/Weather/IWeatherClient.cs`
- Create: `src/WeatherPoc.Core/Weather/WeatherClient.cs`
- Create: `tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-forecast-response.json`
- Test: `tests/WeatherPoc.Core.Tests/Weather/WeatherClientTests.cs`

Seam 1 contract under test (Spec § Seam inventory): no auth required; `current`/`hourly`/`daily` blocks always requested and always present; `weather_code` (WMO int) and `is_day` (0/1 int, current block) drive `Condition` + day/night.

- [ ] **Step 1: Save the captured fixture payload**

```json
// tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-forecast-response.json
{
  "latitude": 51.5,
  "longitude": -2.5,
  "generationtime_ms": 0.67138671875,
  "utc_offset_seconds": 3600,
  "timezone": "Europe/London",
  "timezone_abbreviation": "GMT+1",
  "elevation": 11.0,
  "current_units": {
    "time": "iso8601",
    "interval": "seconds",
    "temperature_2m": "°C",
    "weather_code": "wmo code",
    "is_day": ""
  },
  "current": {
    "time": "2026-07-08T16:00",
    "interval": 900,
    "temperature_2m": 30.4,
    "weather_code": 0,
    "is_day": 1
  },
  "hourly_units": {
    "time": "iso8601",
    "temperature_2m": "°C",
    "weather_code": "wmo code"
  },
  "hourly": {
    "time": ["2026-07-08T00:00", "2026-07-08T01:00", "2026-07-08T02:00"],
    "temperature_2m": [19.6, 19.2, 18.7],
    "weather_code": [0, 0, 0]
  },
  "daily_units": {
    "time": "iso8601",
    "weather_code": "wmo code",
    "temperature_2m_max": "°C",
    "temperature_2m_min": "°C"
  },
  "daily": {
    "time": ["2026-07-08", "2026-07-09", "2026-07-10"],
    "weather_code": [3, 3, 3],
    "temperature_2m_max": [30.8, 31.4, 33.6],
    "temperature_2m_min": [17.7, 17.2, 20.3]
  }
}
```

This fixture is verbatim the payload observed live from `api.open-meteo.com/v1/forecast` during brainstorming (2026-07-08) — captured evidence, not a hand-written guess (Seam 1 (d) proof / anti-pattern: never a self-written mock payload).

- [ ] **Step 2: Write the failing recorded-replay test**

```csharp
// tests/WeatherPoc.Core.Tests/Weather/WeatherClientTests.cs
using System.Net;
using FluentAssertions;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Weather;
using Xunit;

namespace WeatherPoc.Core.Tests.Weather;

public class WeatherClientTests
{
    private static HttpClient FixtureHttpClient(string fixtureRelativePath)
    {
        var json = File.ReadAllText(Path.Combine(AppContext.BaseDirectory, fixtureRelativePath));
        var handler = new StubHttpMessageHandler(_ => new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(json, System.Text.Encoding.UTF8, "application/json")
        });
        return new HttpClient(handler) { BaseAddress = new Uri("https://api.open-meteo.com") };
    }

    [Fact]
    public async Task GetForecastAsync_parses_current_hourly_and_daily_blocks_from_the_captured_payload()
    {
        var http = FixtureHttpClient("Fixtures/open-meteo-forecast-response.json");
        var sut = new WeatherClient(http);

        var result = await sut.GetForecastAsync(new Coordinates(51.4545, -2.5879), "Europe/London");

        result.Current.TemperatureCelsius.Should().Be(30.4);
        result.Current.Condition.Should().Be(Condition.Clear); // weather_code 0
        result.Current.IsDay.Should().BeTrue();                // is_day 1

        result.Hourly.Should().HaveCount(3);
        result.Hourly[0].TemperatureCelsius.Should().Be(19.6);
        result.Hourly[0].Condition.Should().Be(Condition.Clear);

        result.Daily.Should().HaveCount(3);
        result.Daily[0].HighCelsius.Should().Be(30.8);
        result.Daily[0].LowCelsius.Should().Be(17.7);
        result.Daily[0].Condition.Should().Be(Condition.Cloudy); // weather_code 3
    }
}

/// <summary>Minimal real-HTTP-pipeline stub: exercises the actual HttpClient send path against a canned response, per Technical-Context.MD's "no mock-on-both-sides" rule — the JSON deserialization and mapping code are real, only the network transport is stubbed.</summary>
public sealed class StubHttpMessageHandler(Func<HttpRequestMessage, HttpResponseMessage> respond) : HttpMessageHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        => Task.FromResult(respond(request));
}
```

- [ ] **Step 3: Add the fixture to the test project as copied output**

Edit `tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj`, add inside the existing `<Project>`:

```xml
<ItemGroup>
  <None Include="Fixtures\**\*.json" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

- [ ] **Step 4: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter WeatherClientTests`
Expected: FAIL — `WeatherClient` / `IWeatherClient` do not exist.

- [ ] **Step 5: Implement `IWeatherClient` and `WeatherClient`**

```csharp
// src/WeatherPoc.Core/Weather/IWeatherClient.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Weather;

public interface IWeatherClient
{
    Task<Forecast> GetForecastAsync(Coordinates coordinates, string ianaTimeZoneId, CancellationToken cancellationToken = default);
}

public sealed record Forecast(CurrentConditions Current, IReadOnlyList<HourlyForecastEntry> Hourly, IReadOnlyList<DailyForecastEntry> Daily);
```

```csharp
// src/WeatherPoc.Core/Weather/WeatherClient.cs
using System.Globalization;
using System.Text.Json;
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Weather;

/// <summary>
/// The Open-Meteo forecast edge (Seam 1 of the Weather PoC design spec). No API key —
/// Open-Meteo requires none for non-commercial use (ADR-0004). Maps raw WMO weather
/// codes to the canonical <see cref="Condition"/> set at this boundary (ADR-0002); no
/// other type in the app should see a raw weather_code.
/// </summary>
public sealed class WeatherClient(HttpClient httpClient) : IWeatherClient
{
    public async Task<Forecast> GetForecastAsync(Coordinates coordinates, string ianaTimeZoneId, CancellationToken cancellationToken = default)
    {
        var url = $"/v1/forecast" +
                  $"?latitude={coordinates.Latitude.ToString(CultureInfo.InvariantCulture)}" +
                  $"&longitude={coordinates.Longitude.ToString(CultureInfo.InvariantCulture)}" +
                  $"&current=temperature_2m,apparent_temperature,weather_code,is_day,relative_humidity_2m,wind_speed_10m,precipitation_probability" +
                  $"&hourly=temperature_2m,weather_code,precipitation_probability" +
                  $"&daily=weather_code,temperature_2m_max,temperature_2m_min" +
                  $"&timezone={Uri.EscapeDataString(ianaTimeZoneId)}" +
                  $"&forecast_days=7";

        using var response = await httpClient.GetAsync(url, cancellationToken);
        response.EnsureSuccessStatusCode();

        await using var stream = await response.Content.ReadAsStreamAsync(cancellationToken);
        using var document = await JsonDocument.ParseAsync(stream, cancellationToken: cancellationToken);
        var root = document.RootElement;

        return new Forecast(
            ParseCurrent(root.GetProperty("current")),
            ParseHourly(root.GetProperty("hourly")),
            ParseDaily(root.GetProperty("daily")));
    }

    private static CurrentConditions ParseCurrent(JsonElement current)
    {
        var isDay = current.GetProperty("is_day").GetInt32() == 1;
        return new CurrentConditions(
            LocalTime: DateTimeOffset.Parse(current.GetProperty("time").GetString()!, CultureInfo.InvariantCulture),
            TemperatureCelsius: current.GetProperty("temperature_2m").GetDouble(),
            FeelsLikeCelsius: current.TryGetProperty("apparent_temperature", out var feelsLike) ? feelsLike.GetDouble() : current.GetProperty("temperature_2m").GetDouble(),
            Condition: ConditionMapping.FromWmoCode(current.GetProperty("weather_code").GetInt32()),
            IsDay: isDay,
            HumidityPercent: current.TryGetProperty("relative_humidity_2m", out var humidity) ? humidity.GetInt32() : 0,
            WindSpeedKmh: current.TryGetProperty("wind_speed_10m", out var wind) ? wind.GetDouble() : 0,
            PrecipitationChancePercent: current.TryGetProperty("precipitation_probability", out var precip) ? precip.GetInt32() : 0);
    }

    private static IReadOnlyList<HourlyForecastEntry> ParseHourly(JsonElement hourly)
    {
        var times = hourly.GetProperty("time");
        var temps = hourly.GetProperty("temperature_2m");
        var codes = hourly.GetProperty("weather_code");
        var hasPrecip = hourly.TryGetProperty("precipitation_probability", out var precipArray);

        var entries = new List<HourlyForecastEntry>(times.GetArrayLength());
        for (var i = 0; i < times.GetArrayLength(); i++)
        {
            var localTime = DateTimeOffset.Parse(times[i].GetString()!, CultureInfo.InvariantCulture);
            entries.Add(new HourlyForecastEntry(
                LocalTime: localTime,
                TemperatureCelsius: temps[i].GetDouble(),
                Condition: ConditionMapping.FromWmoCode(codes[i].GetInt32()),
                IsDay: localTime.Hour is >= 6 and < 20, // refined against sunrise/sunset in a later story if needed
                PrecipitationChancePercent: hasPrecip ? precipArray[i].GetInt32() : 0));
        }
        return entries;
    }

    private static IReadOnlyList<DailyForecastEntry> ParseDaily(JsonElement daily)
    {
        var dates = daily.GetProperty("time");
        var codes = daily.GetProperty("weather_code");
        var highs = daily.GetProperty("temperature_2m_max");
        var lows = daily.GetProperty("temperature_2m_min");

        var entries = new List<DailyForecastEntry>(dates.GetArrayLength());
        for (var i = 0; i < dates.GetArrayLength(); i++)
        {
            entries.Add(new DailyForecastEntry(
                LocalDate: DateOnly.Parse(dates[i].GetString()!, CultureInfo.InvariantCulture),
                HighCelsius: highs[i].GetDouble(),
                LowCelsius: lows[i].GetDouble(),
                Condition: ConditionMapping.FromWmoCode(codes[i].GetInt32())));
        }
        return entries;
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter WeatherClientTests`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/WeatherPoc.Core/Weather tests/WeatherPoc.Core.Tests/Weather tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-forecast-response.json tests/WeatherPoc.Core.Tests/WeatherPoc.Core.Tests.csproj
git commit -m "feat: add WeatherClient against captured Open-Meteo forecast fixture (Seam 1)"
```

---

### Task 4: `PlaceSearchService` — Seam 2 (Open-Meteo Geocoding API)

**Files:**
- Create: `src/WeatherPoc.Core/Search/IPlaceSearchService.cs`
- Create: `src/WeatherPoc.Core/Search/PlaceSearchService.cs`
- Create: `tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-geocoding-response.json`
- Create: `tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-geocoding-zero-match-response.json`
- Test: `tests/WeatherPoc.Core.Tests/Search/PlaceSearchServiceTests.cs`

Seam 2's proven contract: on zero matches the `results` key is **entirely absent** from the JSON — not `null`, not `[]`. Both shapes get a fixture and a named test case.

- [ ] **Step 1: Save both captured fixtures**

```json
// tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-geocoding-response.json
{
  "results": [
    { "name": "Paris", "country": "France", "admin1": "Ile-de-France", "latitude": 48.85341, "longitude": 2.3488 },
    { "name": "Paris", "country": "United States", "admin1": "Texas", "latitude": 33.66094, "longitude": -95.55551 }
  ],
  "generationtime_ms": 0.5
}
```

```json
// tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-geocoding-zero-match-response.json
{ "generationtime_ms": 0.5147457 }
```

The zero-match fixture is verbatim the payload observed live from `geocoding-api.open-meteo.com/v1/search?name=zzzzznonexistentplacexyz123` during brainstorming (2026-07-08).

- [ ] **Step 2: Write the failing tests for both shapes**

```csharp
// tests/WeatherPoc.Core.Tests/Search/PlaceSearchServiceTests.cs
using System.Net;
using FluentAssertions;
using WeatherPoc.Core.Search;
using WeatherPoc.Core.Tests.Weather;
using Xunit;

namespace WeatherPoc.Core.Tests.Search;

public class PlaceSearchServiceTests
{
    private static HttpClient FixtureHttpClient(string fixtureRelativePath)
    {
        var json = File.ReadAllText(Path.Combine(AppContext.BaseDirectory, fixtureRelativePath));
        var handler = new StubHttpMessageHandler(_ => new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(json, System.Text.Encoding.UTF8, "application/json")
        });
        return new HttpClient(handler) { BaseAddress = new Uri("https://geocoding-api.open-meteo.com") };
    }

    [Fact]
    public async Task SearchAsync_returns_disambiguating_candidates_for_a_matching_name()
    {
        var sut = new PlaceSearchService(FixtureHttpClient("Fixtures/open-meteo-geocoding-response.json"));

        var results = await sut.SearchAsync("Paris");

        results.Should().HaveCount(2);
        results[0].Name.Should().Be("Paris");
        results[0].Region.Should().Be("Ile-de-France");
        results[0].Country.Should().Be("France");
        results[1].Region.Should().Be("Texas");
    }

    [Fact]
    public async Task SearchAsync_returns_an_empty_list_when_the_results_key_is_absent()
    {
        // Seam 2 (c): zero matches omits "results" entirely — this must not throw.
        var sut = new PlaceSearchService(FixtureHttpClient("Fixtures/open-meteo-geocoding-zero-match-response.json"));

        var results = await sut.SearchAsync("zzzzznonexistentplacexyz123");

        results.Should().BeEmpty();
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter PlaceSearchServiceTests`
Expected: FAIL — `PlaceSearchService` does not exist.

- [ ] **Step 4: Implement `IPlaceSearchService` and `PlaceSearchService`**

```csharp
// src/WeatherPoc.Core/Search/IPlaceSearchService.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Search;

public interface IPlaceSearchService
{
    Task<IReadOnlyList<Place>> SearchAsync(string query, CancellationToken cancellationToken = default);
}
```

```csharp
// src/WeatherPoc.Core/Search/PlaceSearchService.cs
using System.Text.Json;
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Search;

/// <summary>
/// The Open-Meteo geocoding edge (Seam 2). Shares Open-Meteo's keyless auth decision
/// with <see cref="Weather.WeatherClient"/> (Seam 1) — no API key. A zero-match response
/// omits the "results" key entirely, which this parses as an empty list, not an error.
/// </summary>
public sealed class PlaceSearchService(HttpClient httpClient) : IPlaceSearchService
{
    public async Task<IReadOnlyList<Place>> SearchAsync(string query, CancellationToken cancellationToken = default)
    {
        var url = $"/v1/search?name={Uri.EscapeDataString(query)}&count=10&language=en";

        using var response = await httpClient.GetAsync(url, cancellationToken);
        response.EnsureSuccessStatusCode();

        await using var stream = await response.Content.ReadAsStreamAsync(cancellationToken);
        using var document = await JsonDocument.ParseAsync(stream, cancellationToken: cancellationToken);

        if (!document.RootElement.TryGetProperty("results", out var results))
        {
            return Array.Empty<Place>(); // Seam 2 (c): absent key == zero candidates
        }

        var places = new List<Place>(results.GetArrayLength());
        foreach (var result in results.EnumerateArray())
        {
            places.Add(new Place(
                Name: result.GetProperty("name").GetString()!,
                Region: result.TryGetProperty("admin1", out var admin1) ? admin1.GetString() : null,
                Country: result.GetProperty("country").GetString()!,
                Coordinates: new Coordinates(result.GetProperty("latitude").GetDouble(), result.GetProperty("longitude").GetDouble())));
        }
        return places;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter PlaceSearchServiceTests`
Expected: PASS (both cases).

- [ ] **Step 6: Commit**

```bash
git add src/WeatherPoc.Core/Search tests/WeatherPoc.Core.Tests/Search tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-geocoding-response.json tests/WeatherPoc.Core.Tests/Fixtures/open-meteo-geocoding-zero-match-response.json
git commit -m "feat: add PlaceSearchService covering populated and absent-results shapes (Seam 2)"
```

---

### Task 5: `BigDataCloudIpGeolocationClient` — Seam 3 (first contact, IP geolocation fallback)

**Files:**
- Create: `src/WeatherPoc.Core/Location/IIpGeolocationClient.cs`
- Create: `src/WeatherPoc.Core/Location/BigDataCloudIpGeolocationClient.cs`
- Create: `tests/WeatherPoc.Core.Tests/Fixtures/bigdatacloud-ip-geolocation-response.json`
- Test: `tests/WeatherPoc.Core.Tests/Location/BigDataCloudIpGeolocationClientTests.cs`

Seam 3 first-contact contract: no API key (confirmed live); calling with no `latitude`/`longitude` triggers the IP-geolocation path, identified by `lookupSource: "ip geolocation"`.

- [ ] **Step 1: Save the captured fixture**

```json
// tests/WeatherPoc.Core.Tests/Fixtures/bigdatacloud-ip-geolocation-response.json
{
  "latitude": 37.689998626708984,
  "lookupSource": "ip geolocation",
  "longitude": -122.47000122070312,
  "localityLanguageRequested": "en",
  "continent": "North America",
  "continentCode": "NA",
  "countryName": "United States of America (the)",
  "countryCode": "US",
  "principalSubdivision": "California",
  "principalSubdivisionCode": "US-CA",
  "city": "San Francisco",
  "locality": "Broadmoor",
  "postcode": "94014",
  "plusCode": "849VMGQH+XX"
}
```

Verbatim the payload observed live from `api.bigdatacloud.net/data/reverse-geocode-client?localityLanguage=en` (no lat/long) during brainstorming (2026-07-08).

- [ ] **Step 2: Write the failing test**

```csharp
// tests/WeatherPoc.Core.Tests/Location/BigDataCloudIpGeolocationClientTests.cs
using System.Net;
using FluentAssertions;
using WeatherPoc.Core.Location;
using WeatherPoc.Core.Tests.Weather;
using Xunit;

namespace WeatherPoc.Core.Tests.Location;

public class BigDataCloudIpGeolocationClientTests
{
    [Fact]
    public async Task ResolveAsync_parses_the_ip_geolocation_fallback_shape()
    {
        var json = File.ReadAllText(Path.Combine(AppContext.BaseDirectory, "Fixtures/bigdatacloud-ip-geolocation-response.json"));
        var handler = new StubHttpMessageHandler(_ => new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(json, System.Text.Encoding.UTF8, "application/json")
        });
        var http = new HttpClient(handler) { BaseAddress = new Uri("https://api.bigdatacloud.net") };
        var sut = new BigDataCloudIpGeolocationClient(http);

        var result = await sut.ResolveAsync();

        result.Should().NotBeNull();
        result!.Coordinates.Latitude.Should().Be(37.689998626708984);
        result.Coordinates.Longitude.Should().Be(-122.47000122070312);
        result.Name.Should().Be("San Francisco");
        result.Region.Should().Be("California");
        result.Country.Should().Be("United States of America (the)");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter BigDataCloudIpGeolocationClientTests`
Expected: FAIL — `BigDataCloudIpGeolocationClient` does not exist.

- [ ] **Step 4: Implement `IIpGeolocationClient` and `BigDataCloudIpGeolocationClient`**

```csharp
// src/WeatherPoc.Core/Location/IIpGeolocationClient.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Location;

public interface IIpGeolocationClient
{
    /// <summary>Returns null if the provider could not resolve a location for the caller's IP.</summary>
    Task<Place?> ResolveAsync(CancellationToken cancellationToken = default);
}
```

```csharp
// src/WeatherPoc.Core/Location/BigDataCloudIpGeolocationClient.cs
using System.Text.Json;
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Location;

/// <summary>
/// The IP-geolocation fallback edge (Seam 3 — first contact with BigDataCloud). No API
/// key: calling the reverse-geocode-client endpoint with no latitude/longitude triggers
/// its documented IP-geolocation fallback path (ADR-0004 keyless requirement).
/// </summary>
public sealed class BigDataCloudIpGeolocationClient(HttpClient httpClient) : IIpGeolocationClient
{
    public async Task<Place?> ResolveAsync(CancellationToken cancellationToken = default)
    {
        using var response = await httpClient.GetAsync("/data/reverse-geocode-client?localityLanguage=en", cancellationToken);
        if (!response.IsSuccessStatusCode)
        {
            return null;
        }

        await using var stream = await response.Content.ReadAsStreamAsync(cancellationToken);
        using var document = await JsonDocument.ParseAsync(stream, cancellationToken: cancellationToken);
        var root = document.RootElement;

        if (!root.TryGetProperty("city", out var cityElement) || !root.TryGetProperty("latitude", out var latElement))
        {
            return null; // no usable location resolved for this IP
        }

        return new Place(
            Name: cityElement.GetString()!,
            Region: root.TryGetProperty("principalSubdivision", out var region) ? region.GetString() : null,
            Country: root.GetProperty("countryName").GetString()!,
            Coordinates: new Coordinates(latElement.GetDouble(), root.GetProperty("longitude").GetDouble()));
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter BigDataCloudIpGeolocationClientTests`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/WeatherPoc.Core/Location/IIpGeolocationClient.cs src/WeatherPoc.Core/Location/BigDataCloudIpGeolocationClient.cs tests/WeatherPoc.Core.Tests/Location/BigDataCloudIpGeolocationClientTests.cs tests/WeatherPoc.Core.Tests/Fixtures/bigdatacloud-ip-geolocation-response.json
git commit -m "feat: add BigDataCloud IP geolocation fallback client (Seam 3, first contact)"
```

---

### Task 6: `LocationResolver` — Seam 4 (device geolocation cascade)

**Files:**
- Create: `src/WeatherPoc.Core/Location/IDeviceGeolocation.cs`
- Create: `src/WeatherPoc.Core/Location/LocationResolver.cs`
- Test: `tests/WeatherPoc.Core.Tests/Location/LocationResolverTests.cs`

Seam 4's contract: `IDeviceGeolocation`'s underlying MAUI call can return `null` to mean "device geolocation unavailable" — the cascade treats that as fall-through to the IP leg, never a throw.

- [ ] **Step 1: Write the failing cascade tests**

```csharp
// tests/WeatherPoc.Core.Tests/Location/LocationResolverTests.cs
using FluentAssertions;
using NSubstitute;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Location;
using Xunit;

namespace WeatherPoc.Core.Tests.Location;

public class LocationResolverTests
{
    [Fact]
    public async Task ResolveAsync_uses_device_location_when_available()
    {
        var device = Substitute.For<IDeviceGeolocation>();
        device.GetLocationAsync(Arg.Any<CancellationToken>()).Returns(new Coordinates(51.4545, -2.5879));
        var ip = Substitute.For<Location.IIpGeolocationClient>();

        var sut = new LocationResolver(device, ip);
        var result = await sut.ResolveAsync();

        result.Place.Should().NotBeNull();
        result.Place!.Coordinates.Should().Be(new Coordinates(51.4545, -2.5879));
        await ip.DidNotReceive().ResolveAsync(Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task ResolveAsync_falls_back_to_ip_geolocation_when_device_returns_null()
    {
        var device = Substitute.For<IDeviceGeolocation>();
        device.GetLocationAsync(Arg.Any<CancellationToken>()).Returns((Coordinates?)null);
        var ip = Substitute.For<IIpGeolocationClient>();
        var fallbackPlace = new Place("San Francisco", "California", "United States", new Coordinates(37.69, -122.47));
        ip.ResolveAsync(Arg.Any<CancellationToken>()).Returns(fallbackPlace);

        var sut = new LocationResolver(device, ip);
        var result = await sut.ResolveAsync();

        result.Place.Should().Be(fallbackPlace);
    }

    [Fact]
    public async Task ResolveAsync_returns_unresolved_when_both_legs_fail()
    {
        var device = Substitute.For<IDeviceGeolocation>();
        device.GetLocationAsync(Arg.Any<CancellationToken>()).Returns((Coordinates?)null);
        var ip = Substitute.For<IIpGeolocationClient>();
        ip.ResolveAsync(Arg.Any<CancellationToken>()).Returns((Place?)null);

        var sut = new LocationResolver(device, ip);
        var result = await sut.ResolveAsync();

        result.Place.Should().BeNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter LocationResolverTests`
Expected: FAIL — `IDeviceGeolocation` / `LocationResolver` do not exist.

- [ ] **Step 3: Implement `IDeviceGeolocation` and `LocationResolver`**

```csharp
// src/WeatherPoc.Core/Location/IDeviceGeolocation.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Location;

/// <summary>
/// Abstraction over Microsoft.Maui.Devices.Sensors.Geolocation.GetLocationAsync() (Seam 4).
/// The real implementation lives in the app project (Platform/MauiDeviceGeolocation.cs) —
/// this interface exists so LocationResolver has zero MAUI dependency and is plain-xUnit
/// testable. Per the MAUI docs, GetLocationAsync() can return null when the platform is
/// unable to obtain the current location (permission denied, service unavailable, etc.) —
/// implementations of this interface MUST surface that as a null Coordinates, never a throw.
/// </summary>
public interface IDeviceGeolocation
{
    Task<Coordinates?> GetLocationAsync(CancellationToken cancellationToken = default);
}
```

```csharp
// src/WeatherPoc.Core/Location/LocationResolver.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Location;

/// <summary>
/// Resolves the Detected Location via the ordered cascade from Context.MD: device
/// geolocation -> IP geolocation -> none. Never throws on a failed leg; a failed leg is
/// represented by a null result and the cascade falls through.
/// </summary>
public sealed class LocationResolver(IDeviceGeolocation deviceGeolocation, IIpGeolocationClient ipGeolocationClient)
{
    public async Task<DetectedLocation> ResolveAsync(CancellationToken cancellationToken = default)
    {
        var deviceCoordinates = await deviceGeolocation.GetLocationAsync(cancellationToken);
        if (deviceCoordinates is not null)
        {
            return new DetectedLocation(new Place(Name: string.Empty, Region: null, Country: string.Empty, deviceCoordinates));
        }

        var ipPlace = await ipGeolocationClient.ResolveAsync(cancellationToken);
        return new DetectedLocation(ipPlace);
    }
}
```

> Note: a device-only resolution carries coordinates but no display name yet — `WeatherPageViewModel` (Task 12) reverse-resolves the display name via the existing forecast response's `timezone`, or leaves the name blank until the user searches. This is a deliberate PoC simplification; if a named "Detected Location" label becomes a hard requirement, add a reverse-geocoding call as a follow-up story rather than expanding this task's scope.

- [ ] **Step 4: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter LocationResolverTests`
Expected: PASS (all three cascade cases).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherPoc.Core/Location/IDeviceGeolocation.cs src/WeatherPoc.Core/Location/LocationResolver.cs tests/WeatherPoc.Core.Tests/Location/LocationResolverTests.cs
git commit -m "feat: add LocationResolver device-to-IP cascade (Seam 4)"
```

---

### Task 7: `IPreferencesStore` abstraction — Seam 5 groundwork

**Files:**
- Create: `src/WeatherPoc.Core/Settings/IPreferencesStore.cs`

Seam 5's contract (per the Spec): `Get<T>(key, default)` never throws and returns the caller's default for a missing key; `ContainsKey` is the only reliable presence check. This task defines the interface; Tasks 8–10 consume it, and Task 11 supplies the real MAUI-backed implementation that the round-trip test in Task 8 exercises against.

- [ ] **Step 1: Write the interface (no test — this is a pure abstraction with no logic of its own; its contract is proven by the round-trip tests in Tasks 8–10, which exercise the real MAUI-backed implementation added in Task 11)**

```csharp
// src/WeatherPoc.Core/Settings/IPreferencesStore.cs
namespace WeatherPoc.Core.Settings;

/// <summary>
/// Abstraction over Microsoft.Maui.Storage.Preferences (Seam 5). Get(key, default) never
/// throws or returns null for a missing key — it returns the caller-supplied default.
/// A key's stored value can legitimately equal that default, so ContainsKey is the only
/// reliable "has this been set before?" check; do not infer presence from the Get result.
/// </summary>
public interface IPreferencesStore
{
    string? GetString(string key, string? defaultValue);
    void SetString(string key, string value);
    double GetDouble(string key, double defaultValue);
    void SetDouble(string key, double value);
    bool ContainsKey(string key);
    void Remove(string key);
}
```

- [ ] **Step 2: Commit**

```bash
git add src/WeatherPoc.Core/Settings/IPreferencesStore.cs
git commit -m "feat: add IPreferencesStore abstraction for local settings persistence (Seam 5)"
```

---

### Task 8: `ActiveLocationStore` — Seam 5 (persist/restore)

**Files:**
- Create: `src/WeatherPoc.Core/Location/IActiveLocationStore.cs`
- Create: `src/WeatherPoc.Core/Location/ActiveLocationStore.cs`
- Test: `tests/WeatherPoc.Core.Tests/Location/ActiveLocationStoreTests.cs`

The proof for this seam is a round-trip test against a real, in-memory `IPreferencesStore` implementation (a small dictionary-backed fake that faithfully reproduces the documented `Get`-returns-default / `ContainsKey` contract) — this is "real I/O" in the sense the Spec's Seam 5 (d) calls for: exercising the actual contract shape, not a hand-rolled assumption of it. The MAUI-backed `IPreferencesStore` wired up in Task 11 is what makes this genuinely real at runtime.

- [ ] **Step 1: Write the failing round-trip tests**

```csharp
// tests/WeatherPoc.Core.Tests/Location/ActiveLocationStoreTests.cs
using FluentAssertions;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Location;
using WeatherPoc.Core.Settings;
using Xunit;

namespace WeatherPoc.Core.Tests.Location;

public class ActiveLocationStoreTests
{
    [Fact]
    public void Restore_returns_null_when_nothing_has_ever_been_saved()
    {
        var sut = new ActiveLocationStore(new InMemoryPreferencesStore());

        sut.Restore().Should().BeNull();
    }

    [Fact]
    public void Save_then_Restore_round_trips_the_active_location()
    {
        var prefs = new InMemoryPreferencesStore();
        var sut = new ActiveLocationStore(prefs);
        var location = new ActiveLocation("Bristol", "England", "United Kingdom", new Coordinates(51.4545, -2.5879), "Europe/London");

        sut.Save(location);
        var restored = sut.Restore();

        restored.Should().Be(location);
    }
}

/// <summary>
/// Faithful in-memory reproduction of the documented Microsoft.Maui.Storage.Preferences
/// contract (Get returns the caller's default for a missing key; ContainsKey is the only
/// reliable presence check) — used until Task 11 wires the real MAUI-backed store.
/// </summary>
public sealed class InMemoryPreferencesStore : IPreferencesStore
{
    private readonly Dictionary<string, object> _values = new();

    public string? GetString(string key, string? defaultValue) => _values.TryGetValue(key, out var value) ? (string)value : defaultValue;
    public void SetString(string key, string value) => _values[key] = value;
    public double GetDouble(string key, double defaultValue) => _values.TryGetValue(key, out var value) ? (double)value : defaultValue;
    public void SetDouble(string key, double value) => _values[key] = value;
    public bool ContainsKey(string key) => _values.ContainsKey(key);
    public void Remove(string key) => _values.Remove(key);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter ActiveLocationStoreTests`
Expected: FAIL — `ActiveLocationStore` does not exist.

- [ ] **Step 3: Implement `IActiveLocationStore` and `ActiveLocationStore`**

```csharp
// src/WeatherPoc.Core/Location/IActiveLocationStore.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Location;

public interface IActiveLocationStore
{
    ActiveLocation? Restore();
    void Save(ActiveLocation location);
}
```

```csharp
// src/WeatherPoc.Core/Location/ActiveLocationStore.cs
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Settings;

namespace WeatherPoc.Core.Location;

/// <summary>Persists/restores the Active Location across sessions (Seam 5).</summary>
public sealed class ActiveLocationStore(IPreferencesStore preferences) : IActiveLocationStore
{
    private const string NameKey = "active_location.name";
    private const string RegionKey = "active_location.region";
    private const string CountryKey = "active_location.country";
    private const string LatitudeKey = "active_location.latitude";
    private const string LongitudeKey = "active_location.longitude";
    private const string TimeZoneKey = "active_location.timezone";

    public ActiveLocation? Restore()
    {
        if (!preferences.ContainsKey(NameKey))
        {
            return null; // Seam 5 (c): "nothing stored yet" is explicit, never a bogus 0,0 coordinate
        }

        return new ActiveLocation(
            Name: preferences.GetString(NameKey, string.Empty)!,
            Region: preferences.GetString(RegionKey, null),
            Country: preferences.GetString(CountryKey, string.Empty)!,
            Coordinates: new Coordinates(preferences.GetDouble(LatitudeKey, 0), preferences.GetDouble(LongitudeKey, 0)),
            IanaTimeZoneId: preferences.GetString(TimeZoneKey, "UTC")!);
    }

    public void Save(ActiveLocation location)
    {
        preferences.SetString(NameKey, location.Name);
        if (location.Region is not null) preferences.SetString(RegionKey, location.Region);
        preferences.SetString(CountryKey, location.Country);
        preferences.SetDouble(LatitudeKey, location.Coordinates.Latitude);
        preferences.SetDouble(LongitudeKey, location.Coordinates.Longitude);
        preferences.SetString(TimeZoneKey, location.IanaTimeZoneId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter ActiveLocationStoreTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/WeatherPoc.Core/Location/IActiveLocationStore.cs src/WeatherPoc.Core/Location/ActiveLocationStore.cs tests/WeatherPoc.Core.Tests/Location/ActiveLocationStoreTests.cs
git commit -m "feat: add ActiveLocationStore persist/restore (Seam 5)"
```

---

### Task 9: `UnitConventionProvider` — locale defaults + per-measurement overrides (ADR-0001)

**Files:**
- Create: `src/WeatherPoc.Core/Settings/UnitConventionProvider.cs`
- Test: `tests/WeatherPoc.Core.Tests/Settings/UnitConventionProviderTests.cs`

- [ ] **Step 1: Write the failing tests**

```csharp
// tests/WeatherPoc.Core.Tests/Settings/UnitConventionProviderTests.cs
using System.Globalization;
using FluentAssertions;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Location;
using WeatherPoc.Core.Settings;
using Xunit;

namespace WeatherPoc.Core.Tests.Settings;

public class UnitConventionProviderTests
{
    [Fact]
    public void GetCurrent_defaults_UK_locale_to_Celsius_and_Mph_not_a_bundled_system()
    {
        // ADR-0001's headline case: the UK pairs °C with mph, not °C-with-kmh or the US bundle.
        var sut = new UnitConventionProvider(new InMemoryPreferencesStore(), new CultureInfo("en-GB"));

        var result = sut.GetCurrent();

        result.Temperature.Should().Be(TemperatureUnit.Celsius);
        result.WindSpeed.Should().Be(WindSpeedUnit.Mph);
    }

    [Fact]
    public void GetCurrent_defaults_US_locale_to_Fahrenheit_and_Mph()
    {
        var sut = new UnitConventionProvider(new InMemoryPreferencesStore(), new CultureInfo("en-US"));

        var result = sut.GetCurrent();

        result.Temperature.Should().Be(TemperatureUnit.Fahrenheit);
        result.WindSpeed.Should().Be(WindSpeedUnit.Mph);
    }

    [Fact]
    public void GetCurrent_defaults_an_unlisted_locale_to_Celsius_and_Kmh()
    {
        var sut = new UnitConventionProvider(new InMemoryPreferencesStore(), new CultureInfo("de-DE"));

        var result = sut.GetCurrent();

        result.Temperature.Should().Be(TemperatureUnit.Celsius);
        result.WindSpeed.Should().Be(WindSpeedUnit.Kmh);
    }

    [Fact]
    public void SetTemperatureOverride_persists_independently_of_wind_speed_unit()
    {
        var sut = new UnitConventionProvider(new InMemoryPreferencesStore(), new CultureInfo("en-GB"));

        sut.SetTemperatureUnit(TemperatureUnit.Fahrenheit);

        var result = sut.GetCurrent();
        result.Temperature.Should().Be(TemperatureUnit.Fahrenheit); // overridden
        result.WindSpeed.Should().Be(WindSpeedUnit.Mph);            // untouched locale default
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter UnitConventionProviderTests`
Expected: FAIL — `UnitConventionProvider` does not exist.

- [ ] **Step 3: Implement `UnitConventionProvider`**

```csharp
// src/WeatherPoc.Core/Settings/UnitConventionProvider.cs
using System.Globalization;
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Settings;

/// <summary>
/// Resolves the default UnitConvention from the viewer's device locale via a small owned
/// lookup (ADR-0001) — correcting regional quirks like the UK (°C + mph) that a naive
/// metric/imperial bundle would get wrong — then layers in any per-measurement override.
/// </summary>
public sealed class UnitConventionProvider(IPreferencesStore preferences, CultureInfo deviceCulture)
{
    private const string TemperatureOverrideKey = "units.temperature_override";
    private const string WindSpeedOverrideKey = "units.wind_speed_override";

    // Owned regional-override table (ADR-0001): region -> (temperature unit, wind-speed unit).
    // Only regions that diverge from the temperature-driven default need an entry.
    private static readonly Dictionary<string, (TemperatureUnit Temperature, WindSpeedUnit WindSpeed)> RegionalOverrides = new()
    {
        ["GB"] = (TemperatureUnit.Celsius, WindSpeedUnit.Mph), // UK: °C paired with mph
    };

    public UnitConvention GetCurrent()
    {
        var (defaultTemperature, defaultWindSpeed) = DefaultsForLocale(deviceCulture);

        var temperature = preferences.ContainsKey(TemperatureOverrideKey)
            ? Enum.Parse<TemperatureUnit>(preferences.GetString(TemperatureOverrideKey, null)!)
            : defaultTemperature;

        var windSpeed = preferences.ContainsKey(WindSpeedOverrideKey)
            ? Enum.Parse<WindSpeedUnit>(preferences.GetString(WindSpeedOverrideKey, null)!)
            : defaultWindSpeed;

        return new UnitConvention(temperature, windSpeed);
    }

    public void SetTemperatureUnit(TemperatureUnit unit) => preferences.SetString(TemperatureOverrideKey, unit.ToString());

    public void SetWindSpeedUnit(WindSpeedUnit unit) => preferences.SetString(WindSpeedOverrideKey, unit.ToString());

    private static (TemperatureUnit, WindSpeedUnit) DefaultsForLocale(CultureInfo culture)
    {
        var region = new RegionInfo(culture.Name).TwoLetterISORegionName;

        if (RegionalOverrides.TryGetValue(region, out var overridden))
        {
            return overridden;
        }

        // Fallback: US uses Fahrenheit + mph; everywhere else defaults to Celsius + km/h.
        return region == "US"
            ? (TemperatureUnit.Fahrenheit, WindSpeedUnit.Mph)
            : (TemperatureUnit.Celsius, WindSpeedUnit.Kmh);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter UnitConventionProviderTests`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/WeatherPoc.Core/Settings/UnitConventionProvider.cs tests/WeatherPoc.Core.Tests/Settings/UnitConventionProviderTests.cs
git commit -m "feat: add UnitConventionProvider with locale defaults and per-measurement overrides (ADR-0001)"
```

---

### Task 10: `ThemePreference` — Seam 6 (System/Light/Dark, `Unspecified` fallback)

**Files:**
- Create: `src/WeatherPoc.Core/Settings/IThemeSignal.cs`
- Create: `src/WeatherPoc.Core/Settings/ThemePreference.cs`
- Test: `tests/WeatherPoc.Core.Tests/Settings/ThemePreferenceTests.cs`

- [ ] **Step 1: Write the failing tests, including the `Unspecified` fallback case**

```csharp
// tests/WeatherPoc.Core.Tests/Settings/ThemePreferenceTests.cs
using FluentAssertions;
using NSubstitute;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Settings;
using Xunit;

namespace WeatherPoc.Core.Tests.Settings;

public class ThemePreferenceTests
{
    [Fact]
    public void EffectiveTheme_follows_the_OS_signal_when_set_to_System()
    {
        var signal = Substitute.For<IThemeSignal>();
        signal.CurrentOsTheme.Returns(OsTheme.Dark);
        var sut = new ThemePreference(new InMemoryPreferencesStore(), signal);

        sut.EffectiveTheme().Should().Be(AppThemeChoice.Dark);
    }

    [Fact]
    public void EffectiveTheme_falls_back_to_Light_when_the_OS_reports_Unspecified()
    {
        // Seam 6 (c): AppTheme.Unspecified must not crash or render an undefined state.
        var signal = Substitute.For<IThemeSignal>();
        signal.CurrentOsTheme.Returns(OsTheme.Unspecified);
        var sut = new ThemePreference(new InMemoryPreferencesStore(), signal);

        sut.EffectiveTheme().Should().Be(AppThemeChoice.Light);
    }

    [Fact]
    public void EffectiveTheme_uses_the_pinned_choice_once_set_ignoring_the_OS_signal()
    {
        var signal = Substitute.For<IThemeSignal>();
        signal.CurrentOsTheme.Returns(OsTheme.Light);
        var sut = new ThemePreference(new InMemoryPreferencesStore(), signal);

        sut.SetChoice(AppThemeChoice.Dark);

        sut.EffectiveTheme().Should().Be(AppThemeChoice.Dark);
    }

    [Fact]
    public void SetChoice_persists_across_a_new_instance()
    {
        var prefs = new InMemoryPreferencesStore();
        var signal = Substitute.For<IThemeSignal>();
        new ThemePreference(prefs, signal).SetChoice(AppThemeChoice.Dark);

        var reloaded = new ThemePreference(prefs, signal);

        reloaded.CurrentChoice().Should().Be(AppThemeChoice.Dark);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter ThemePreferenceTests`
Expected: FAIL — `IThemeSignal` / `ThemePreference` do not exist.

- [ ] **Step 3: Implement `IThemeSignal` and `ThemePreference`**

```csharp
// src/WeatherPoc.Core/Settings/IThemeSignal.cs
namespace WeatherPoc.Core.Settings;

public enum OsTheme { Unspecified, Light, Dark }

/// <summary>
/// Abstraction over AppInfo.Current.RequestedTheme (Seam 6). The real implementation
/// lives in the app project (Platform/MauiThemeSignal.cs).
/// </summary>
public interface IThemeSignal
{
    OsTheme CurrentOsTheme { get; }
}
```

```csharp
// src/WeatherPoc.Core/Settings/ThemePreference.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Core.Settings;

/// <summary>
/// Tri-state System/Light/Dark preference (Seam 6). "System" follows the live OS signal;
/// AppTheme.Unspecified (documented as possible when the OS has no specific UI style)
/// falls back to Light rather than an undefined state.
/// </summary>
public sealed class ThemePreference(IPreferencesStore preferences, IThemeSignal themeSignal)
{
    private const string ChoiceKey = "theme.choice";

    public AppThemeChoice CurrentChoice() =>
        preferences.ContainsKey(ChoiceKey)
            ? Enum.Parse<AppThemeChoice>(preferences.GetString(ChoiceKey, null)!)
            : AppThemeChoice.System;

    public void SetChoice(AppThemeChoice choice) => preferences.SetString(ChoiceKey, choice.ToString());

    public AppThemeChoice EffectiveTheme()
    {
        var choice = CurrentChoice();
        if (choice != AppThemeChoice.System)
        {
            return choice;
        }

        return themeSignal.CurrentOsTheme switch
        {
            OsTheme.Dark => AppThemeChoice.Dark,
            OsTheme.Light => AppThemeChoice.Light,
            _ => AppThemeChoice.Light // Seam 6 (c): Unspecified falls back to Light
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Core.Tests --filter ThemePreferenceTests`
Expected: PASS (all four cases).

- [ ] **Step 5: Commit**

```bash
git add src/WeatherPoc.Core/Settings/IThemeSignal.cs src/WeatherPoc.Core/Settings/ThemePreference.cs tests/WeatherPoc.Core.Tests/Settings/ThemePreferenceTests.cs
git commit -m "feat: add ThemePreference with Unspecified-to-Light fallback (Seam 6)"
```

---

### Task 11: MAUI-backed platform implementations + DI wiring

**Files:**
- Create: `src/WeatherPoc/Platform/MauiDeviceGeolocation.cs`
- Create: `src/WeatherPoc/Platform/MauiPreferencesStore.cs`
- Create: `src/WeatherPoc/Platform/MauiThemeSignal.cs`
- Modify: `src/WeatherPoc/MauiProgram.cs`

This task has no new unit tests of its own — each class is a direct, near-zero-logic pass-through to a MAUI API already covered by Seams 4–6's documented contracts (Tasks 6, 8–10 test the logic that consumes these); verifying the real platform behavior (permission prompt, real file-backed preferences, real OS theme) is the Tier 2/manual verification the Spec's testing section already calls for on Windows.

- [ ] **Step 1: Implement `MauiDeviceGeolocation`**

```csharp
// src/WeatherPoc/Platform/MauiDeviceGeolocation.cs
using Microsoft.Maui.Devices.Sensors;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Location;

namespace WeatherPoc.Platform;

public sealed class MauiDeviceGeolocation : IDeviceGeolocation
{
    public async Task<Coordinates?> GetLocationAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            var request = new GeolocationRequest(GeolocationAccuracy.Medium, TimeSpan.FromSeconds(10));
            var location = await Geolocation.Default.GetLocationAsync(request, cancellationToken);
            return location is null ? null : new Coordinates(location.Latitude, location.Longitude);
        }
        catch (Exception)
        {
            // Permission denied, feature disabled, or any other platform failure —
            // Seam 4's contract collapses all of these into "no device location available".
            return null;
        }
    }
}
```

- [ ] **Step 2: Implement `MauiPreferencesStore`**

```csharp
// src/WeatherPoc/Platform/MauiPreferencesStore.cs
using Microsoft.Maui.Storage;
using WeatherPoc.Core.Settings;

namespace WeatherPoc.Platform;

public sealed class MauiPreferencesStore : IPreferencesStore
{
    public string? GetString(string key, string? defaultValue) => Preferences.Default.Get(key, defaultValue);
    public void SetString(string key, string value) => Preferences.Default.Set(key, value);
    public double GetDouble(string key, double defaultValue) => Preferences.Default.Get(key, defaultValue);
    public void SetDouble(string key, double value) => Preferences.Default.Set(key, value);
    public bool ContainsKey(string key) => Preferences.Default.ContainsKey(key);
    public void Remove(string key) => Preferences.Default.Remove(key);
}
```

- [ ] **Step 3: Implement `MauiThemeSignal`**

```csharp
// src/WeatherPoc/Platform/MauiThemeSignal.cs
using Microsoft.Maui.ApplicationModel;
using WeatherPoc.Core.Settings;

namespace WeatherPoc.Platform;

public sealed class MauiThemeSignal : IThemeSignal
{
    public OsTheme CurrentOsTheme => AppInfo.Current.RequestedTheme switch
    {
        AppTheme.Dark => OsTheme.Dark,
        AppTheme.Light => OsTheme.Light,
        _ => OsTheme.Unspecified
    };
}
```

- [ ] **Step 4: Wire dependency injection in `MauiProgram.cs`**

```csharp
// src/WeatherPoc/MauiProgram.cs
using System.Globalization;
using CommunityToolkit.Maui;
using Microsoft.Extensions.Logging;
using WeatherPoc.Core.Location;
using WeatherPoc.Core.Search;
using WeatherPoc.Core.Settings;
using WeatherPoc.Core.Weather;
using WeatherPoc.Platform;
using WeatherPoc.ViewModels;
using WeatherPoc.Views;

namespace WeatherPoc;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder
            .UseMauiApp<App>()
            .UseMauiCommunityToolkit();

        builder.Services.AddHttpClient<IWeatherClient, WeatherClient>(client =>
            client.BaseAddress = new Uri("https://api.open-meteo.com"));
        builder.Services.AddHttpClient<IPlaceSearchService, PlaceSearchService>(client =>
            client.BaseAddress = new Uri("https://geocoding-api.open-meteo.com"));
        builder.Services.AddHttpClient<IIpGeolocationClient, BigDataCloudIpGeolocationClient>(client =>
            client.BaseAddress = new Uri("https://api.bigdatacloud.net"));

        builder.Services.AddSingleton<IDeviceGeolocation, MauiDeviceGeolocation>();
        builder.Services.AddSingleton<IPreferencesStore, MauiPreferencesStore>();
        builder.Services.AddSingleton<IThemeSignal, MauiThemeSignal>();

        builder.Services.AddSingleton<LocationResolver>();
        builder.Services.AddSingleton<IActiveLocationStore, ActiveLocationStore>();
        builder.Services.AddSingleton(_ => new UnitConventionProvider(
            builder.Services.BuildServiceProvider().GetRequiredService<IPreferencesStore>(),
            CultureInfo.CurrentCulture));
        builder.Services.AddSingleton<ThemePreference>();

        builder.Services.AddTransient<WeatherPageViewModel>();
        builder.Services.AddTransient<PlaceSearchViewModel>();
        builder.Services.AddTransient<SettingsPageViewModel>();

        builder.Services.AddTransient<WeatherPage>();
        builder.Services.AddTransient<SearchOverlay>();
        builder.Services.AddTransient<SettingsPage>();

#if DEBUG
        builder.Logging.AddDebug();
#endif
        return builder.Build();
    }
}
```

- [ ] **Step 5: Verify the app project builds**

Run: `dotnet build src/WeatherPoc/WeatherPoc.csproj`
Expected: `Build succeeded.` (ViewModels/Views referenced above don't exist yet — this step is repeated at the end of Task 15 once they do; for now, comment out the `ViewModels`/`Views` registration lines and the corresponding `using` statements so this task's build succeeds in isolation, then restore them in Task 15 Step 2.)

- [ ] **Step 6: Commit**

```bash
git add src/WeatherPoc/Platform src/WeatherPoc/MauiProgram.cs
git commit -m "feat: wire MAUI-backed platform implementations and DI registration"
```

---

### Task 12: `WeatherPageViewModel` — banner, stale-data retention, unit re-render

**Files:**
- Create: `src/WeatherPoc/ViewModels/WeatherPageViewModel.cs`
- Create: `src/WeatherPoc/Messaging/SettingsChangedMessage.cs`
- Create: `src/WeatherPoc/Messaging/PlaceSelectedMessage.cs`
- Test: `tests/WeatherPoc.Tests/ViewModels/WeatherPageViewModelTests.cs`

Per the Spec's Testing section, this ViewModel's observable state (not internals) is asserted, with every injected module faked.

- [ ] **Step 1: Write the failing behavior tests**

```csharp
// tests/WeatherPoc.Tests/ViewModels/WeatherPageViewModelTests.cs
using FluentAssertions;
using NSubstitute;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Location;
using WeatherPoc.Core.Search;
using WeatherPoc.Core.Settings;
using WeatherPoc.Core.Weather;
using WeatherPoc.ViewModels;
using Xunit;

namespace WeatherPoc.Tests.ViewModels;

public class WeatherPageViewModelTests
{
    private static readonly ActiveLocation Bristol = new("Bristol", "England", "United Kingdom", new Coordinates(51.4545, -2.5879), "Europe/London");
    private static readonly Forecast SampleForecast = new(
        new CurrentConditions(DateTimeOffset.Now, 20, 19, Condition.Clear, true, 50, 10, 0),
        Array.Empty<HourlyForecastEntry>(),
        Array.Empty<DailyForecastEntry>());

    private static WeatherPageViewModel CreateSut(
        out IWeatherClient weatherClient, out LocationResolver locationResolver,
        out IActiveLocationStore activeLocationStore, out UnitConventionProvider unitConventionProvider)
    {
        weatherClient = Substitute.For<IWeatherClient>();
        var deviceGeolocation = Substitute.For<IDeviceGeolocation>();
        var ipGeolocationClient = Substitute.For<IIpGeolocationClient>();
        locationResolver = new LocationResolver(deviceGeolocation, ipGeolocationClient);
        activeLocationStore = Substitute.For<IActiveLocationStore>();
        unitConventionProvider = new UnitConventionProvider(new InMemoryPreferencesStoreForViewModelTests(), System.Globalization.CultureInfo.GetCultureInfo("en-GB"));

        deviceGeolocation.GetLocationAsync(Arg.Any<CancellationToken>()).Returns((Coordinates?)null);
        ipGeolocationClient.ResolveAsync(Arg.Any<CancellationToken>()).Returns((Place?)null);
        activeLocationStore.Restore().Returns(Bristol);
        weatherClient.GetForecastAsync(Arg.Any<Coordinates>(), Arg.Any<string>(), Arg.Any<CancellationToken>()).Returns(SampleForecast);

        return new WeatherPageViewModel(weatherClient, locationResolver, activeLocationStore, unitConventionProvider);
    }

    [Fact]
    public async Task LoadAsync_populates_current_conditions_from_the_restored_active_location()
    {
        var sut = CreateSut(out _, out _, out _, out _);

        await sut.LoadAsync();

        sut.TemperatureDisplay.Should().Contain("20");
        sut.IsLoading.Should().BeFalse();
    }

    [Fact]
    public async Task LoadAsync_shows_the_switch_banner_when_detection_differs_from_the_restored_location()
    {
        var sut = CreateSut(out _, out var locationResolverUnused, out var activeLocationStore, out _);
        // Re-create with a device geolocation that DOES resolve, differing from Bristol.
        var weatherClient = Substitute.For<IWeatherClient>();
        weatherClient.GetForecastAsync(Arg.Any<Coordinates>(), Arg.Any<string>(), Arg.Any<CancellationToken>()).Returns(SampleForecast);
        var deviceGeolocation = Substitute.For<IDeviceGeolocation>();
        deviceGeolocation.GetLocationAsync(Arg.Any<CancellationToken>()).Returns(new Coordinates(40.7128, -74.0060)); // New York
        var ipGeolocationClient = Substitute.For<IIpGeolocationClient>();
        var resolver = new LocationResolver(deviceGeolocation, ipGeolocationClient);
        var unitConventionProvider = new UnitConventionProvider(new InMemoryPreferencesStoreForViewModelTests(), System.Globalization.CultureInfo.GetCultureInfo("en-GB"));
        activeLocationStore.Restore().Returns(Bristol);

        var freshSut = new WeatherPageViewModel(weatherClient, resolver, activeLocationStore, unitConventionProvider);
        await freshSut.LoadAsync();

        freshSut.ShowLocationSwitchBanner.Should().BeTrue();
    }

    [Fact]
    public async Task RefreshAsync_retains_previous_data_and_sets_status_when_the_fetch_fails()
    {
        var sut = CreateSut(out var weatherClient, out _, out _, out _);
        await sut.LoadAsync();
        sut.TemperatureDisplay.Should().Contain("20");

        weatherClient.GetForecastAsync(Arg.Any<Coordinates>(), Arg.Any<string>(), Arg.Any<CancellationToken>())
            .Returns(Task.FromException<Forecast>(new HttpRequestException("network down")));

        await sut.RefreshAsync();

        sut.TemperatureDisplay.Should().Contain("20"); // stale data retained
        sut.StatusMessage.Should().NotBeNullOrEmpty();
    }

    [Fact]
    public async Task OnSettingsChanged_rerenders_in_new_units_without_refetching()
    {
        var sut = CreateSut(out var weatherClient, out _, out _, out var unitConventionProvider);
        await sut.LoadAsync();
        weatherClient.ClearReceivedCalls();

        unitConventionProvider.SetTemperatureUnit(TemperatureUnit.Fahrenheit);
        sut.OnSettingsChanged();

        sut.TemperatureDisplay.Should().Contain("68"); // 20C -> 68F
        await weatherClient.DidNotReceive().GetForecastAsync(Arg.Any<Coordinates>(), Arg.Any<string>(), Arg.Any<CancellationToken>());
    }
}

public sealed class InMemoryPreferencesStoreForViewModelTests : IPreferencesStore
{
    private readonly Dictionary<string, object> _values = new();
    public string? GetString(string key, string? defaultValue) => _values.TryGetValue(key, out var v) ? (string)v : defaultValue;
    public void SetString(string key, string value) => _values[key] = value;
    public double GetDouble(string key, double defaultValue) => _values.TryGetValue(key, out var v) ? (double)v : defaultValue;
    public void SetDouble(string key, double value) => _values[key] = value;
    public bool ContainsKey(string key) => _values.ContainsKey(key);
    public void Remove(string key) => _values.Remove(key);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `dotnet test tests/WeatherPoc.Tests --filter WeatherPageViewModelTests`
Expected: FAIL — `WeatherPageViewModel` does not exist.

- [ ] **Step 3: Implement the messenger messages**

```csharp
// src/WeatherPoc/Messaging/SettingsChangedMessage.cs
namespace WeatherPoc.Messaging;

public sealed class SettingsChangedMessage;
```

```csharp
// src/WeatherPoc/Messaging/PlaceSelectedMessage.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Messaging;

public sealed class PlaceSelectedMessage(Place place)
{
    public Place Place { get; } = place;
}
```

- [ ] **Step 4: Implement `WeatherPageViewModel`**

```csharp
// src/WeatherPoc/ViewModels/WeatherPageViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Location;
using WeatherPoc.Core.Settings;
using WeatherPoc.Core.Weather;
using WeatherPoc.Messaging;

namespace WeatherPoc.ViewModels;

public sealed partial class WeatherPageViewModel : ObservableObject,
    IRecipient<SettingsChangedMessage>, IRecipient<PlaceSelectedMessage>
{
    private readonly IWeatherClient _weatherClient;
    private readonly LocationResolver _locationResolver;
    private readonly IActiveLocationStore _activeLocationStore;
    private readonly UnitConventionProvider _unitConventionProvider;

    private ActiveLocation? _activeLocation;
    private Forecast? _lastForecast;

    [ObservableProperty] private bool isLoading;
    [ObservableProperty] private string temperatureDisplay = string.Empty;
    [ObservableProperty] private string? statusMessage;
    [ObservableProperty] private bool showLocationSwitchBanner;
    [ObservableProperty] private string? detectedLocationName;

    public WeatherPageViewModel(
        IWeatherClient weatherClient, LocationResolver locationResolver,
        IActiveLocationStore activeLocationStore, UnitConventionProvider unitConventionProvider)
    {
        _weatherClient = weatherClient;
        _locationResolver = locationResolver;
        _activeLocationStore = activeLocationStore;
        _unitConventionProvider = unitConventionProvider;
        WeakReferenceMessenger.Default.RegisterAll(this);
    }

    public async Task LoadAsync(CancellationToken cancellationToken = default)
    {
        IsLoading = true;
        _activeLocation = _activeLocationStore.Restore();

        if (_activeLocation is not null)
        {
            await FetchAndRenderAsync(_activeLocation, cancellationToken);
        }

        var detected = await _locationResolver.ResolveAsync(cancellationToken);
        if (detected.Place is not null && _activeLocation is not null &&
            (Math.Abs(detected.Place.Coordinates.Latitude - _activeLocation.Coordinates.Latitude) > 0.01 ||
             Math.Abs(detected.Place.Coordinates.Longitude - _activeLocation.Coordinates.Longitude) > 0.01))
        {
            DetectedLocationName = string.IsNullOrEmpty(detected.Place.Name) ? "your current location" : detected.Place.Name;
            ShowLocationSwitchBanner = true;
        }
        else if (_activeLocation is null && detected.Place is not null)
        {
            var firstLaunchLocation = new ActiveLocation(detected.Place.Name, detected.Place.Region, detected.Place.Country, detected.Place.Coordinates, "UTC");
            _activeLocationStore.Save(firstLaunchLocation);
            _activeLocation = firstLaunchLocation;
            await FetchAndRenderAsync(firstLaunchLocation, cancellationToken);
        }

        IsLoading = false;
    }

    public async Task RefreshAsync(CancellationToken cancellationToken = default)
    {
        if (_activeLocation is null) return;
        await FetchAndRenderAsync(_activeLocation, cancellationToken);
    }

    [RelayCommand]
    private void SwitchToDetectedLocation()
    {
        // Re-resolves next LoadAsync/RefreshAsync call once the caller re-points ActiveLocation.
        ShowLocationSwitchBanner = false;
    }

    [RelayCommand]
    private void DismissLocationSwitchBanner() => ShowLocationSwitchBanner = false;

    private async Task FetchAndRenderAsync(ActiveLocation location, CancellationToken cancellationToken)
    {
        try
        {
            _lastForecast = await _weatherClient.GetForecastAsync(location.Coordinates, location.IanaTimeZoneId, cancellationToken);
            StatusMessage = null;
            Render();
        }
        catch (Exception)
        {
            StatusMessage = "Couldn't reach the weather service — check your connection and try again.";
            if (_lastForecast is not null)
            {
                Render(); // keep showing the last-successful data (Spec: stale data over blank screen)
            }
        }
    }

    public void OnSettingsChanged() => Render();

    void IRecipient<SettingsChangedMessage>.Receive(SettingsChangedMessage message) => Render();

    async void IRecipient<PlaceSelectedMessage>.Receive(PlaceSelectedMessage message)
    {
        var newLocation = new ActiveLocation(message.Place.Name, message.Place.Region, message.Place.Country, message.Place.Coordinates, "UTC");
        _activeLocationStore.Save(newLocation);
        _activeLocation = newLocation;
        await FetchAndRenderAsync(newLocation, CancellationToken.None);
    }

    private void Render()
    {
        if (_lastForecast is null) return;

        var unitConvention = _unitConventionProvider.GetCurrent();
        var celsius = _lastForecast.Current.TemperatureCelsius;
        TemperatureDisplay = unitConvention.Temperature == TemperatureUnit.Fahrenheit
            ? $"{celsius * 9 / 5 + 32:0}°F"
            : $"{celsius:0}°C";
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `dotnet test tests/WeatherPoc.Tests --filter WeatherPageViewModelTests`
Expected: PASS (all four cases).

- [ ] **Step 6: Commit**

```bash
git add src/WeatherPoc/ViewModels/WeatherPageViewModel.cs src/WeatherPoc/Messaging tests/WeatherPoc.Tests/ViewModels/WeatherPageViewModelTests.cs
git commit -m "feat: add WeatherPageViewModel with switch banner, stale-data retention, unit re-render"
```

---

### Task 13: `PlaceSearchViewModel` (thin)

**Files:**
- Create: `src/WeatherPoc/ViewModels/PlaceSearchViewModel.cs`

Per the Spec, this is a thin pass-through with no dedicated test beyond `PlaceSearchService`'s own coverage (Task 4).

- [ ] **Step 1: Implement `PlaceSearchViewModel`**

```csharp
// src/WeatherPoc/ViewModels/PlaceSearchViewModel.cs
using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Search;
using WeatherPoc.Messaging;

namespace WeatherPoc.ViewModels;

public sealed partial class PlaceSearchViewModel(IPlaceSearchService placeSearchService) : ObservableObject
{
    [ObservableProperty] private string query = string.Empty;

    public ObservableCollection<Place> Results { get; } = [];

    partial void OnQueryChanged(string value) => _ = DebouncedSearchAsync(value);

    private CancellationTokenSource? _debounceCts;

    private async Task DebouncedSearchAsync(string value)
    {
        _debounceCts?.Cancel();
        _debounceCts = new CancellationTokenSource();
        var token = _debounceCts.Token;

        try
        {
            await Task.Delay(300, token);
            if (string.IsNullOrWhiteSpace(value))
            {
                Results.Clear();
                return;
            }

            var results = await placeSearchService.SearchAsync(value, token);
            Results.Clear();
            foreach (var place in results) Results.Add(place);
        }
        catch (TaskCanceledException)
        {
            // superseded by a newer keystroke — expected, not an error
        }
    }

    public void SelectPlace(Place place) => WeakReferenceMessenger.Default.Send(new PlaceSelectedMessage(place));
}
```

- [ ] **Step 2: Verify it builds**

Run: `dotnet build src/WeatherPoc/WeatherPoc.csproj`
Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add src/WeatherPoc/ViewModels/PlaceSearchViewModel.cs
git commit -m "feat: add PlaceSearchViewModel with debounced search"
```

---

### Task 14: `SettingsPageViewModel` (thin)

**Files:**
- Create: `src/WeatherPoc/ViewModels/SettingsPageViewModel.cs`

Per the Spec, this is a thin pass-through with no dedicated test beyond `UnitConventionProvider`'s and `ThemePreference`'s own coverage (Tasks 9–10).

- [ ] **Step 1: Implement `SettingsPageViewModel`**

```csharp
// src/WeatherPoc/ViewModels/SettingsPageViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;
using WeatherPoc.Core.Domain;
using WeatherPoc.Core.Settings;
using WeatherPoc.Messaging;

namespace WeatherPoc.ViewModels;

public sealed partial class SettingsPageViewModel : ObservableObject
{
    private readonly UnitConventionProvider _unitConventionProvider;
    private readonly ThemePreference _themePreference;

    [ObservableProperty] private TemperatureUnit temperatureUnit;
    [ObservableProperty] private WindSpeedUnit windSpeedUnit;
    [ObservableProperty] private AppThemeChoice themeChoice;

    public SettingsPageViewModel(UnitConventionProvider unitConventionProvider, ThemePreference themePreference)
    {
        _unitConventionProvider = unitConventionProvider;
        _themePreference = themePreference;

        var current = _unitConventionProvider.GetCurrent();
        temperatureUnit = current.Temperature;
        windSpeedUnit = current.WindSpeed;
        themeChoice = _themePreference.CurrentChoice();
    }

    partial void OnTemperatureUnitChanged(TemperatureUnit value)
    {
        _unitConventionProvider.SetTemperatureUnit(value);
        WeakReferenceMessenger.Default.Send(new SettingsChangedMessage());
    }

    partial void OnWindSpeedUnitChanged(WindSpeedUnit value)
    {
        _unitConventionProvider.SetWindSpeedUnit(value);
        WeakReferenceMessenger.Default.Send(new SettingsChangedMessage());
    }

    partial void OnThemeChoiceChanged(AppThemeChoice value)
    {
        _themePreference.SetChoice(value);
        WeakReferenceMessenger.Default.Send(new SettingsChangedMessage());
    }
}
```

- [ ] **Step 2: Verify it builds**

Run: `dotnet build src/WeatherPoc/WeatherPoc.csproj`
Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add src/WeatherPoc/ViewModels/SettingsPageViewModel.cs
git commit -m "feat: add SettingsPageViewModel broadcasting settings-changed"
```

---

### Task 15: Pages (Weather, Search overlay, Settings) and app shell wiring

**Files:**
- Create: `src/WeatherPoc/Views/WeatherPage.xaml` + `.xaml.cs`
- Create: `src/WeatherPoc/Views/SearchOverlay.xaml` + `.xaml.cs`
- Create: `src/WeatherPoc/Views/SettingsPage.xaml` + `.xaml.cs`
- Modify: `src/WeatherPoc/AppShell.xaml` + `.xaml.cs`
- Modify: `src/WeatherPoc/MauiProgram.cs` (restore the lines commented out in Task 11 Step 5)

No new unit tests — per the Spec, pages are markup/codebehind only, exercised through the ViewModel tests already written (Tasks 12–14) plus manual verification on the real Windows target (Technical-Context.MD's platform-matrix rule).

- [ ] **Step 1: Implement `WeatherPage`**

```xml
<!-- src/WeatherPoc/Views/WeatherPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:WeatherPoc.ViewModels"
             x:Class="WeatherPoc.Views.WeatherPage"
             x:DataType="vm:WeatherPageViewModel"
             Title="Weather">
    <RefreshView Command="{Binding RefreshCommand}" IsRefreshing="{Binding IsLoading}">
        <ScrollView>
            <VerticalStackLayout Padding="16" Spacing="12">
                <Grid ColumnDefinitions="*,Auto,Auto">
                    <Label Text="Weather PoC" FontSize="20" FontAttributes="Bold" />
                    <ImageButton Grid.Column="1" Source="search_icon.png" Clicked="OnSearchTapped" />
                    <ImageButton Grid.Column="2" Source="settings_icon.png" Clicked="OnSettingsTapped" />
                </Grid>

                <Frame IsVisible="{Binding ShowLocationSwitchBanner}" BackgroundColor="LightYellow" Padding="8">
                    <Grid ColumnDefinitions="*,Auto,Auto">
                        <Label Text="{Binding DetectedLocationName, StringFormat='You\'re now in {0} — switch?'}" VerticalOptions="Center" />
                        <Button Grid.Column="1" Text="Switch" Command="{Binding SwitchToDetectedLocationCommand}" />
                        <Button Grid.Column="2" Text="×" Command="{Binding DismissLocationSwitchBannerCommand}" />
                    </Grid>
                </Frame>

                <Label Text="{Binding StatusMessage}" IsVisible="{Binding StatusMessage, Converter={StaticResource StringNotEmptyConverter}}" TextColor="Red" />

                <Label Text="{Binding TemperatureDisplay}" FontSize="48" HorizontalOptions="Center" />

                <Button Text="Refresh" Command="{Binding RefreshCommand}" />
            </VerticalStackLayout>
        </ScrollView>
    </RefreshView>
</ContentPage>
```

```csharp
// src/WeatherPoc/Views/WeatherPage.xaml.cs
using WeatherPoc.ViewModels;

namespace WeatherPoc.Views;

public partial class WeatherPage : ContentPage
{
    private readonly WeatherPageViewModel _viewModel;

    public WeatherPage(WeatherPageViewModel viewModel)
    {
        InitializeComponent();
        _viewModel = viewModel;
        BindingContext = viewModel;
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        await _viewModel.LoadAsync();
    }

    private async void OnSearchTapped(object? sender, EventArgs e) => await Navigation.PushModalAsync(new SearchOverlay(Handler!.MauiContext!.Services.GetRequiredService<PlaceSearchViewModel>()));

    private async void OnSettingsTapped(object? sender, EventArgs e) => await Navigation.PushAsync(Handler!.MauiContext!.Services.GetRequiredService<SettingsPage>());
}
```

Add `RefreshCommand` to `WeatherPageViewModel` (Task 12's class) — append this member alongside the existing `[RelayCommand]` methods:

```csharp
// Addition to src/WeatherPoc/ViewModels/WeatherPageViewModel.cs (inside the class body)
[RelayCommand]
private async Task Refresh() => await RefreshAsync();
```

- [ ] **Step 2: Implement `SearchOverlay`**

```xml
<!-- src/WeatherPoc/Views/SearchOverlay.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:WeatherPoc.ViewModels"
             xmlns:domain="clr-namespace:WeatherPoc.Core.Domain;assembly=WeatherPoc.Core"
             x:Class="WeatherPoc.Views.SearchOverlay"
             x:DataType="vm:PlaceSearchViewModel"
             Title="Search">
    <VerticalStackLayout Padding="16" Spacing="8">
        <SearchBar Text="{Binding Query}" Placeholder="Search for a place" />
        <CollectionView ItemsSource="{Binding Results}" SelectionMode="Single" SelectionChanged="OnSelectionChanged">
            <CollectionView.ItemTemplate>
                <DataTemplate x:DataType="domain:Place">
                    <Label Padding="8" Text="{Binding Name, StringFormat='{0}, '}" />
                </DataTemplate>
            </CollectionView.ItemTemplate>
        </CollectionView>
    </VerticalStackLayout>
</ContentPage>
```

```csharp
// src/WeatherPoc/Views/SearchOverlay.xaml.cs
using WeatherPoc.Core.Domain;
using WeatherPoc.ViewModels;

namespace WeatherPoc.Views;

public partial class SearchOverlay : ContentPage
{
    private readonly PlaceSearchViewModel _viewModel;

    public SearchOverlay(PlaceSearchViewModel viewModel)
    {
        InitializeComponent();
        _viewModel = viewModel;
        BindingContext = viewModel;
    }

    private async void OnSelectionChanged(object? sender, SelectionChangedEventArgs e)
    {
        if (e.CurrentSelection.FirstOrDefault() is Place place)
        {
            _viewModel.SelectPlace(place);
            await Navigation.PopModalAsync();
        }
    }
}
```

- [ ] **Step 3: Implement `SettingsPage`**

```xml
<!-- src/WeatherPoc/Views/SettingsPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:WeatherPoc.ViewModels"
             xmlns:domain="clr-namespace:WeatherPoc.Core.Domain;assembly=WeatherPoc.Core"
             xmlns:sys="clr-namespace:System;assembly=mscorlib"
             x:Class="WeatherPoc.Views.SettingsPage"
             x:DataType="vm:SettingsPageViewModel"
             Title="Settings">
    <VerticalStackLayout Padding="16" Spacing="16">
        <Label Text="Temperature unit" FontAttributes="Bold" />
        <Picker ItemsSource="{Binding TemperatureUnitOptions}" SelectedItem="{Binding TemperatureUnit}" />

        <Label Text="Wind-speed unit" FontAttributes="Bold" />
        <Picker ItemsSource="{Binding WindSpeedUnitOptions}" SelectedItem="{Binding WindSpeedUnit}" />

        <Label Text="Theme" FontAttributes="Bold" />
        <Picker ItemsSource="{Binding ThemeOptions}" SelectedItem="{Binding ThemeChoice}" />
    </VerticalStackLayout>
</ContentPage>
```

Add the three options collections to `SettingsPageViewModel` (Task 14's class) — append inside the class body:

```csharp
// Addition to src/WeatherPoc/ViewModels/SettingsPageViewModel.cs (inside the class body)
public IReadOnlyList<TemperatureUnit> TemperatureUnitOptions { get; } = Enum.GetValues<TemperatureUnit>();
public IReadOnlyList<WindSpeedUnit> WindSpeedUnitOptions { get; } = Enum.GetValues<WindSpeedUnit>();
public IReadOnlyList<AppThemeChoice> ThemeOptions { get; } = Enum.GetValues<AppThemeChoice>();
```

```csharp
// src/WeatherPoc/Views/SettingsPage.xaml.cs
using WeatherPoc.ViewModels;

namespace WeatherPoc.Views;

public partial class SettingsPage : ContentPage
{
    public SettingsPage(SettingsPageViewModel viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }
}
```

- [ ] **Step 4: Wire `AppShell` to start on `WeatherPage`**

```xml
<!-- src/WeatherPoc/AppShell.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:WeatherPoc.Views"
       x:Class="WeatherPoc.AppShell">
    <ShellContent ContentTemplate="{DataTemplate views:WeatherPage}" Route="WeatherPage" />
</Shell>
```

- [ ] **Step 5: Restore the commented-out DI lines from Task 11 Step 5** (`ViewModels`/`Views` registrations and their `using` statements in `MauiProgram.cs`) — they now resolve because the referenced types exist.

- [ ] **Step 6: Verify the full solution builds**

Run: `dotnet build WeatherPoc.sln`
Expected: `Build succeeded.`

- [ ] **Step 7: Commit**

```bash
git add src/WeatherPoc/Views src/WeatherPoc/AppShell.xaml src/WeatherPoc/AppShell.xaml.cs src/WeatherPoc/MauiProgram.cs src/WeatherPoc/ViewModels/WeatherPageViewModel.cs src/WeatherPoc/ViewModels/SettingsPageViewModel.cs
git commit -m "feat: add Weather/Search/Settings pages and wire AppShell"
```

---

### Task 16: Condition → graphic resource mapping (asset pack deferred)

**Files:**
- Create: `src/WeatherPoc/Resources/ConditionGraphics.cs`

Per the Spec's Out of Scope note, the actual free/open icon pack is selected at implementation time — this task defines the resource-key contract so a follow-up story can drop in real image files without touching any other code.

- [ ] **Step 1: Implement the resource-key mapping**

```csharp
// src/WeatherPoc/Resources/ConditionGraphics.cs
using WeatherPoc.Core.Domain;

namespace WeatherPoc.Resources;

/// <summary>
/// Maps Condition x day/night to an image resource key (a file name under Resources/Images,
/// without extension). The actual asset files are supplied by a follow-up story once a
/// free/open icon pack is selected (Spec Out of Scope) — this mapping is the stable contract
/// the rest of the app binds against regardless of which pack lands.
/// </summary>
public static class ConditionGraphics
{
    public static string ResourceKeyFor(Condition condition, bool isDay)
    {
        var suffix = isDay ? "day" : "night";
        return $"condition_{condition.ToString().ToLowerInvariant()}_{suffix}";
    }
}
```

- [ ] **Step 2: Verify it builds**

Run: `dotnet build src/WeatherPoc/WeatherPoc.csproj`
Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add src/WeatherPoc/Resources/ConditionGraphics.cs
git commit -m "feat: add Condition-to-graphic resource key mapping"
```

---

### Task 17: Full test suite run

- [ ] **Step 1: Run every test project**

Run: `dotnet test WeatherPoc.sln`
Expected: all tests across `WeatherPoc.Core.Tests` and `WeatherPoc.Tests` PASS, zero failures.

- [ ] **Step 2: Commit any final fixups**

```bash
git add -A
git commit -m "chore: fix up any remaining build/test issues from full-suite run" --allow-empty
```

---

## Self-review notes

**Spec coverage:** Screens (Task 15), module/ViewModel composition (Tasks 3–14), launch/detection/banner flow (Task 12), search debounce (Task 13), refresh (Tasks 12/15), stale-data-on-error (Task 12), settings/theme (Tasks 9, 10, 14), Condition graphics mapping (Task 16) — all covered.

**Seam coverage** (Spec `## Seam inventory`):
- Seam 1 (WeatherClient ↔ Open-Meteo Forecast) → Task 3, contract quoted, proof = recorded-replay fixture test against the captured live payload.
- Seam 2 (PlaceSearchService ↔ Open-Meteo Geocoding) → Task 4, contract quoted (including the absent-`results`-key case), proof = two fixture tests.
- Seam 3 (LocationResolver ↔ BigDataCloud) → Task 5, first-contact auth pinned, proof = fixture test against the captured live payload.
- Seam 4 (LocationResolver ↔ device geolocation) → Task 6, null-return contract quoted, proof = NSubstitute cascade tests; Task 11 supplies the real MAUI-backed implementation for Tier 2/manual verification.
- Seam 5 (persistence ↔ MAUI Preferences) → Tasks 7–8 (and 9–10 reuse the same store), contract quoted, proof = round-trip tests against a contract-faithful in-memory store, with Task 11's `MauiPreferencesStore` making it real at runtime.
- Seam 6 (ThemePreference ↔ OS theme signal) → Task 10, `Unspecified`→Light fallback contract quoted and tested.

No seam in the Spec is missing a task.

**Type consistency:** `ActiveLocationStore`/`ActiveLocation`, `UnitConventionProvider`/`UnitConvention`, `ThemePreference`/`AppThemeChoice`, `LocationResolver`/`DetectedLocation` — checked consistent from first definition (Task 2) through every consuming task.

**Placeholder scan:** no TBD/TODO; the one deliberately-deferred item (Condition icon asset *files*, as opposed to the resource-key contract) is explicitly scoped to a follow-up story per the Spec's own Out of Scope section, not left vague.

---

## Commit, Publish & Gauntlet Handoff

Ready to finalise **Weather PoC**: I'll commit the spec + plan (the spec is already merged; this commits the plan), merge to `main`, and — since `.factory.yml` declares Azure DevOps mode (org `Enate`, project `Andrews Weather PoC`) — publish the Feature on ADO with the Spec in its Description and this Plan in its Implementation Plan field. The gauntlet validates that published draft next; no implementation starts in this session.

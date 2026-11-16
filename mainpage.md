\mainpage WeatherGameplay Plugin

The game-facing way to get weather: a world subsystem that owns a `UWeatherInstance` for you.

## Overview

Use this plugin when gameplay code needs the current weather and a way to move to
the next day, and you would rather not construct, own, and pass around a
`UWeatherInstance` yourself. `UWeatherGameplaySubsystem` is a `UWorldSubsystem`, so
there is one per world and any actor can find it. You tell it which climate to use
once, then read today's weather and advance the day through it.

It adds no weather logic of its own. Generation, seeds, and the shape of the
result all belong to the Weather plugin; this plugin owns lifetime and access.
The subsystem does not tick and has no idea how long a day lasts in your game.
When a day ends is your decision, and you call `AdvanceDay`.

It depends on `Weather`, `EcologyCommon`, and `STD_Utilities`.

## The primary path

```cpp
UWeatherGameplaySubsystem* Weather = GetWorld()->GetSubsystem<UWeatherGameplaySubsystem>();

// 1. Choose a climate, once. Only the template is mandatory; FWeatherInitParams
//    carries everything else, each field defaulted, so override what you need.
Weather->InitializeWeather(TemperateClimate, { .Seed = 543243524, .Latitude = 51.5f, .StartingDay = 93 });

// 2. Read today. Valid from the moment InitializeWeather returns.
const FWeatherDay& Today = Weather->GetTodaysWeather();
const float NoonTempC = Today.Temperature_C.GetRichCurveConst()->Eval(12.0f);

// 3. When your game's day rolls over, advance. Returns the new day.
const FWeatherDay Tomorrow = Weather->AdvanceDay();
// Weather->GetCurrentDay() is now 94.
```

The subsystem builds its `UWeatherInstance` lazily on the first `InitializeWeather`,
not when the world starts. That keeps the choice of climate and seed a decision
your game makes, and it is why nothing works until you call it.

`FWeatherInitParams` and `FWeatherDay` are Weather's types, and the Weather main
page documents every field and channel. `InitializeWeather` forwards the params
to `UWeatherInstance::Initialize` unchanged.

### Saving and reproducing a run

If you pass no seed, each `InitializeWeather` picks a fresh random one. To
reproduce that run later, read it back right after initializing and save it:

```cpp
Weather->InitializeWeather(TemperateClimate);   // Seed defaults to 0: randomize
const int32 SavedSeed = Weather->GetSeed();     // the seed actually in use, never 0
```

Passing `SavedSeed` back in with the same template and params replays the same
weather. See the Weather main page for what that does and does not guarantee.

## Gotchas

**Every accessor requires `InitializeWeather` first.** `GetSeed`, `GetCurrentDay`,
`GetTodaysWeather`, and `AdvanceDay` each raise
`WeatherGameplay::NotInitializedException` if called earlier. With exceptions
enabled that throws. In a default Unreal build (exceptions off) it logs and then
hits a Fatal log, so treat it as a programming error and not something to catch.
This is stricter than `UWeatherInstance` itself, whose plain getters do not check.

**Calling `InitializeWeather` again rebinds everything.** The same instance is
reused, but its template, seed, day, and all carried weather state are replaced. It
is a restart, not an update.

**The subsystem exists in every world type.** Its `Initialize` logs a verbose
message for worlds other than PIE and Game (an editor preview world, say) but does
not stop the subsystem from being created. Nothing is constructed until you call
`InitializeWeather`, so an unused subsystem in a preview world costs nothing.

**`StartingDay` is not a seek.** See the Weather main page. Initializing at day 200
gives a plausible day 200, not the one you would reach by advancing from day 1.

**Not built yet.** There is no MVVM binding, no replication, and no automatic
day-advance. Replicating weather to clients is planned and will need a separate
carrier actor or component, since a world subsystem cannot replicate.

## Reference

| Call | Returns | Notes |
|---|---|---|
| `InitializeWeather(Template, Params)` | `void` | Constructs the owned instance on first call. `Params` defaults to an all-defaults `FWeatherInitParams`. Null template is fatal. |
| `GetSeed()` | `int32` | Seed actually in use, never 0. |
| `GetCurrentDay()` | `int32` | Current day, 1-based. |
| `GetTodaysWeather()` | `const FWeatherDay&` | Weather for the current day. |
| `AdvanceDay()` | `FWeatherDay` | Generates and returns the next day; same value `GetTodaysWeather()` now returns. |

All are `BlueprintCallable`; the getters are also `BlueprintPure`.

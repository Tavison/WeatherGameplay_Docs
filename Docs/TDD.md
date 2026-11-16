# WeatherGameplay: Technical Design

This document records why `UWeatherGameplaySubsystem` is shaped the way it is.
Read the [main page](../mainpage.md) first for how to use it. The plugin is small
on purpose; most of the interesting decisions are about what it refuses to do.

## Where it sits

```
game code
  └─ UWeatherGameplaySubsystem   (this plugin: lifetime + access, per world)
       └─ UWeatherInstance       (Weather: generation)
            implements IWeatherDaySource + returns FWeatherDay   (EcologyCommon)
```

Weather is deliberately game-agnostic, so it has no opinion on worlds, save games,
UI, or networking. This plugin is where those opinions belong. It is the only place
in the weather stack that is allowed to know it is inside a game. Its public
surface depends only on `FWeatherDay` and `FWeatherInitParams` and on
`UWeatherInstance`'s day-based calls (`Initialize`, `GetCurrentDay`,
`GetTodaysWeather`, `AdvanceDay`, `GetSeed`), so Weather can change how it
generates without touching this code.

## Decisions

**A world subsystem, not a component or a singleton.** Weather is per-world state
that any actor may need and that nothing in particular owns. A `UWorldSubsystem`
gives one instance per world with the world's own lifetime, findable without
plumbing a reference through gameplay code.

**The `UWeatherInstance` is built lazily, on the first `InitializeWeather`.** Which
climate and which seed to use is a game decision, and it is often not known when
the world starts. Constructing eagerly would force a default climate that the game
then immediately replaces. Laziness also makes the class directly testable with
`NewObject`, without a live `UWorld`, the same way `UWeatherInstance` is tested.

**`InitializeWeather` takes the same `FWeatherInitParams` that Weather does, and
forwards it untouched.** The wrapper used to repeat Weather's seven-argument
signature and its defaults, which meant every new Weather parameter had to be added
in two places and the two lists could drift. Sharing the struct removes that. The
mandatory input, the template, stays a plain parameter: only genuinely optional
values belong in a struct whose defaults are all valid.

**Every accessor fails loudly before initialization.** `GetSeed`, `GetCurrentDay`,
`GetTodaysWeather`, and `AdvanceDay` go through `RequireInitialized`, which raises
`WeatherGameplay::NotInitializedException` through `WeatherGameplay::EH::Throw`. That
is the shared error-handling policy every plugin in the ecology stack uses: an
exceptions-enabled build throws, and an exceptions-disabled build (the Unreal
default) logs and then hits a Fatal log, so a call made out of order can never
quietly return a default day. It is stricter than `UWeatherInstance`, whose plain
getters return defaults before `Initialize`, because at this layer a missing
initialization is always a game-code mistake and never a legitimate empty state.

**`RequireInitialized` is a free function in a `*Private` namespace, not a private
method.** Its only dependency is the owned instance pointer, so an explicit
signature says everything it can touch. This is the same convention as
`VegetationInstancePrivate`.

**`Initialize(FSubsystemCollectionBase&)` does nothing and logs at Verbose for
non-PIE, non-Game worlds.** It is a documented guard for later, not a current
requirement. It does not prevent the subsystem from existing in those worlds,
because nothing is constructed until `InitializeWeather` is called, so there is
nothing to prevent.

**No ticking, no time, no compression.** The subsystem never advances the day on
its own and has no concept of how long a day is. This mirrors Weather, which
generates exactly one real day per call and leaves pacing to the consumer. A game
that wants a compressed calendar or a day-night-driven rollover owns that logic and
calls `AdvanceDay` when it decides a day has passed.

## What is not built

- **MVVM hooks.** Not started. Open question: whether the ViewModel belongs in this
  plugin or in the game's app layer, under the convention that plugins are the Model
  and the application supplies View and ViewModel.
- **Replication.** Not started. A `UWorldSubsystem` cannot replicate, so this needs a
  separate `AActor` or `UActorComponent` carrier. `FWeatherDay` is curve-shaped, with
  no scalar accessor, so the carrier would replicate derived scalars (evaluate the
  curves at the current time of day) rather than the struct itself.
- **A planet asset.** Bundling planet-level constants such as `AxialTiltDeg` into one
  authored object, so a designer picks a planet instead of typing a number, has been
  discussed and not built. This plugin is a plausible home for it.

## Test surface

Five tests, `WeatherGameplay.WeatherGameplaySubsystem.*`, all in
`WeatherGameplaySubsystemTests.cpp`. They construct the subsystem with `NewObject`
and never need a world.

| Test | What it verifies |
|---|---|
| `InitializeWeather.DefaultsToDay1` | Defaults start on day 1 and today's weather exists immediately. |
| `InitializeWeather.ExplicitValuesPassThrough` | An explicit `StartingDay` and `YearLength` reach the instance. |
| `InitializeWeather.OmittedSeedRandomizesPerInstance` | An omitted seed randomizes per subsystem, so two subsystems land on different seeds, and `GetSeed` reports the real value, never 0. |
| `InitializeWeather.HorizonObstructionReachesGeneration` | A nonzero `HorizonObstructionDeg` shortens `DayLength_hours`, proving the forwarding reaches generation and is not merely accepted. |
| `AdvanceDay.MovesCurrentDayByOne` | `AdvanceDay` advances by exactly one day and keeps `GetTodaysWeather` in step. |

There is deliberately no test for calling an accessor before `InitializeWeather`.
That path ends in a Fatal log under the default exceptions-disabled build, which
would take down the whole test process rather than fail one test. None of the
ecology plugins test their equivalent guards for the same reason.

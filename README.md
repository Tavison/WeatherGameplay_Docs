# WeatherGameplay: Game-Facing Weather for Unreal Engine 5

WeatherGameplay is the layer between game code and the Weather plugin. Its
`UWeatherGameplaySubsystem` is a world subsystem that owns a `UWeatherInstance`,
so gameplay code can pick a climate once, read today's weather, and advance the
day without constructing or passing around a weather object of its own.

It adds no weather logic. Generation belongs to Weather; this plugin owns lifetime
and access. Replication and UI binding are planned and not built.

## Docs layout

```
mainpage.md   Doxygen entry point: the call sequence, gotchas, and API reference
Docs/TDD.md   Technical design: why the subsystem is shaped this way, and what is not built
```

## Companion repositories

| Repo | Contents |
|---|---|
| WeatherGameplay_Docs (public) | These docs |
| WeatherGameplay (private) | C++ source for this plugin |
| Weather_Code (private) | The Weather plugin this one wraps |
| Weather (public) | Docs for the Weather plugin |

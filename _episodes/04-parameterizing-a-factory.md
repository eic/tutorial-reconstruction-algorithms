---
title: "Parameterizing a factory"
teaching: 5
exercises: 0
objectives:
- "Learn how to set parameters on a factory"
- "Learn how to override factories via a generator"
- "Learn how to override factories via the command line"
- "Learn how to access services from a factory"
---

## Setting parameters on a factory

Parameters are also handled using registered members. JOmniFactory provides a `Parameter` class template which can hold its own value, but in EICrecon we prefer to use Config structs. Thus JOmniFactory provides `ParameterRef`, which stores a reference into the Config object.

```c++
    ParameterRef<double> m_samplingFraction {this, "samplingFraction", config().sampFrac};
    ParameterRef<std::string> m_energyWeight {this, "energyWeight", config().energyWeight};
```

Parameters are fetched immediately before `Init()` is called, so you may access them from any of the callbacks like so:

```c++
    void Process(int64_t run_number, uint64_t event_number) {
        logger()->debug( "Event {}: samplingFraction = {}", event_number, m_samplingFraction() );
    }

```
Because we are using ParameterRefs, we can also access the field the ref points to directly:
```c++
    void Process(int64_t run_number, uint64_t event_number) {
        logger()->debug( "Event {}: samplingFraction = {}", event_number, config().sampFrac );
    }
```

## Config objects

We create a plain-old struct to hold our parameters. For now this config struct can live in the same header file as our factory, although eventually it should belong with the algorithm instead.

```c++
struct ReconstructedElectrons_config {
    double threshold = 0.9;
    int bucket_count = 4;
    // ...
}
```

By passing it in to the JOmniFactory base class, we can make it automatically available via the `config()` method.

```c++
class ReconstructedElectrons_factory : public JOmniFactory<ReconstructedElectrons_factory, ElectronReconstructionConfig> {
    ...
}
```


## Overriding parameters via a generator

If you use a Config object for your parameters, you can pass it in directly to the factory generator:

```c++
    app->Add(new JOmniFactoryGeneratorT<BasicTestAlg>(
        "FunTest", {"MyHits"}, {"MyClusters"}, 
        {
          .threshold = 6.1,
          .bucket_count = 22
        },
        app));
```

## Overriding parameters via the command line

We can override parameters on the command line like so:

```bash
eicrecon -PFunTest:threshold=12.0 in.root
```


## Accessing services from a factory

Services are singletons that provide access to resources such as loggers, geometry, magnetic field maps, etc. Services need to be thread-safe because they are shared by different threads. The most relevant service right now is `DD4hep_service`. You obtain a service using a registered member like this:

```c++
    Service<DD4hep_service> m_geoSvc {this};
```

Oftentimes we want to retrieve a resource from a Service and refresh it whenever the run number changes. OmniFactory provides `Resource` for this purpose.

> Exercise:
> - Give your factory a Config struct
> - Give your Config struct some parameters
> - Experiment with overriding parameter values in the generator and on the command line.
{: .challenge}
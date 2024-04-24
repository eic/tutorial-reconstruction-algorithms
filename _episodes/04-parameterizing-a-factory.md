---
title: "Parameterizing a factory"
teaching: 5
exercises: 0
questions:
objectives:
- "Learn how to set parameters on a factory"
- "Learn how to override factories via a generator"
- "Learn how to override factories via the command line"
- "Learn how to access services from a factory"

keypoints:
---

## Setting parameters on a factory

Parameters are also handled using registered members. JOmniFactory provides a `Parameter` class template which can hold its own value, but in EICrecon we prefer to use Config structs. Thus JOmniFactory provides `ParameterRef`, which stores a reference into the Config object.

```c++
    ParameterRef<double> m_samplingFraction {this, "samplingFraction", config().sampFrac};
    ParameterRef<std::string> m_energyWeight {this, "energyWeight", config().energyWeight};
```

Parameters are fetched immediately before `Init()` is called, so you may access them from any of the callbacks like so:

```
```


## Overriding parameters via a generator

If you use a Config object for your parameters, you can pass it in like so.

```c++
        app->Add(new JOmniFactoryGeneratorT<CalorimeterHitReco_factory>(
          "B0ECalRecHits", {"B0ECalRawHits"}, {"B0ECalRecHits"},
          {
            .capADC = 16384,
            .dyRangeADC = 20. * dd4hep::GeV,
            .pedMeanADC = 100,
            .pedSigmaADC = 1,
            .resolutionTDC = 1e-11,
            .thresholdFactor = 0.0,
            .thresholdValue = 0.0,
            .sampFrac = 0.998,
            .readout = "B0ECalHits",
            .sectorField = "sector",
          },
          app
        ));
```


## Overriding parameters via the command line

Suppose our factory is configured like so:
```c++
    app->Add(new JOmniFactoryGeneratorT<BasicTestAlg>(
        "FunTest", {"MyHits"}, {"MyClusters"}, 
        {
          .threshold = 6.1,
          .bucket_count = 22
        },
        app);
```

We can override it's `threshold` parameter on the command line like so:
```bash
eicrecon -PFunTest:threshold=12.0 in.root
```


## Accessing services from a factory

Services are singletons that provide access to resources such as loggers, geometry, magnetic field maps, etc. Services need to be thread-safe because they are shared by different threads. The most relevant service right now is `DD4hep_service`. You obtain a service using a registered member like this:

```c++
    Service<DD4hep_service> m_geoSvc {this};
```

Oftentimes we want to retrieve a resource from a Service and refresh it whenever the run number changes. OmniFactory provides `Resource` for this purpose.


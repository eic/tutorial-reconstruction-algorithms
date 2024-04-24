---
title: "Calling a factory"
teaching: 10
exercises: 1
objectives:
- "Learn how to wire a factory using a factory generator"
- "Learn how to call the factory as a once-off"
- "Learn how to call the factory every time"
---

## Wiring a factory using a factory generator

Instead of handing over the OmniFactory to JANA directly, we create a `JOmniFactoryGeneratorT` which is basically a recipe that JANA uses to create factories later. There are several reasons we need this:

1. JANA is designed to process events in parallel, when enabled. Factories are allowed to cache *local* state using member variables (e.g. calibrations and resources) Despite this, factories don't need to be thread-safe, because each factory is only used by one thread at a time. This means that JANA needs to spin up at least one instance of each factory for each thread. 

2. We want to be able to spin up separate instances of the same factory class (within the context of a single event and thread), and give them different parameter values and collection names. The way we manage these different instances is by giving each factory instance a unique prefix which will be used to namespace its parameters, inputs, outputs, and logger. This happens at the JFactoryGenerator level.

Here is how you set up a factory generator:

```c++
    app->Add(new JOmniFactoryGeneratorT<MC2SmearedParticle_factory>(
            "GeneratedParticles",
            {"MCParticles"},
            {"GeneratedParticles"},
            app
            ));
```

In this example, "GeneratedParticles" is the factory instance's unique tag, `{"MCParticles"}` is the list of input collection names, and `{"GeneratedParticles"}` is the list of output collection names. Some observations:

- If you are only creating one instance of this factory, feel free to use the "primary" output collection name as the factory prefix. (This has to be unique because PODIO collection names have to be unique.)

- Collection names are positional, so they need to be in the same order as the `PodioInput` and `VariationalPodioInput` declarations in the factory. 

- Variadic inputs are a little bit interesting: You can have any number of variadic inputs mixed in among the non-variadic inputs, as long as there are the same number of collection names for each variadic input. If this confuses you, just restrict yourself to one variadic input and put it as the very last input, like most programming languages do.

- When assigning names to collections and `JOmniFactory` prefixes, uniqueness is extremely important! JANA will throw an error if it detects a naming collision, but because of the dynamic plugin loading, some collisions can't be detected. DO NOT use the same output collection name or prefix in different plugins, and DO NOT rely the plugin loading order to make sure you ran the "correct" factory! To swap out different versions of a factory, change or override the *input* collection name on the factories and processors downstream.

## Where to put factory generators

Factory generators need to be added inside an `InitPlugin` for a particular plugin. If the factory generator is specific to one particular detector, it would go in that detector's plugin. Because electron reconstruction is not, the generator is set up in one of the plugins under `src/global`. Because this falls in the category of reconstruction, we put it in `src/global/reco/reco.cc`. 

## Calling the factory


#### To temporarily include your factory's outputs in the output file

On the command line, set the `podio:output_include_collections` parameter to include your collection names:

```bash
eicrecon -Ppodio:output_include_collections=MyNewCollectionName1,MyNewCollectionName2 in.root
```

#### To permanently include your factory's outputs in the output file:

Add your collection name to the `output_include_collections` list in src/services/io/podio/JEventProcessorPODIO.cc:44
```c++
    std::vector<std::string> output_include_collections={
            "EventHeader",
            "MCParticles",
            "CentralTrackingRecHits",
            "CentralTrackSeedingResults",
            "CentralTrackerMeasurements",
            //...
```

### To temporarily use your factory's outputs as inputs to another factory

```bash
eicrecon -Ptargetfactory:InputTags=MyNewCollectionName1,MyNewCollection2 in.root
```

### To permanently use your factory's outputs as inputs to another factory

Change the collection name in the `OmniFactoryGeneratorT` or `JChainMultifactoryGeneratorT`:
```c++
    app->Add(new JOmniFactoryGeneratorT<MC2SmearedParticle_factory>(
            "GeneratedParticles",
            {"MCParticlesSmeared"},            // <== Used to be "MCParticles"
            {"GeneratedParticles"},
            app
            ));
```

> Exercise:
> - Create a JOmniFactoryGenerator for your ElectronReconstruction factory
> - Give your factory's output collection a fun name
> - Call your factory from the command line and verify that you see its logger output.
> - Add it to the `JEventSourcePODIO::output_include_collections`, so that it gets called automatically.
> - Experiment with multiple factory generators so you can have multiple instances of the same factory
{: .challenge}


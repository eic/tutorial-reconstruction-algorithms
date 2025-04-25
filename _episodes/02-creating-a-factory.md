---
title: "Creating a factory"
teaching: 10
exercises: 1
objectives:
- "Understand the basics of EICrecon's plugin structure"
- "Understand where to put new factories"
- "Understand which factory base class to use"
- "Understand the JOmniFactory interface"
---

## Algorithms and Factories

We make a crisp distinction between *algorithms* and *factories*. 

*Algorithms* are classes that perform one kind of calculation we need and they do so in a generic, framework-independent way. *Factories*, in comparison, are classes that attach Algorithms to the JANA framework. You should write Algorithms to be independently testable, and you should write a Factory so that JANA can use the Algorithm within EICrecon. The factory layer handles issues like obtaining all of the inputs from other factories, publishing the outputs so that other factories can use them or they can be written to file, obtaining the correct parameters, making sure the Algorithm has been initialized, and making sure that the correct calibrations are loaded when the run number changes.

Here's an example to help illustrate what goes into the Algorithm and what goes into the Factory. Consider calorimeter clustering. The clustering algorithm should be independent of any individual detector, and have a set of parameters that control its behavior and live in a plain-old-data `Config` object. You could copy-paste this code into a codebase that uses a completely different reconstruction framework and it would still work, as long as you were using the same datamodel (e.g. `edm4hep`). Each detector could have its own factory (if it calls the algorithm in a substantially different way) or they may all use the same factory (if the factories only differ in their parameter values). The parameter values themselves could be hardcoded to the factory, but we strongly prefer to set them externally using a factory generator. This gives us a cleaner separation of configuration from code, and will let us do fun things in the future such as wiring factories together from an external config file, or performing parameter studies.


## The basics of EICrecon's plugin structure

JANA plugins are a mechanism for controlling which parts of `EICrecon` get compiled and linked together. They give us the ability to avoid having to compile and link heavy dependencies that not everybody will be using all the time. For instance, by default EICrecon uses ACTS for tracking, but perhaps someone wants to benchmark ACTS against Genfit -- we wouldn't want to have to ship Genfit inside eic-shell all the time.

Plugins were also designed so that users could integrate their analyses directly into reconstruction while keeping them independent and optional. This pattern is heavily used in the GlueX experiment and recommended in the tutorials on JANA's own documentation. In EICrecon, we set up separate plugins for each detector and each benchmark, but not for each analysis. We strongly recommend following the advice given in the analysis tutorials instead. The instructions for adding a new plugin are [here](https://eic.github.io/tutorial-jana2/03-end-user-plugin/index.html).



## Where to put new factories

The EICrecon plugins are organized as follows. Under `src/detectors` we have subdirectories for each individual detector, and each of them corresponds to one plugin that adds the detector's factory generators. Benchmarks are analogous. If an algorithm/factory will only ever be used in that one context, it can live there; otherwise, and preferably, the corresponding algorithm lives under `src/algorithms` and the corresponding factory lives under `src/factories`. 

Once you figure out which plugin your algorithm naturally belongs to, find its `InitPlugin()` method. By convention this lives in a `.cc` file with the same name as the plugin itself. This is where you will add your factory generator.

## Which factory base class to use


There are a number of different kinds of factories available in JANA which we have used within EICrecon at different points in time. Luckily, if you are writing an Algorithm from scratch, there is only one you will need to be familiar with: `JOmniFactory`. However, some of the earlier ones are still around, and just in case you need to modify or reuse those, here is a quick history lesson. 


`JFactoryT<T>` is JANA's fundamental factory base class. However, we don't use it in EICrecon because it has the following limitations:

1. It has difficulty with PODIO data. PODIO data needs very special handling, otherwise it will leak memory or corrupt your object associations. To address this, we developed `JFactoryPodioT`, which extends `JFactoryT` to support PODIO correctly.

2. *It can only output one collection.* This might seem fine at first, but frequently we need to output "Association" collections alongside the primary output collection. To address this, we developed `JMultifactory`, which supports multiple outputs, including PODIO data. 

3. If you want to reuse an Algorithm in a different context, you need to duplicate the JFactoryT/JPodioFactoryT/JMultifactory. Until this point, collection and parameter names were hardcoded inside individual factories. To get around this, we developed `JChainMultifactoryT` so that we could create multiple instances of the same factory and assign them different collection and parameter names in a logical way.

4. It requires a deeper understanding of JANA internals to use correctly. The user is allowed to perform actions inside the factory callbacks that don't necessarily make sense. We remedied this issue by developing `JOmniFactory`, which *declares* what it needs upfront, and JANA *provides* it only when it makes sense. `JOmniFactory` supports all of the functionality developed for points (1), (2), and (3), and presents a simpler interface. 


In summary, always use `JOmniFactory` if you are writing something new. All existing factories in EICrecon are in the process of being migrated right now: https://github.com/eic/EICrecon/issues/1176.


## The JOmniFactory interface


The basic idea behind an OmniFactory is to declare what you need upfront. That way, the framework can retrieve everything you need at the right time,
and it can handle complex namespacing logic behind the scenes so that you can dynamically rewire and reconfigure factories.

Earlier factory base classes, such as JChainMultifactory, require users to do a lot in their callbacks. Not so with JOmniFactory, which moves most of the functionality into registered members instead, as we shall discuss later. The callbacks are still there, but are made much simple, and focus on satisfying the underlying Algorithm's needs instead of JANA's.

These are the callbacks you'll need to implement:
```c++
    void Configure();
    void ChangeRun(int32_t run_number);
    void Process(int32_t run_number, uint64_t event_number);
```

`Configure` is called once when the factory is instantiated. This is where the user should initialize the underlying Algorithm. JANA will have already fetched the services, configured the logger, and set the values of the `Config` struct, so all the user needs to do is pass these things to the Algorithm.

`ChangeRun` is called once JANA detects that a new run has been started. This is where the user should update calibration data or other resources keyed off of the run number. JOmniFactory also provides a `Resource` registered member to automatically retrieve data from an arbitrary Service, though this is still experimental.

`Process` is called for every event. (Side note: Although note that because different threads have different factory instances, any individual factory cannot be guaranteed to witness the entire event stream. If you need to have one instance that processes the entire event stream, JANA provides JEventProcessors and JEventProcessorSequential for that purpose.) JANA will have already prefetched the registered Inputs before `Process` is called. The user needs to execute the Algorithm using those inputs, and copy the resulting outputs back to the registered Outputs. JANA will then take care of publishing the outputs downstream.

Note that unlike earlier factory base classes, JOmniFactory uses the [Curiously Recurring Template Pattern](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) so that the callback methods aren't virtual. This lets the optimizer get rid of any performance penalty for the extra layer of indirection.

Here is the full JOmniFactory code skeleton:

```c++

#pragma once
#include "extensions/jana/JOmniFactory.h"

class ReconstructedElectrons_factory : public JOmniFactory<ReconstructedElectrons_factory> {
private:

    // Declare inputs and outputs
    // PodioInput<edm4hep::MCParticle> m_in_mc_particles {this, "MCParticles"};
    // PodioOutput<edm4eic::ReconstructedParticle> m_out_reco_particles {this};

    // Declare parameters
    // ParameterRef<double> m_min_energy_over_momentum {this, "minEnergyOverMomentum", config().min_energy_over_momentum};
 
    // Declare services
    // Service<DD4hep_service> m_geoSvc {this};

public:
    void Configure() {
        // This is called when the factory is instantiated.
        // Use this callback to make sure the algorithm is configured.
        // The logger, parameters, and services have all been fetched before this is called
    }

    void ChangeRun(int64_t run_number) {
        // This is called whenever the run number is changed.
        // Use this callback to retrieve state that is keyed off of run number.
    }

    void Process(int64_t run_number, uint64_t event_number) {
        // This is called on every event.
        // Use this callback to call your Algorithm using all inputs and outputs
        // The inputs will have already been fetched for you at this point.
        // m_algo->execute(...);

        logger()->debug( "Event {}: Calling Process()", event_number );
    }
};
```

## The JOmniFactory inputs and outputs

The user specifies the JOmniFactory's inputs by declaring `PodioInput` or `VariationalPodioInput` objects as data members. These are templated on the basic PODIO type (Not the collection type or mutable type or object type or data type), and require the user to pass `this` as a constructor argument. These objects immediately register themselves with the factory, so that the factory always knows exactly what data it needs to fetch. To access the data once it has been fetched, the user can call the object's `operator()`, which returns a constant pointer to a PODIO collection of the correct type. For instance, suppose the user declares the data member:

```c++
PodioInput<MCParticles> m_particles_in {this};
```

In this case, the user would access the input data like this:

```c++
const MCParticlesCollection* particles_in = m_particles_in();
```

Of course, for brevity, the user could simply write this instead:
```c++
m_particles_out() = smearing_algo->execute( m_particles_in() );
```

As you have just seen, PodioOutputs are very analogous to PodioInputs. 


> Exercise:
> - Create your own ElectronReconstruction factory from the code skeleton above
> - Give your OmniFactory a single output collection
> - Have its Process() method produce some log output
> - Experiment with giving it different input collections
{: .challenge}

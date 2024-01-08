---
title: "Approach"
teaching: 10
exercises: 0
questions:
objectives:
- "Understand relevent EICrecon concepts/considerations"
keypoints:
---

## Working with PODIO objects


## Algorithms vs. factories

We make a crisp distinction between *algorithms* and *factories*. 

### What is an Algorithm?

*Algorithms* are classes that perform one kind of calculation we need and they do so in a generic, framework-independent way. The core of an Algorithm is a method called `execute` which inputs some PODIO collections and outputs some other PODIO collections. Algorithms don't know or care where the inputs come from and where they go. Algorithms also don't know much about where their parameters come from; rather, they are passed a `Config` structure which contains the parameters' values. The nice thing about algorithms is that they are simple to design and test, and easy to reuse for different detectors, frameworks, or even entire experiments.


Most of what makes an Algorithm an Algorithm is convention. (These are largely inspired by the KISS principle in software engineering!) There is an ongoing effort to create a "framework-less framework" for formally expressing Algorithms using templates, which lives at https://github.com/eic/algorithms. Eventually, we may encourage users to have all Algorithms inherit from the `Algorithm<Input<...>, Output<...>>` templated interface. For now, however, just follow the Algorithm conventions that we will go over later.


### What is a Factory?

*Factories* are classes that attach Algorithms to the JANA framework. Once you have written an Algorithm, you should write a Factory so that JANA can use it. The factory layer handles issues like obtaining all of the inputs from other factories, publishing the outputs so that other factories can use them or they can be written to file, obtaining the correct parameters, making sure the Algorithm has been initialized, and making sure that the correct calibrations are loaded when the run number changes.


Here's an example to help illustrate what goes into the Algorithm and what goes into the Factory. Consider calorimeter clustering. The clustering algorithm should be independent of any individual detector, and have a set of parameters that control its behavior and live in a plain-old-data `Config` object. You could copy-paste this code into a codebase that uses a completely different reconstruction framework and it would still work, as long as you were using the same datamodel (e.g. `edm4hep`). Each detector could have its own factory (if it calls the algorithm in a substantially different way) or they may all use the same factory (if the factories only differ in their parameter values). The parameter values themselves could be hardcoded to the factory, but we strongly prefer to set them externally using a factory generator. This gives us a cleaner separation of configuration from code, and will let us do fun things in the future such as wiring factories together from an external config file, or performing parameter studies.


## Which factory base class should I use?

There are a number of different kinds of factories available in JANA which we have used within EICrecon at different points in time. Luckily, if you are writing an Algorithm from scratch, there is only one you will need to be familiar with: `JOmniFactory`. However, some of the earlier ones are still around, and just in case you need to modify or reuse those, here is a quick history lesson. 


`JFactoryT<T>` is JANA's fundamental factory base class. However, we don't use it in EICrecon because it has the following limitations:

1. It has difficulty with PODIO data. PODIO data needs very special handling, otherwise it will leak memory or corrupt your object associations. To address this, we developed `JFactoryPodioT`, which extends `JFactoryT` to support PODIO correctly.

2. *It can only output one collection.* This might seem fine at first, but frequently we need to output "Association" collections alongside the primary output collection. To address this, we developed `JMultifactory`, which supports multiple outputs, including PODIO data. 

3. If you want to reuse an Algorithm in a different context, you need to duplicate the JFactoryT/JPodioFactoryT/JMultifactory. Until this point, collection and parameter names were hardcoded inside individual factories. To get around this, we developed `JChainMultifactoryT` so that we could create multiple instances of the same factory and assign them different collection and parameter names in a logical way.

4. It requires a deeper understanding of JANA internals to use correctly. The user is allowed to perform actions inside the factory callbacks that don't necessarily make sense. We remedied this issue by developing `JOmniFactory`, which *declares* what it needs upfront, and JANA *provides* it only when it makes sense. `JOmniFactory` supports all of the functionality developed for points (1), (2), and (3), and presents a simpler interface. 


In summary, always use `JOmniFactory` if you are writing something new. All existing factories in EICrecon are in the process of being migrated right now: https://github.com/eic/EICrecon/issues/1176.


## What are the basic pieces of an OmniFactory?

The basic idea behind an OmniFactory is to declare what you need upfront. That way, the framework can retrieve everything you need at the right time,
and it can handle complex namespacing logic behind the scenes so that you can dynamically rewire and reconfigure factories.

### Registered members

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

Parameters are handled slightly differently. JOmniFactory provides a `Parameter` class template which can hold , but in EICrecon we use Config structs
// TODO: Discuss variadic Inputs
// TODO: Discuss Parameters
// TODO: Discuss Services


### Callbacks
Earlier factory base classes, such as JChainMultifactory, require users to do a lot in their callbacks. Not so with JOmniFactory, which moves most of the functionality into the registered members instead. The callbacks are still there, but are made much simple, and focus on satisfying the underlying Algorithm's needs instead of JANA's.

These are the callbacks you'll need to implement:
```c++
    void Configure();
    void ChangeRun(int64_t run_number);
    void Process(int64_t run_number, uint64_t event_number);
```

`Configure` is called once when the factory is instantiated. This is where the user should initialize the underlying Algorithm. JANA will have already fetched the services, configured the logger, and set the values of the `Config` struct, so all the user needs to do is pass these things to the Algorithm.

`ChangeRun` is called once JANA detects that a new run has been started. This is where the user should update calibration data or other resources keyed off of the run number. JOmniFactory also provides a `Resource` registered member to automatically retrieve data from an arbitrary Service, though this is still experimental.

`Process` is called for every event. (Side note: Although note that because different threads have different factory instances, any individual factory cannot be guaranteed to witness the entire event stream. If you need to have one instance that processes the entire event stream, JANA provides JEventProcessors and JEventProcessorSequential for that purpose.) JANA will have already prefetched the registered Inputs before `Process` is called. The user needs to execute the Algorithm using those inputs, and copy the resulting outputs back to the registered Outputs. JANA will then take care of publishing the outputs downstream.

Note that unlike earlier factory base classes, JOmniFactory uses the [Curiously Recurring Template Pattern](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern) so that the callback methods aren't virtual. This lets the optimizer get rid of any performance penalty for the extra layer of indirection.



## I've created an Algorithm and a corresponding OmniFactory. How do I run it?

There are several steps which we will go over in more detail in turn. First, you figure out which plugin should provide the factory, and find the plugin's `InitPlugin` function. Inside `InitPlugin`, you want to create a new `JOmniFactoryGeneratorT` and add it to the `JApplication`. This is where you set up your input and output collection names - make sure your output collection names are unique. Finally, adding the factory won't cause the factory to run unless someone else asks for it to run, so we either use some of the new factory's outputs as inputs to other factories, or we add the outputs to the list of collections to write to the output file for subsequent analysis.


### What are plugins, and which one should I use?

JANA plugins are a mechanism for controlling which parts of `EICrecon` get compiled and linked together. They give us the ability to avoid having to compile and link heavy dependencies that not everybody will be using all the time. For instance, by default EICrecon uses ACTS for tracking, but perhaps someone wants to benchmark ACTS against Genfit -- we wouldn't want to have to ship Genfit inside eic-shell all the time.

Plugins were also designed so that users could integrate their analyses directly into reconstruction while keeping them independent and optional. This pattern is heavily used in the GlueX experiment and recommended in the tutorials on JANA's own documentation. In EICrecon, we set up separate plugins for each detector and each benchmark, but not for each analysis. We strongly recommend following the advice given in the analysis tutorials instead. The instructions for adding a new plugin are [here](https://eic.github.io/tutorial-jana2/03-end-user-plugin/index.html).

The EICrecon plugins are organized as follows. Under `src/detectors` we have subdirectories for each individual detector, and each of them corresponds to one plugin that adds the detector's factory generators. Benchmarks are analogous. If an algorithm/factory will only ever be used in that one context, it can live there; otherwise, and preferably, the corresponding algorithm lives under `src/algorithms` and the corresponding factory lives under `src/factories`. There is also a `src/global/{pid,reco,tracking}` and some factories live there. TODO: Why both `src/global` and `src/factories`?

Once you figure out which plugin your algorithm naturally belongs to, find its `InitPlugin()` method. By convention this lives in a `.cc` file with the same name as the plugin itself. This is where you will add your factory generator.


### What is a factory generator?
Instead of handing over the OmniFactory to JANA directly, we create a `JOmniFactoryGeneratorT` which is basically a recipe that JANA uses to create factories by itself. There are several reasons we need this:

1. JANA is designed to process events in parallel, when enabled. Factories are allowed to cache *local* state using member variables (e.g. calibrations and resources) Despite this, factories don't need to be thread-safe, because each factory is only used by one thread at a time. This means that JANA needs to spin up at least one instance of each factory for each thread. 

2. We want to be able to spin up separate instances of the same factory class (within the context of a single event and thread), and give them different parameter values and collection names. The way we manage these different instances is by giving each factory instance a unique prefix which will be used to namespace the parameters, collections, and logger. This happens at the JFactoryGenerator level.

Here is how you set up a factory generator:

```c++
    app->Add(new JOmniFactoryGeneratorT<MC2SmearedParticle_factory>(
            "GeneratedParticles",
            {"MCParticles"},
            {"GeneratedParticles"},
            app
            ));
```

In this example, "GeneratedParticles" is the factory instance's unique prefix, `{"MCParticles"}` is the list of input collection names, and `{"GeneratedParticles"}` is the list of output collection names. Some observations:

- If you are only creating one instance of this factory, feel free to use the "primary" output collection name as the factory prefix. (This has to be unique because PODIO collection names have to be unique.)

- Collection names are positional, so they need to be in the same order as the `PodioInput` and `VariationalPodioInput` declarations in the factory. 

- Variadic inputs are a little bit interesting: You can have any number of variadic inputs mixed in among the non-variadic inputs, as long as there are the same number of collection names for each variadic input. If this confuses you, just restrict yourself to one variadic input and put it as the very last input, like most programming languages do.

- When assigning names to collections and `JOmniFactory` prefixes, uniqueness is extremely important! JANA will throw an error if it detects a naming collision, but because of the dynamic plugin loading, some collisions can't be detected. DO NOT use the same output collection name or prefix in different plugins, and DO NOT rely the plugin loading order to make sure you ran the "correct" factory! To swap out different versions of a factory, change or override the *input* collection name on the factories and processors downstream.


### How do I get my factory to run, and how do I get my output?

#### To temporarily include your factory's outputs in the output file

```bash
eicrecon -Ppodio:output_include_collections=MyNewCollectionName1,MyNewCollectionName2 in.root
```
Note that this *appends* our collections to the list of collections that get written.
TODO: Look this up

#### To permanently include your factory's outputs in the output file:
```bash
eicrecon -Ppodio:output_include_collections=MyNewCollectionName in.root
```
TODO: Look this up

#### To temporarily use your factory's outputs in another factory:
```bash
eicrecon -Ppodio:output_include_collections=MyNewCollectionName in.root
```
TODO: Look this up

#### To permanently use your factory's outputs in another factory:
```bash
eicrecon -Ppodio:output_include_collections=MyNewCollectionName in.root
```
TODO: Look this up



## What is the future of OmniFactories and Algorithms?

Algorithms will gradually become more structured, so that the inputs, outputs, parameters, services, and callbacks will all be declared in a way that 
other code will be able to introspect them. This will pave the way to create a single generalized JOmniFactory that automatically knows what to do with any 
Algorithm. This means that eventually you won't need to write your own factories anymore, and that if you write an Algorithm that adheres to the formal Algorithm interface, it can be used as-is within both JANA and Gaudi.

Overall this moves us towards a future where we write software that is interoperable and reusable, even between experiments and collaborations. This is part of a broader effort that includes [ACTS](https://acts.readthedocs.io/en/latest/) and [key4hep](https://github.com/key4hep).







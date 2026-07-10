---
title: "Working with PODIO"
teaching: 5
exercises: 1
---

::::::::::::::::::::::::::::::::::::::::::::: questions

- What is PODIO and why does the EIC data model use it?
- How do I create and fill PODIO collections, and what is a subset collection?

:::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::: objectives

- Gain familiarity working with PODIO collections.
- Understand PODIO subset collections.

:::::::::::::::::::::::::::::::::::::::::::::

## Introduction to PODIO

Our data model is in a library/namespace/repository called `edm4eic`, and it is built on top of `edm4hep`, a data model designed to capture commonalities across HEP experiments. `edm4eic` is implemented using PODIO, which is a toolkit for generating the data model classes from a specification written in YAML. Here is a very simple example of a PODIO specification:

```yaml
options :
  # should getters / setters be prefixed with get / set?
  getSyntax: False
  # should POD members be exposed with getters/setters in classes that have them as members?
  exposePODMembers: True
  includeSubfolder: True

datatypes :
  ExampleHit :
    Description : "Hit"
    Author : "B. Hegner"
    Members:
      - unsigned long long cellID      // cellID
      - double x      // x-coordinate
      - double y      // y-coordinate
      - double z      // z-coordinate
      - double energy // measured energy deposit

  ExampleCluster :
    Description : "Cluster"
    Author : "N. Brei"
    Members:
      - double energy // cluster energy
    OneToManyRelations:
      - ExampleHit Hits // hits contained in the cluster
      - ExampleCluster Clusters // sub clusters used to create this cluster
```

PODIO will then generate for us the following classes:
```
DatamodelDefinition.h ExampleCluster.h ExampleClusterCollection.h ExampleClusterCollectionData.h ExampleClusterData.h ExampleClusterObj.h ExampleHit.h ExampleHitCollection.h ExampleHitCollectionData.h ExampleHitData.h ExampleHitObj.h MutableExampleHit.h MutableExampleCluster.h
```

As you can see, PODIO has a lot of moving pieces. Why? 

1. PODIO adds a separate layer for managing memory in a way which is more consistent with Python and other garbage-collected languages. The user only has to work with values, no explicit allocations or deletions.
2. PODIO separates the data's memory layout from its accessors
3. PODIO enforces immutability directly in the object model
4. PODIO has sophisticated (though fragile!) mechanisms for tracking object references

These design principles in principle should eliminate entire classes of bugs. However, there are still subtleties when using PODIO that can lead to leaks, crashes, or corrupted references. Luckily, the correct usage pattern is quite simple, as we will discuss next.


## Working with PODIO objects, collections, and subset collections:


```c++
auto hits = std::make_unique<ExampleHitCollection>();

hits->push_back(ExampleHit(22, 0.0, 0.0, 0.0, 0.001));

MutableExampleHit hit;
hit.x(0.0);
hit.energy(0.001);
// ...
hits->push_back(hit);

MutableExampleCluster cluster;
cluster.addHits(hit);

// Safety tip: Add object to a collection BEFORE creating an association to it

auto clusters = std::make_unique<ExampleClusterCollection>();
clusters->push_back(cluster);

auto subset_clusters = std::make_unique<ExampleClusterCollection>();
subset_clusters->setSubsetCollection(true);
subset_clusters->push_back(cluster);

// Safety tip: Every PODIO object is owned by exactly one collection.
// If you want to put the object in other collections, those collections need to 
// be designated as "subset collections", which means that they don't own their contents. 
```


Note that when you write a factory, its inputs will be `const ExampleHitCollection*`, which are *immutable*.
Its output is held by a `PodioOutput<ExampleHit>` member; calling `m_output()` returns a `std::unique_ptr<ExampleHitCollection>&` that the factory can mutate, and `m_output().get()` gives a raw mutable pointer that you can hand to your algorithm's `process()` method. After `Process()` returns, JOmniFactory transfers ownership of that collection to JANA2, which adds it to a podio `Frame`. From that point on, the collection is immutable and owned by the `Frame`.

JANA2 will create and destroy `Frame`s internally. 


::::::::::::::::::::::::::::::::::::::::::::: challenge

## Exercise

- Have your algorithm produce some (fake) output data!

::::::::::::::: solution

In your algorithm's `process()`, create mutable objects, set some field values, and `push_back`
them onto the output collection (or, for an electron finder, mark the output as a subset collection
with `setSubsetCollection()` and push existing input objects onto it). Running EICrecon and
inspecting the output collection should now show your fabricated entries.

:::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::: keypoints

- `edm4eic` (on top of `edm4hep`) is generated by PODIO from a YAML specification.
- PODIO manages memory, separates layout from accessors, and enforces immutability, but object references are fragile.
- Every PODIO object is owned by exactly one collection; use a *subset collection* (`setSubsetCollection()`) to reference objects owned elsewhere.
- Factory inputs are immutable `const *`; the output collection becomes immutable once JANA2 moves it into a `Frame`.

:::::::::::::::::::::::::::::::::::::::::::::

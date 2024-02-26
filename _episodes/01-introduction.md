---
title: "Introduction: EICrecon algorithms for physics analysis"
teaching: 5
exercises: 0
questions:
objectives:
- "Define physics goal of new reconstruction algorithm"
- "Identify what information is needed to accomplish this goal"
keypoints:
---

## Background

### The simulation-reconstruction-analysis pipeline

The ePIC software pipeline consists of the following stages.

1. Event generation
2. Simulation
3. Digitization (eventually)
4. Reconstruction
5. Analysis

This tutorial focuses exclusively on the reconstruction stage and the EICrecon software package. At the highest conceptual level, EICrecon's inputs are simulated (and eventually, optionally digitized) detector hits, and the outputs are reconstructed tracks and particle identifications. Internally, EICrecon uses a data model and software architecture which is designed to cleanly separate dozens of different reconstruction algorithms and facilitate reconfiguring them and reusing them according to users' needs.


### Levels of user interaction with reconstruction

Depending on how a user contributes to ePIC, their level of interaction with EICrecon will vary. We identify four distinct patterns, or levels of interaction:

1. Performing analyses using the reconstructed data as-is
2. Modifying an algorithm's parameters and re-running the reconstruction
3. Applying an existing algorithm to a fresh context
4. Adding a new algorithm

This tutorial is designed to be most helpful for users doing (3) or (4), although users at all levels will benefit from the deeper understanding of EICrecon's architecture.

## Motivating example: simple electron ID with E/p cut

One key ingredient in electron ID is the ratio of the energy deposited in the calorimeter (E) to the momentum of the particle track (p).  For electrons, this should be close to 1.  We want to implement a rudimentary electron ID algorithm by identifying particles that satisfy 0.9 < E/p < 1.2.

### What information is required?

This simple electron ID algorithm requires three pieces of information, which will be obtained from pre-existing algorithms/factories:
- Reconstructed tracks
- Calorimeter clusters
- Matching between tracks and clusters

Matching between tracks and clusters can be obtained from truth information, or from track projections.  
- In the former case, the reconstructed tracks and clusters are matched through their association to the same MC particle.  
- In the latter case, a matching criteria is applied to the position of the calorimeter cluster and the projected position of the track at the calorimeter.  
For the purposes of this tutorial, we will use truth matching.

The input and output objects of our factory should be stored as PODIO collections.  
- Reconstructed tracks are stored as a collection of `edm4eic::ReconstructedParticle`
- Calorimeter clusters are stored as a collection of `edm4eic::Cluster`
- Associations between reconstructed tracks and MC particles are stored as a collection of `edm4eic::MCRecoParticleAssociation`
- Associations between calorimeter clusters and MC particles are stored as a collection of `edm4eic:MCRecoClusterParticleAssociation`

The members of each of these data types can be found in `edm4eic.yaml` in the `EDM4eic` repository. 

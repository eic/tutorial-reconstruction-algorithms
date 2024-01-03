---
title: "Introduction"
teaching: 5
exercises: 0
questions:
objectives:
- "Define physics goal of new reconstruction algorithm"
- "Identify what information is needed to accomplish this goal"
keypoints:
---

## Example: simple electron ID with $E/p$ cut

One key ingredient in electron ID is the ratio of the energy deposited in the calorimeter ($E$) to the momentum of the particle track ($p$).  For electrons, this should be close to 1.  We want to implement a simple electron ID algorithm by identifying particles that satisfy $0.9 < E/p < 1.2$.

### What information is required?

This simple electron ID algorithm requires three pieces of information, which will be obtained from pre-existing algorithms/factories:
- Reconstructed tracks
- Calorimeter clusters
- Association between tracks and clusters

The association between tracks and clusters can be obtained from truth information, or by applying some matching criteria between cluster position and track projections.  For the purposes of this tutorial, we will use the truth information.


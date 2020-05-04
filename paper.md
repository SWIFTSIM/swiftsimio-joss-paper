---
title: "swiftsimio: A python library for reading SWIFT data"
tags:
  - Python
  - astronomy
  - cosmology
  - simulations
  - i/o
authors: 
  - name: Josh Borrow
    orcid: 0000-0002-1327-1921
    affiliation: 1
affiliations:
  - name: Institute for Computational Cosmology, Durham University
    index: 1
date: 4 May 2020
bibliography: bibliography.bib
---

# Summary

`swiftsimio` is a python package for reading data created by the SWIFT [@SWIFT]
simulation code. SWIFT is designed to run cosmological hydrodynamics
simulations that produce petabytes of data, and `swiftsimio` leverages the huge
amounts of custom metadata that SWIFT produces to allow for chunked loading of
the particle data and to enable integration with `unyt` [@unyt].

# Background

Cosmological galaxy formation simulations have been used for decades now to
enable the understanding of the process of galaxy formation. As time has
progressed, so has the scale of these simulations. The state-of-the-art
simulations planned for the next decade, using codes like SWIFT, will generate
petabytes of data thanks to their use of hundreds of billions of particles.
Analysing this data presents a unique challenge, as typically the data
reduction is performed either on single compute nodes or even on individual
desktop machines. 

In the original EAGLE simulation [@EAGLE], just the co-ordinates of the gas
particles in a single snapshot used 82 Gb of storage space. State-of-the-art
simulations are now using at least 10 times as many particles as this, making
an analysis pipeline that reads the whole array each time expensive in the best
case and infeasible in the worst. There are two useful properties of this data:
it is stored in the HDF5 format, allowing for easy slicing, and usually users
are interested in a very small sub-set of the data, usually less than 1%, at a
time.

This requirement to load less than 1% of the data at a time is primarily due to
the huge amount of dynamic range in these simulations [@dynamicRange]. Although
the simulation volume may be hundreds of megaparsecs on a side, the objects
that form under self-gravity are typically less than a few megaparsecs in
diameter, representing a very small proportion of the total volume. Users are
usually interested in using the particle data present in a few objects
(selected using pre-computed data in 'halo catalogues') at any given time.

# Structure of a SWIFT snapshot

At pre-determined times during a SWIFT simulation, a full particle dump is
performed.  This particle dump is stored in the HDF5 format [@hdf5], with
arrays corresponding to different properties, with the overall structure of the
file being compatible with the Gadget-2 format [@Gadget2].

An example snapshot would be as follows:
```
eagle_0003.hdf5
├Cells 
├Code (15 attributes)
├Cosmology (22 attributes)
├GravityScheme (19 attributes)
├Header (14 attributes)
├HydroScheme (36 attributes)
├InternalCodeUnits (5 attributes)
├Parameters (295 attributes)
├PartType0
│ ├Coordinates	[float64: 49615231 × 3] (11 attributes)
| ⋮
│ └ViscosityParameters	[float32: 49615231] (11 attributes)
├PartType1
│ ├Coordinates	[float64: 53157376 × 3] (11 attributes)
| ⋮
│ └Velocities	[float32: 53157376 × 3] (11 attributes)
├Policy (22 attributes)
├StarsScheme (8 attributes)
├SubgridScheme (13 attributes)
├Units (5 attributes)
└UnusedParameters (1 attributes)
```
Each of the `PartType` sections include particle data for different types. The
table below shows what each particle type corresponds to and how it is accessed
in `swiftsimio`.

| Particle type | Description                                                                                     | `swiftsimio` name |
|---------------|-------------------------------------------------------------------------------------------------|-------------------|
| 0             | Gas particles, the only type of particles to have hydrodynamics calculations performed on them. | `gas`             |
| 1             | Dark matter particles, only feel the force of gravity.                                          | `dark_matter`     |
| 2             | Boundary particles; the same as dark matter but typically more massive.                         | `boundary`        |
| 3             | Boundary particles; the same as dark matter but typically more massive.                         | `second_boundary` |
| 4             | Star particles representing either individual stars or a stellar population.                    | `stars`           |
| 5             | Black hole particles representing individual black holes.                                       | `black_holes`     |

The rest of the fields in the above data are metadata fields, and are used to
include information about the physics that is included as well as the current
global state (e.g.  time) of the simulation.

This metadata also allows an `unyt` array to be created for each field,
ensuring consistent units throughout all analysis. A custom cosmology object
also ensures that the differences between quantities co-moving with the
expansion of the universe and those stored in 'physical' co-ordinates are
maintained.

# Solving the data reduction challenge

To solve this dynamic range problem, we can turn to the spatial arrangement of
the data. Simulations in SWIFT are performed in a cuboid volume that is split
into a fixed number of top-level cells. When a snapshot of the simulation is
dumped, each array is written to disk top-level cell by top-level cell,
ensuring a well-characterised order.  The order in which the data is stored is
then written as metadata in the `Cells` dataset.  The `SWIFTMask` object within
`swiftsimio` uses this metadata to provide a mask to be used with the `h5py`
library to read a sub-set of the particle data. This process ensures that as
little data is read from disk as possible.

# Why swiftsimio?

There are many python libraries that are able to read the Gadget-style
formatted HDF5 data that SWIFT outputs, not limited to but including `yt`
[@yt], `pynbody` [@pynbody] and `pnbody` [@pnbody]. The other option is simply
to use the `h5py` library directly, and forgo any extra processing of the data.

`swiftsimio` was created because these libraries either are too slow (usually
performing significant pre-calculation when loading a snapshot), provide too
weak of a connection to the data in the snapshot (e.g. reading the data in a
different order than it is stored in the file), or would have been unable to
integrate with the metadata that SWIFT outputs.  To make full use of the SWIFT
metadata all of these packages would have had to have significant re-writes of
their internals, which would have taken significantly more time (due to their
more complex codebases) or would have been unwelcome changes due to their
impact on the users of other simulation codes.

# Acknowledgements


# References

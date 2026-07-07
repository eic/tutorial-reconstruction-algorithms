# Writing Reconstruction Algorithms in EICrecon

[![The Carpentries Workbench](https://img.shields.io/badge/Built%20with-The%20Carpentries%20Workbench-071159.svg)](https://carpentries.github.io/sandpaper-docs/)

An ePIC tutorial on implementing reconstruction algorithms in
[EICrecon](https://github.com/eic/EICrecon). It covers JANA2 factories, the `JOmniFactory`
interface, parameterizing factories, writing framework-independent algorithms, and working with
PODIO collections, culminating in a working electron-finder.

This lesson is built with [The Carpentries Workbench](https://carpentries.github.io/sandpaper-docs/).

## Building the lesson locally

The lesson is rendered with the Workbench Docker image (no local R installation needed). A
`Makefile` wraps the commands:

```bash
make preview   # build the site into site/docs/
make serve     # serve site/docs/ at http://localhost:4321
make clean     # drop the build cache
```

## Contributing

We welcome all contributions to improve the lesson! Please familiarize yourself with our
[Contribution Guide](CONTRIBUTING.md). For the Workbench Markdown syntax (callouts, challenges,
etc.) see the [sandpaper documentation](https://carpentries.github.io/sandpaper-docs/).

## Maintainer(s)

Current maintainers of this lesson are

* Tyler Kutz, @tkutz, tkutz@mit.edu

## Citation

To cite this lesson, please consult the [CITATION.cff](CITATION.cff) file.

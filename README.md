[![Website](https://img.shields.io/badge/dmtn--348-lsst.io-brightgreen.svg)](https://dmtn-348.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-348/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-348/actions/workflows/ci.yaml)

# Nightly Ephemerides Cache Generation and Service at the USDF

## DMTN-348

This technote documents the design and implementation of Rubin Observatory solar system small body ephemerides generation system. The system comprises of two main components, mpsky and ephemcache. mpsky answers which small bodies lie within a given field of view at a given time, using nightly ephemerides caches precomputed from the Minor Planet Center orbit catalog. This note describes the architecture, configuration, measured performance, and known operational gaps of this system

**Links:**

- Publication URL: https://dmtn-348.lsst.io
- Alternative editions: https://dmtn-348.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-348
- Build system: https://github.com/lsst-dm/dmtn-348/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-dm/dmtn-348
cd dmtn-348
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-348.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-348.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.

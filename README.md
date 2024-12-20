[![Website](https://img.shields.io/badge/sqr--097-lsst.io-brightgreen.svg)](https://sqr-097.lsst.io)
[![CI](https://github.com/lsst-sqre/sqr-097/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-sqre/sqr-097/actions/workflows/ci.yaml)

# Integrating QServ HTTP API into RSP TAP services

## SQR-097

The existing CADC-based TAP services that the RSP deployes use JDBC connections to run queries on the QServ catalog data. This has certain limitations related to being able to asynchronously drive queries (i.e. stopping, getting info on). The QServ team have developed a prototype HTTP REST API which can be used to drive queries. This technote proposes a design where we modify/extend the existing TAP service software to enable integration and use of this new API to allow driving queries via HTTP.

**Links:**

- Publication URL: https://sqr-097.lsst.io
- Alternative editions: https://sqr-097.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-097
- Build system: https://github.com/lsst-sqre/sqr-097/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-sqre/sqr-097
cd sqr-097
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://sqr-097.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-097.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.

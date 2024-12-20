# Integrating QServ HTTP API into RSP TAP services

```{abstract}
The existing CADC-based TAP services that the RSP deployes use JDBC connections to run queries on the QServ catalog data. This has certain limitations related to being able to asynchronously drive queries (i.e. stopping, getting info on). The QServ team have developed a prototype HTTP REST API which can be used to drive queries. This technote proposes a design where we modify/extend the existing TAP service software to enable integration and use of this new API to allow driving queries via HTTP.
```

## Add content here

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.

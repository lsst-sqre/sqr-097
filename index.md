# Integrating QServ HTTP API into RSP TAP services

```{abstract}
The existing CADC-based TAP services that the RSP deploys use JDBC connections to run queries on the QServ catalog data. The current setup lacks crucial query control capabilities - specifically, the ability to stop or monitor long-running queries over large datasets. The QServ team have developed a prototype HTTP REST API which can be used to drive queries. This technote proposes a design where we modify and extend the existing TAP service software to enable integration and use of this new API to allow driving queries via HTTP.
```

## 1. Introduction

### 1.1 Current architecture
   	
The current implementation connects the CADC TAP service to QServ through a JDBC interface. The service is built using the CADC TAP library and implements the TAP 1.1 protocol. 
Database connectivity is configured via a JNDI DataSource (jdbc/qservuser), utilizing Apache Tomcat's connection pooling for persistent JDBC connections to QServ, with the parameters set in the server configuration. 

The query execution flow starts when a query is received through the TAP interface, which is then processed by the `QServQueryRunner` class and executed via the JDBC connection. 
Upon completion, results are written to GCS storage and made accessible to users through the UWS result interface, which provides a redirect to the storage location through a Redirect Servlet that is a Rubin specific implementation rather than part of the CADC TAP library.

### 1.2 Motivation for change

While this architecture is functional and serves most use cases effectively, it has some limitations in our specific context. QServ operates as a distributed database system handling large datasets, where queries can potentially run for extended periods. In this environment, it is very likely that users might submit inefficient queries that consume substantial resources over long periods.

These types of queries have the potential to overwhelm the system at high concurrency and the current JDBC-based implementation lacks core operational capabilities that would help alleviate this, specifically a mechanism to stop executing queries once started and monitoring capabilities for users for these long-running queries.

The use-case scenario we would like to support that is not currently possible is allowing users to identify that a query has been running longer than they expected, to stop this potentially inefficient query, modify its parameters and restart it.
                                                
The new QServ HTTP API makes it possible to address these limitations by providing native query cancellation support, real-time progress monitoring and improved status reporting. 

Beyond addressing immediate operational needs, we identify components within the TAP library where it would be possible to abstract the underlying storage solution from the TAP execution layer, allowing us, as well as other operators of the CADC TAP service to more flexibly move between different solutions if needed.


### 1.3 Goals and Requirements

#### Goals

The primary goals for this transition are: 

- Enhance control over query execution 
- Enable a core feature of allowing cancellation of running queries
- Abstractions in the CADC TAP service that separate the query execution layer from the underlying storage implementation 
- Get detailed information about query execution such as progress metrics & status updates.
- Backward Compatibility. Also any changes should be transparent to the user and not change any of the observed behaviour of the TAP service

#### Requirements

- Integration with the existing UWS job management system & support for both synchronous and asynchronous query execution 
- Efficient handling of large result sets 
- Proper error handling and status reporting 
- Ability to scale to a large number of concurrent users (1-10k initially)


## 2. QServ HTTP API Analysis

### 2.1 Overview

QServ's HTTP API provides a REST interface for managing QServ queries including running, monitoring and control. 
The API supports both synchronous and asynchronous query execution and provides descriptive status reports and query management capabilities.
The interface is version-controlled through either a pull or push mode described in the [QServ](https://qserv.lsst.io/user/http-frontend.html) documentation.


### 2.2 Key Features

The API offers several key capabilities essential for queries to the Rubin catalog data:

#### Query Execution Control

The API provides both synchronous and asynchronous query interfaces. The asynchronous interface offers a query ID that can be used for monitoring and control. 

#### Progress Monitoring

The status endpoint provides detailed execution statistics, including the number of completed chunks, execution start time, and last update time. 
Status information is available through `/query-async/status/:queryId`, providing: 

- Total and completed chunk counts 
- Query start and last update timestamps 
- Execution state information 
- Distributed processing metrics

#### Result Retrieval 

Results are accessed through `/query-async/result/:queryId` with support for: 

- Schema information including column types 
- Row-based data representation 
- Multiple binary data encoding options (hex, base64, array)

In the current implementation, once results are fetched they are deleted on the QServ side. This means that attempts to refetch the data will fail with an error message. This requires careful consideration depending on our approach and specifically in the case where the TAP service requests and streams the results from QServ to TAP via this endpoint.

#### Query Management

The API includes endpoints for query cancellation. This allows users (or an admin service or person) to terminate queries that are taking too long or consuming excessive resources.

The API provides query cancellation through a DELETE request to `/query-async/<queryId>`. For example, to cancel query 123: 
```
HTTP DELETE /query-async/123?version=39
```

A successful cancellation returns:

```
{
    "success": 1
}
```

The query state will transition to ABORTED, and this state change can be observed through the status endpoint.


#### User Table Management

The QServ HTTP API also provides functionality for managing user-defined tables in QServ. 
This includes table creation, data ingestion, and deletion operations. 

Two options for ingesting table data are supported:

- `application/json`: Both data and metadata are encapsulated into a single JSON object sent in the request body.

- `multipart/form-data`: The data is sent as a CSV-formatted file, and metadata is encapsulated in separate JSON objects. All this information is packaged into the body of a request.

The API enforces naming conventions for user databases and tables to prevent conflicts with production data. 
User databases must follow the pattern `user_<username>`, while table names cannot start with the `qserv_` prefix which is reserved for internal QServ tables.

There are both synchronous and asynchronous modes through which tables can be ingested and the service provides options for defining schema, index creation, binary data handling though there are some limitations in terms of table sizes when using the JSON ingestion method.

This endpoint could be relevant to the RSP TAP services both in terms of providing TAP_UPLOAD for TAP services that are backed by a QServ database, as well as for user-uploaded tables. Both cases would require development effort to enable use of the API in the TAP library code, and would probably benefit from a standalone technote outlining the steps required.


### 2.3 Proposed Changes

While the API provides essential functionality, there are a few areas which could be adjusted to make integration with the RSP (CADC) TAP service easier:


#### VOTable Support

The current API returns results in a JSON format that, while comprehensive, requires conversion into XML (VOTable) for TAP compatibility. We recommend:
- Adding direct VOTable output support for query results
- Including proper astronomical metadata (units, UCDs)
- Supporting different serialization formats (binary2, tabledata). We don't necessarily need both and binary2 would be the preferred option, though if it is possible to have both as optional formats that would be ideal
- Maintaining JSON output for the rest of the query management API

Adding VOTable support is a key change that allows us to skip an extra deserialization / serialization step.

#### Storing results

A main difference of the architecture proposed here is that handling the results now shifts towards the QServ Service. Specifically QServ upon completion would write the results out to the GCS bucket and the URL for this file would be communicated back to the TAP Service upon request of the results. 

#### HTTP Status Code Usage

The current implementation always returns HTTP 200, using a success flag in the JSON response to indicate status. This deviates from REST best practices and complicates error handling. We propose:
- Using standard HTTP status codes (400 for client errors, 500 for server errors)
- Maintaining detailed error information in the response body
- Aligning error reporting with TAP error patterns

#### Additional Enhancement Opportunities

Adding estimated completion time to status responses would be one additional enhancement that would allow us to communicate up-to-date progress information to users/clients

#### Query Validation Endpoint

A new `/validate` endpoint would provide pre-flight query validation, allowing verification of query correctness and efficiency before actual execution. This endpoint could be integrated into the TAP service's query processing pipeline to perform early validation, potentially rejecting problematic queries before they reach the database engine. This would be particularly valuable for:

- Syntax validation
- Potential performance issues
- Expected execution cost
- Query optimization suggestions
- Required resources

The implementation specifics and ultimate utility of this endpoint would need further evaluation, particularly in terms of its ability to provide meaningful validation without becoming a performance bottleneck itself. The effectiveness of the validation would depend on how well it can predict actual query behavior without full execution.

Overall, of the proposed changes VOTable support and storing results in GCS would be the more important changes in order to implement the architecture described in the technote, while 3-5 are small user-experience better or client implementation cleaner.
 

## 3. Implementation Strategy

The figure below illustrates the system architecture demonstrating the interactions between TAP and QServ during query execution. 

The key interaction paths shown include:
1. Query submission and job management between clients and the TAP service
2. Query execution and monitoring through QServ's HTTP API
3. Direct result storage from QServ to GCS
4. Result access via TAP service redirection


```{figure} system_design.png
:figclass: technote-wide-content
:scale: 50%

System Architecture
```


### 3.1 Proposed TAP query execution flow:

#### Asynchronous TAP Query

Here's the complete async TAP query flow with the new design:

##### Job Creation

  - Client creates a UWS job via TAP service with their query parameters
  - System validates parameters
  - Returns job identifier to client

##### Job Execution

  - Client requests job phase change to EXECUTING
  - When client changes job phase to EXECUTING, TAP service submits query to QServ via HTTP API and stores returned QServ query ID as the UWS `runID` parameter
  - QServ processes query across its distributed workers, updating status which TAP service can check via `/query-async/status/:queryId`
  - Upon completion, QServ writes results directly to GCS in VOTable format (XML with binary2 serialization) using predetermined bucket/path convention

##### Status Monitoring

   - Client periodically checks job status via UWS interface
   - System retrieves QServ status using stored QServ query ID (Stored as UWS runID)
   - Maps QServ progress info to UWS job parameters
   - Reports execution phase (EXECUTING, COMPLETED, ERROR, ABORTED)
  
##### Result Retrieval

   - Client requests results using UWS Job URL.
   - TAP Service checks if UWS results column is set. If not it fetches it from the QServ results endpoint, i.e. GET /query-async/result/:queryId
   - The TAP service stores the url to the result in the UWS table 
   - Client is redirected to GCS bucket with results 
   - When client requests the UWS job results, TAP service redirects them to the GCS location which follows the pattern https://gcs-bucket/results/result-{queryId}.xml
   
##### Query Cancellation

   - Client requests job phase to be changed to ABORTED
   - System uses stored QServ query ID to cancel via API
   - Updates job phase accordingly

This design removes result handling from TAP service while maintaining the standard TAP/UWS interface for clients. The key change is that instead of TAP service managing result transfer and storage, QServ writes directly to GCS using an agreed-upon location convention.

It leverages each component's strengths:

- TAP service handles the user interface and job management
- QServ handles query execution and result generation
- GCS provides reliable storage for the results
- Each component has a single & clear responsibility

This architecture should help address the goals and requirements described previously and maintain efficiency since there is no duplicate data transfer or unnecessary hops, as the results flow directly from QServ to GCS, while at the same time having an agreed-upon location convention eliminates any need for explicit coordination between QServ and TAP.

This design should also scale well with increased usage as well as data (query result) volumes since the large result files go directly to storage, there is no memory pressure on TAP service and we make efficient use of network resources.


```{figure} async_flow.png
:figclass: technote-wide-content full-width
:scale: 50%

Asynchronous TAP Query Request Flow
```


#### Synchronous TAP Query

##### Query Submission

   - Client submits query with using sync API
   - System (TAP) submits to QServ synchronous endpoint
   - Existing CADC TAP services create a UWS job for synchronous queries as well, so we maintain this design and do the same.
   - TAP Service waits for completion, and polls the QServ job status on regular intervals (HTTP GET /query-async/status/:queryId)

##### Result Streaming

   - Upon job completion, the TAP service requests the results from the QServ API
   - The results are streamed directly to client in VOTable format
   - Connection maintained until complete

```{figure} sync_flow.png
:figclass: technote-wide-content full-width
:scale: 60%

Synchronous TAP Query Request Flow
```

There is an optional alternative path where the results are written to GCS, and the TAP service upon completion of the job makes a request to the GCS bucket to fetch the results and stream them back to the user. This adds additional latency and complexity, with the main benefit being consistent behaviours between the two and persistence of the results (even though it’s most likely that these will be garbage collected eventually anyway). The nature of synchronous queries probably is better suited for the initial proposed solution where results are not written to GCS.

The key differences from async are:

- Client connection remains open
- TAP service actively monitors completion instead of client polling
- Results stream through TAP service to client rather than client being redirected to GCS
- UWS job is used for record-keeping rather than client interaction


### 3.3 TAP Implementation Changes

The current CADC TAP implementation has a few areas where there is coupling to relational databases that would need to be addressed to support the QServ HTTP API integration.
For a bit more context, our TAP services are built off of the CADC TAP library while introducing Rubin-specific customizations through our own extension layer. 
Ideally, to best support the QServ HTTP API integration and decouple the codebase from JDBC, changes would be needed in both the core CADC TAP library and our extension layer.

However, if upstream modifications cannot align with our timeline, we can implement the necessary changes within our extension layer, however this would require some complext adaptations to bridge between the JDBC-oriented base library and our QServ HTTP API.


#### Current JDBC Dependencies

The key areas of JDBC coupling in the current implementation are:

**Direct use of `java.sql.ResultSet` throughout the codebase:**
```
// Current TableWriter interface
public interface TableWriter {
    void write(ResultSet rs, OutputStream out) throws IOException;
    void write(ResultSet rs, OutputStream out, Long maxrec) throws IOException;
}
```


**JNDI DataSource configuration in QServQueryRunner:**

```
protected DataSource getQueryDataSource() throws Exception {
    Context initContext = new InitialContext();
    Context envContext = (Context) initContext.lookup("java:comp/env");
    return (DataSource) envContext.lookup(qservDataSourceName);
}
```

**Direct SQL connection management:**
```
connection.setAutoCommit(false);
pstmt = connection.prepareStatement(sql);
pstmt.setFetchSize(1000);
resultSet = pstmt.executeQuery();
```

#### Proposed Abstraction Layer

To remove these dependencies, we propose introducing a new abstraction layer:

**QueryService interface**

This provides an abstraction where either a Relational Database, or a Result store behind an API as in our case is used in the same way.

```
public interface QueryService {
    // Core query operations
    QueryJob submitQuery(String query, Map<String, String> parameters);
    QueryStatus getStatus(String queryId);
    void cancelQuery(String queryId);
    
    // Result handling
    ResultData getResults(String queryId);
}
```

**ResultData interface**

Rather than using JDBC ResultSet we could introduce a ResultData interface to the relevant sections of the code where the JDBC results are parsed.
This abstraction would aid in not having to customize too far up the class hierarchy and just use the methods with our own adaptation of the result output.

```
// Query result abstraction to replace ResultSet
public interface ResultData {
    List<ColumnInfo> getColumns();
    Stream<List<Object>> getRows();
    long getRowCount();
}

```

A slight alternative to the above could be that we instead write a result adapter which implements the ResultSet class.
```
public class QServResultSetAdapter implements ResultSet {
    private final List<Map<String, Object>> rows;
    private final List<ColumnDefinition> columns;
    private int currentRow = -1;
    
    public QServResultSetAdapter(QServQueryResponse response) {
        this.columns = parseColumns(response.schema());
        this.rows = parseRows(response.rows());
    }
    
    // Rest of implementation here..
}
```
This approach would allow us to maintain most of the existing TAP codebase by adapting QServ's JSON response format to the JDBC ResultSet interface. 
The existing TAP components could continue operating as before, with the adapter providing a bridge between the HTTP API responses and the expected JDBC interfaces.

The main benefit would be simplicity of integration, since we could introduce the QServ HTTP API with minimal changes to the TAP service code. 
The `TableWriter` implementations and result handling code would continue functioning without modification, so this provides a clear migration path with lower risk.

However, implementing a complete `ResultSet` interface requires significant effort and may introduce complexity in maintaining compatibility between QServ's native response format and JDBC's expectations. 
We would also be masking the true nature of the implementation, which could make debugging more difficult. 
Also some `ResultSet` features like transaction isolation and updatable result sets have no clear mapping to the HTTP API model.



**QServQueryService implementation**

Our implementation of the QueryService interface for QServ might look something like this:
```
public class QServQueryService implements QueryService {
    private final QServHttpService httpService;
    
    @Override
    public QueryJob submitQuery(String query, Map<String, String> parameters) {
        String qservId = httpService.submitQuery(query, createContext(parameters));
        return new QueryJob(qservId, query, parameters);
    }
    
    @Override
    public QueryStatus getStatus(String queryId) {
        return httpService.getStatus(queryId);
    }
    
    @Override
    public ResultData getResults(String queryId) {
        // For async queries, return URL to GCS storage
        String gcsUrl = httpService.getResultLocation(queryId);
        return new GCSResultData(gcsUrl);
    }
}
```

**Modified QServQueryRunner (or new implementation)**

We would then either introduce a new QueryRunner class, or modify the existing QServQueryRunner to utilize the above:
```
public class HttpQServQueryRunner implements JobRunner {
    private final QueryService queryService;  // New abstraction
    private final JobUpdater jobUpdater;
    private final ResultStore resultStore;
    
    // Remove JDBC DataSource methods
    // Remove getQueryDataSource(), getTapSchemaDataSource(), getUploadDataSource()

    private void doIt() {
        try {
            // Submit query to QServ
            QueryJob queryJob = queryService.submitQuery(
                query.getSQL(), 
                createParameters()
            );
            job.setRunID(queryJob.getId());  // Store QServ ID for tracking
        
            if (syncOutput != null) {
                // Synchronous execution
                streamResults(queryJob);
            } else {
                // Async execution - monitor until complete
                monitorQuery(queryJob);
            }
        } catch (Exception e) {
           handleError(e);
        }
    }
}
```

**Modified Result Handling**

The result handling flow needs to change to accommodate both direct streaming and GCS storage:

```
public class QServTableWriter implements TableWriter {
    private final QueryService queryService;
    
    @Override
    public void write(ResultData results, OutputStream out) throws IOException {
        if (results instanceof GCSResultData) {
            // For async queries, redirect to GCS
            String gcsUrl = ((GCSResultData) results).getUrl();
            redirectToGCS(gcsUrl);
        } else {
            // For sync queries, stream directly
            writeVOTable(results, out);
        }
    }
}
```
The async write is slightly odd here in the case where it is handled by QServ, since the TAP service is not writing out any results, so perhaps we may end up doing nothing in the async/write case.

**Cancelling Queries**

In the RSP implementation we use a `QueryJobManager` which extends `SimpleJobManager` for UWS Job management. Cancelling a query will be triggered when a UWS job is set to "ABORTED" either by a user or a system action.

In the `SimpleJobManager` this is the abort method:

```
    @Override
    public void abort(String requestPath, String jobID)
            throws JobNotFoundException, JobPersistenceException, JobPhaseException, TransientException {
        JobPersistence jobPersistence = getJobPersistence(requestPath);
        Job job = jobPersistence.get(jobID);
        doAuthorizationCheck(job);
        JobExecutor jobExecutor = getJobExecutor(requestPath, jobPersistence);
        jobExecutor.abort(job);
    }
```

In addition, the other relevant class used here is the JobPersistense class, which in our case is a `PostgresJobPersistence` which extends `DatabaseJobPersistence`. In `DatabaseJobPersistence` the delete method is the following:	

```
    public void delete(String jobID)
        throws JobPersistenceException, TransientException
    {
        JobDAO dao = getDAO();
        dao.delete(jobID);
    }
```

In our case we want to both run the UWS Job delete process which handles the delete process of the UWS job, but also the cancellation of the QServ query.
This means that we'd probably have to most likely add a new implementation of DatabaseJobPersistence and overwrite the delete method to also call the cancel method of our `QServQueryService`.

Overall, the changes will have an impact on various parts of the the RSP [lsst-tap-service](https://github.com/lsst-sqre/lsst-tap-service) repository  with the Rubin customized version of the CADC TAP software, but it also will potentially require upstream changes to decouple the [dal](https://github.com/opencadc/dal) and [uws](https://github.com/opencadc/uws) query handling code from the use of JDBC. To briefly summarize the main areas that would be impacted:

`QServQueryRunner` Changes:

- Remove JDBC connection management
- Add new QServService and utilize it here
- Replace ResultSet handling with ResultData interface or introduce new QServResultSet class which implements ResultSet
- Add QServ status monitoring for async queries
- Identify if query is targeted towards TAP_SCHEMA or data and run accordingly.

`TableWriter` Changes:

- Update interfaces to use `ResultData` instead of `ResultSet`
- Add support for GCS storage integration
- Modify VOTable generation to work with new result format

Configuration Changes:

- Add HTTP client settings
- Add QServ API endpoint configuration


### 3.4 Result Storage Architecture

The implementation of result storage presents an important architectural decision in terms of how results are communicated between TAP & QServ and here we present two potential approaches.

#### Direct Storage Approach

In this approach, QServ writes query results directly to GCS in VOTable format. Both QServ and the TAP service are configured with knowledge of a predetermined GCS bucket structure and naming conventions. QServ maintains its own GCS credentials for writing results, while the TAP service manages the user interface and result access.

When a query is executed, QServ writes results to a location determined by an agreed-upon naming convention, typically with the use of the query ID. The TAP service, using the same convention, can construct the appropriate GCS URL for redirecting users to their results without requiring explicit communication about storage locations. 

A slight alternative could be that the results endpoint of the QServ HTTP API returns a link to the bucket URL where the results can be fetched from. Which option to go for can be discussed and revisited when the implementation is mature and we have a better idea of the benefits of either.

Generally this approach offers significant advantages for handling large result sizes. By eliminating the need to stream results through the TAP service, it reduces network overhead and system resource usage. 
If we decide to go with predetermined storage conventions we even further simplify the interaction between services and reduce API complexity, though at the cost of introducing a level of coupling between QServ and TAP. Also, if QServ is able to generate VOTable output, this directly eliminates redundant format conversion steps.

Implementation considerations include establishing robust naming conventions that accommodate all query scenarios, implementing proper error handling in both services and maintaining consistent logging for operational visibility. The system must handle cases such as partial writes, failed transfers, dropped connections and cleanup of temporary results.


#### Intermediary Processing Approach

The alternative approach maintains the current model where the TAP service receives results from QServ and manages the storage process. The TAP service would retrieve results via the QServ HTTP API, convert them to VOTable format, and manage the upload to GCS storage. A slight alternative here is that the TAP Service receives the results from the QServ HTTP API in VOTable format rather than JSON so that no conversion is needed on the TAP side.

This approach provides greater control over result formatting and storage but introduces additional overhead in processing large result sets as well as serialization & deserialization overhead in the case where JSON is the only available result output of the QServ API. 


#### Average VOTable Size calculation:

To further help the discussion we make few assumptions regarding the average result size described here:

Assumed average column size: 50
The schema definitions are assumed to occupy around 5-10KB.

Characters:
Per row: 50 columns × 15 characters ≈ 750 characters
Total data: 750 characters × 10,000 rows = 7.5 million characters

A breakdown of the total size is thus:

Content: 7MB
XML Markup overhead ~= 3MB
Total estimated size: approximately 10MB

This would significantly decrease if results are serialized using base64 encoding.

It is very possible that these are overestimating the expected return size of the queries. However an overestimation here is probably better than underestimating in terms of ensuring a stable system, while at the same time it’s very difficult to calculate what the expected usage will look like. This may be revisited when we have better metrics and statistics.

For the intermediary processing approach where results flow through the TAP service, handling 10MB files would create moderate but manageable load. 
The primary concern would be concurrent queries, where if multiple users request queries simultaneously, the TAP service would need to process several 10MB streams at once. 
This could impact service performance and thus requires careful resource management.

However, the direct storage approach becomes even more appealing at this scale. Although 10MB isn't enormous by modern standards, eliminating the need to stream this data through an intermediate service reduces system load and network utilization. When we consider that some queries might return significantly larger results, and that multiple queries might run concurrently, the benefits of using direct storage makes it even more appealing.


Note that we should implement appropriate retention policies for these result files, as storing numerous 10MB files could quickly consume significant storage space. For example, if the system processes 1000 queries per day, we'd be generating around 10GB of result data daily. We should establish clear garbage collection policies for these files.



#### Recommendation

The direct storage approach appears better suited to our goals & requirements. Its key advantage is the boost in performance, through the elimination of double-handling large result sets, which reduces system resource usage and potential bottlenecks and also provides better scalability both in terms of data volume as well as concurrency. 
Using predetermined storage conventions will also reduce service-to-service communication and simplify the API, though to avoid coupling there is also the option to simply provide links to the result files via the API, both of which seem reasonable options.

We should make sure to have robust error handling and validation in both services to ensure reliable operation and on top of that having comprehensive logging and monitoring will also benefit us in the long term when it comes to visibility into the storage process.



### 3.5 Error Handling Strategy

Error handling needs to be coordinated across multiple system layers:

QServ API Level:

The QServ service will report errors through HTTP status codes and detailed error messages. For example, when a query fails due to syntax errors:

```
{
   "success": 0,
   "error": "Syntax error in query",
   "error_ext": {
       "position": 45,
       "message": "Unexpected token LIMIT"
   }
}
```

TAP Service Level:

The TAP service will map these errors to appropriate TAP/UWS error states and VOTable error documents. A QServ syntax error would be translated to:

```
<VOTABLE>
  <RESOURCE type="results">
    <INFO name="QUERY_STATUS" value="ERROR">
      Syntax error at position 45: Unexpected token LIMIT
    </INFO>
  </RESOURCE>
</VOTABLE>
```

### 3.6 Connection Pool Configuration

The HTTP client should be configured with settings appropriate for astronomical query workloads:
```
HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .pool(
        maxTotal: 200,          // Maximum concurrent connections
        maxPerRoute: 20,        // Maximum connections per endpoint
        validateAfterInactivity: Duration.ofSeconds(30),
        keepAliveTimeout: Duration.ofSeconds(60)
    )
    .build()
```

These settings should be made configurable through environment variables:
```
QSERV_HTTP_MAX_CONNECTIONS=200
QSERV_HTTP_KEEPALIVE_TIMEOUT=60
QSERV_HTTP_CONNECTION_TIMEOUT=10
```


### 3.7 Phase Mapping

The TAP service needs to map QServ query states to appropriate UWS execution phases. The mapping ensures consistent status reporting through the TAP interface while accurately reflecting the underlying query state:

| QServ State | UWS Phase   | Description |
|-------------|-------------|-------------|
| EXECUTING   | EXECUTING   | Query is actively being processed by QServ workers |
| COMPLETED   | COMPLETED   | Query has finished successfully and results are available |
| FAILED      | ERROR      | Query execution failed due to an error |
| ABORTED     | ABORTED    | Query was explicitly cancelled by user or system |

Additional state transitions that need handling:

- If QServ becomes unavailable during execution, the UWS job should transition to ERROR
- When QServ reports query exceeds result limit, job transitions to ABORTED
- If QServ fails to write results to storage, job moves to ERROR
- Initial query submission failure should leave job as PENDING

The TAP service will monitor these states and update the UWS job accordingly, allowing clients to have clear visibility into query progress and completion status.


## 4. Impact Analysis

### 4.1 Performance Implications

#### Network and Processing Impact

The transition from JDBC to HTTP introduces an additional layer of communication between the TAP service and QServ. This could potentially add some latency, but its impact could be minimized through connection pooling and keep-alive connection management. Assuming we go with the approach of QServ writing query results directly to GCS, we eliminate the need to transfer large result sets through the TAP service before moving it to the storage layer. The existing QServ HTTP API returns results in JSON format, which will add serialization/deserialization overhead if used, though this is mitigated if we go with the proposed approach where results are written directly to the result store in a VOTable format.


#### Resource Management

The new architecture provides significantly improved visibility into query execution, enabling better resource allocation and usage across the system.
Providing more detailed and accurate query progress monitoring to users, allows them to make informed decisions about query strategies and the ability to cancel long-running or inefficient queries prevents resource waste.

### 4.2 Operational Changes

#### Monitoring

The QServ HTTP API endpoints could also benefit from monitoring to ensure service health and availability. This includes tracking query progress through chunk completion metrics and maintaining visibility into the distributed query execution process. If we decide to add monitoring capabilities of the QServ API and integrate with existing TAP service monitoring systems we can have a complete view of the service health and better tracing of query execution.

####  Operational Procedures

The transition also potentially updates query management procedures and troubleshooting methods. The direct mapping of UWS runID to QServ query ID provides clear traceability and makes the process of investigating queries and resolving issues easier.


### 4.3 Risks and Mitigations

#### Service Availability
  
  Risk: HTTP API endpoint becomes unavailable
  
  Mitigation: 
  - Implement robust error handling
  - Consider fallback options (Could we keep JDBC as a backup option? Would this overly complicate the system?)
  - Monitor endpoint health

#### Performance Degradation
  
  Risk: Unexpected performance issues in production
  
  Mitigation:
  - Comprehensive performance testing before deployment
  - Ability to revert to JDBC if needed

#### Query Consistency
  
  Risk: Differences in query behavior or query results between implementations
  
  Mitigation:
  - Thorough testing of query execution patterns
  - Validation of result sets including side-by-side comparison of outputs
  - Clear documentation of any behavioral changes

#### Network Reliability

  Risk: The additional network hop through the HTTP API could introduce reliability issues and additional failure points.
  
  Mitigation: 
  - Implement reliable retry logic with exponential backoff
  - Add careful monitoring of network latency and error rates between the services

#### Resource Coordination

  Risk: Distributed nature of the system could introduce complexity regarding resource limits & cleanup of results
  
  Mitigation: 
  - Implement clear ownership of resource management
  - Explicitly develop and document automated cleanup processes & responsibilities.


## 5. System Considerations

### 5.1 TAP_SCHEMA queries

In the current setup the TAP_SCHEMA database is not stored in QServ along with the catalogue data, instead it is deployed as a sidecar MySQL container in the RSP cluster. It is also possible that this will move to a CloudSQL database in the future. In any case, if TAP_SCHEMA is not available & queryable under the same QServ API the TAP service will have to identify queries that are targeted towards TAP_SCHEMA and follow the existing JDBC route. This has the potential to introduce complexity to the system, which will require some careful consideration.

### 5.2 Garbage Collection

Assuming we go with the solution where QServ is writing the results to a bucket, the responsibility of managing the bucket and clearing the results during regular intervals will have to belong to the TAP service administrators rather than QServ.


### 5.3 Security Considerations

The proposed architecture requires careful management of GCS access permissions across services. In the direct storage approach, QServ needs write access to the results bucket. The TAP service needs read & delete access to manage result retrieval and cleanup of QServ queries, however assuming TAP_SCHEMA is elsewhere and the previous JDBC approach is used for it, we'd need to maintain read & write access for TAP as well.


### 5.4 Connection Management

Managing HTTP connections between TAP and QServ requires careful consideration of timeouts, retries, and connection pooling. The system needs to handle long-running queries without exhausting connection resources, while also being able to detect and recover from connection failures. This is particularly important for synchronous queries where client connections must be maintained throughout query execution.
In the existing JDBC-based approach, the connection pool is primarily managed by Tomcat, whereas moving to an HTTP connection moves the burden on us, which includes properly managing idle connections, timeouts, failed connections, retry logic etc. We'd need to ensure a temporary network issue does not cause the entire query to fail.


### 5.5 Query Progress Reporting

The transition to HTTP API enables better query progress monitoring, but we need to consider how to effectively expose this information to users. The UWS job parameters could be extended to include QServ specific progress information like chunk completion status, but this needs to be done in a way that maintains compatibility with standard TAP clients. One possible location for this could be the the uws:result entry, i.e. `<uws:result id="diag" xlink:href="info_here"/>`. At the same time we could utilize the `/quote` UWS endpoint to provide an estimate of execution duration, though it is unclear if there are any clients that properly utilize this.


### 5.6 Resource Limits
The system needs to handle various resource limits consistently:

- QServ query result size limits (What should the result size limit be for a TAP query? How should this be communicated to QServ? Should truncation happen on the QServ side or TAP?)
- Storage quota management in GCS (Will there be a total quota, and/or user quota? If it is exceeded what should the system behaviour be? How should temporary results be cleared and which service manages this?)
- Connection pool limits
- Query concurrency limits

These limits need to be coordinated between QServ and TAP service configurations.


### 5.7 Authentication Between Services

One more aspect that needs to be carefully evaluated is the authentication mechanism between TAP and the QServ HTTP API. Currently this is limited to network-level security, where access is restricted to specific subnets. This is slightly limiting especially in terms of the development stages, since it requires running the development version of TAP alongside the same network where QServ resides, which may in some cases not be possible.

Further authentication methods could be considered, with examples being token-based authentication which is used throughout the RSP ecosystem, or basic authentication which would be simpler to implement but would require secure credential management. We should also consider whether the TAP service is the sole client of the API as this may have an effect on which solution best meets the requirements while minimizng effort and complexity.



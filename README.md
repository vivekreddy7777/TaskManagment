Task ID,Description,Estimated Hours,Estimated Days,Required Skills,Dependencies,Assigned Developer
PBBTE-2257,Console Setup & Initialization (parent row),40,5,.NET/C#, SQL,None,Vivek
 ,Create console solution & baseline project (net8.0),4,0.5,.NET SDK,None,Vivek
 ,Bring over Program.cs and wire generic host, DI container, configuration,6,0.75,.NET Hosting/DI,None,Vivek
 ,Add appsettings.json/appsettings.Development.json; bind strongly-typed options,4,0.5,.NET Configuration,None,Vivek
 ,Migrate logging (Serilog/AppInsights); add correlationId + JobId enrichment,4,0.5,Logging/Observability,None,Vivek
 ,Set up environment variables & secrets (conn strings, DB2 creds, feature flags),4,0.5,KeyVault/Secrets Mgmt,None,Vivek
 ,Implement working folder strategy (workRoot, secureRoot, tempRoot),4,0.5,IO/Paths,None,Vivek
 ,Move “GetSystemVariableAsync” and config-server reads into Console bootstrap,4,0.5,C#, EF/ADO,. ,Vivek
 ,Create base job context (JobId, RequestorId, CompareType, paths) + DI registration,6,0.75,C#, DI,None,Vivek
 ,Smoke run: host builds, config resolves, logging flows,4,0.5,C#,. ,Vivek

PBBTE-2258,File Load Module (parent row),40,5,.NET/C#, SQL Bulk Insert,PBBTE-2257,Vivek
 ,Read input file metadata from UI (queue/DB table) and resolve file path,4,0.5,C#, DB,2257,Vivek
 ,Validate extension & schema (CSV/XLSX); map columns to canonical names,6,0.75,C#, Data Annotations,2257,Vivek
 ,Create/verify staging tables per compare type; add indexes,6,0.75,SQL DDL/Indexing,2257,Vivek
 ,Implement fast load (SqlBulkCopy/TVP); chunking for large files,8,1,SQL Bulk/TVP,2257,Vivek
 ,Row-count & checksum validation; reject on thresholds,4,0.5,SQL,2258,Vivek
 ,Error handling (bad schema, empty file, duplicate job); move to quarantine folder,4,0.5,C#/SQL,2258,Vivek
 ,Telemetry (rows/sec, bytes, duration); log to JobStep table,4,0.5,Observability,2258,Vivek
 ,Unit/dev tests with sample CSV/XLSX (small/medium/large),4,0.5,Testing,2258,Vivek

PBBTE-2259,DB2 Integration (parent row),48,6,.NET/C#, DB2/ODBC,SQL,PBBTE-2258,Vivek
 ,Configure DB2 provider/driver (ODBC/.NET), connection pooling & timeouts,6,0.75,DB2 Admin,2258,Vivek
 ,Parameterize queries; avoid OPENQUERY; add paging/batching for large reads,8,1,DB2 SQL,2258,Vivek
 ,Retry/backoff (transient), circuit breaker on repeated failures,6,0.75,Resilience Patterns,2258,Vivek
 ,Streaming reader → DataTable/TVP bridge; memory guardrails,6,0.75,C#, Performance,2258,Vivek
 ,Merge/Upsert to SQL staging (MERGE..WHEN MATCHED/NOT MATCHED),8,1,SQL MERGE,2258,Vivek
 ,Isolation level & locking review; add (NOLOCK)/snapshot where safe,4,0.5,SQL Tuning,2258,Vivek
 ,Telemetry (rows, batches, duration, retries),4,0.5,Observability,2258,Vivek
 ,Dev validation against legacy outputs (row counts, spot values),6,0.75,Testing/SQL,2258,Vivek

PBBTE-2260,Rules & Export Functionality (parent row),56,7,SQL, C#, Excel API,PBBTE-2259,Vivek
 ,Wire stored procedures for business rules/validations; capture result tables,10,1.25,SQL SPs,2259,Vivek
 ,Apply rule sequencing & fail-fast on critical defects; soft-warn on minors,6,0.75,C#/SQL,2259,Vivek
 ,Persist rule metrics (violations by type, severity) to reporting tables,6,0.75,SQL,2259,Vivek
 ,Implement Export to Excel/TSV (EPPlus/ClosedXML); sheet/tab layout,10,1.25,C# Excel,2259,Vivek
 ,Deterministic file naming (JobId_timestamp); checksum & size limits,4,0.5,IO/Conventions,2259,Vivek
 ,Store artifacts (UNC/SharePoint/S3 as applicable); set ACLs,6,0.75,Storage/Permissions,2259,Vivek
 ,Publish completion message/status back to queue/db; include paths & metrics,6,0.75,Messaging/DB,2259,Vivek
 ,End-to-end dev run; reconcile with baseline exports,8,1,Testing,2259,Vivek

PBBTE-2001,Developer Testing (parent row),32,4,C#, SQL, Debugging,PBBTE-2260,Vivek
 ,Golden path run (small/medium); verify stage→db2→rules→export,8,1,Testing,2260,Vivek
 ,Negative cases: bad schema, missing columns, corrupted file,6,0.75,Testing,2260,Vivek
 ,Resilience: DB2 down/timeout; bulk insert failures; disk full,6,0.75,Chaos/Testing,2260,Vivek
 ,Performance smoke: big file & long DB2 read; record P50/P95,6,0.75,Perf Testing,2260,Vivek
 ,Log/metric review; adjust levels & correlation ids,6,0.75,Observability,2260,Vivek

PBBTE-2261,QA Testing (parent row),80,10,QA, SQL, Test Mgmt,PBBTE-2001,Ritwik / QA
 ,Prepare test matrix (compare types, file sizes, rule coverage),8,1,Test Planning,2001,QA
 ,Functional pass: each compare type end-to-end,24,3,Manual QA,2001,QA
 ,Regression on exports: structure, formulas, filters,8,1,Excel QA,2001,QA
 ,Edge cases & error handling confirmation,16,2,QA/SQL,2001,QA
 ,Defect logging, triage, and retest loop,16,2,QA Process,2001,QA
 ,Final sign-off checklist & release notes input,8,1,Coordination,2001,QA

Date,Phase,Task
2025-10-01,Development + Dev Testing,Set up Console structure & configs
2025-10-02,Development + Dev Testing,Move initialization (system variables, folders) from Service → Console
2025-10-03,Development + Dev Testing,Refactor job hydration (GetJobDetails)
2025-10-06,Development + Dev Testing,Validate config + job hydration in Console
2025-10-07,Development + Dev Testing,Unit test init + job hydration
2025-10-08,Development + Dev Testing,Implement file load (CSV/XLSX → staging)
2025-10-09,Development + Dev Testing,Validate staging schema and row counts
2025-10-10,Development + Dev Testing,Dev test with sample input files
2025-10-13,Development + Dev Testing,Add error handling for file load (bad/missing columns)
2025-10-14,Development + Dev Testing,Regression test file load module
2025-10-15,Development + Dev Testing,Move DB2 fetch logic into Console
2025-10-16,Development + Dev Testing,Implement DB2 merge into staging
2025-10-17,Development + Dev Testing,Dev test DB2 query execution
2025-10-20,Development + Dev Testing,Validate DB2 merge row counts
2025-10-21,Development + Dev Testing,Add retry handling for DB2 failures
2025-10-22,Development + Dev Testing,Move SQL rules/validations into Console
2025-10-23,Development + Dev Testing,Implement export logic (Excel/TSV naming conventions)
2025-10-24,Development + Dev Testing,Run end-to-end test: staging → DB2 → rules → export
2025-10-27,Development + Dev Testing,Debug issues and refine logging
2025-10-28,Development + Dev Testing,Finalize Dev + Dev Testing; prepare QA handover
2025-10-29,QA Testing + Bug Fixes,QA test with small input files
2025-10-30,QA Testing + Bug Fixes,QA regression check export formatting
2025-10-31,QA Testing + Bug Fixes,QA validate DB2 merges with expected counts
2025-11-03,QA Testing + Bug Fixes,QA test large input files (performance checks)
2025-11-04,QA Testing + Bug Fixes,QA edge case testing (invalid columns, missing metadata)
2025-11-05,QA Testing + Bug Fixes,Triage QA defects and log bug list
2025-11-06,QA Testing + Bug Fixes,Fix defects (file load + DB2 queries)
2025-11-07,QA Testing + Bug Fixes,Retest fixed defects with targeted scenarios
2025-11-10,QA Testing + Bug Fixes,Validate logs, metrics, dashboards
2025-11-11,QA Testing + Bug Fixes,Final QA sign-off and project closure

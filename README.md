Task ID	Description	Estimated Hours	Estimated Days	Required Skills	Dependencies
Section 1	Development Steps					
DSN-001	DSN Service - .NET 8 Worker Setup & Initialization	6	0.75	.NET 8, Worker Service	None
DSN-001.1	Create .NET 8 Worker Service project + Windows Service hosting	2	0.25	.NET 8	None
DSN-001.2	Add appsettings.json + strongly-typed options + validation	2	0.25	.NET Configuration	DSN-001.1
DSN-001.3	Add logging (EventLog/Console) + startup smoke run	2	0.25	Logging/Observability	DSN-001.1
DSN-002	DSN Service - SQL Access Layer (Replace SqlUtilities)	8	1	SQL, ADO.NET/.NET	DSN-001
DSN-002.1	Implement Proc execution wrapper (DataTable/Scalar/NonQuery)	4	0.5	SQL, ADO.NET	DSN-001
DSN-002.2	Add parameter mapping + null handling + timeout handling	2	0.25	SQL	DSN-002.1
DSN-002.3	Add safe error logging + SP call tracing	2	0.25	Logging/SQL	DSN-002.1
DSN-003	DSN Service - Core DSN Job Orchestration	10	1.25	C#, Workflow, SQL	DSN-002
DSN-003.1	Load main request + build job context (RACF/LAN/Bundle/IDs)	3	0.38	C#, SQL	DSN-002
DSN-003.2	Implement Audit ID generation + WorkTranx update flow	3	0.38	SQL, C#	DSN-003.1
DSN-003.3	Implement claim count logic + split/trigger DSN parts	2	0.25	SQL	DSN-003.1
DSN-003.4	Implement “no claims → reprocess as CCA-only” workflow	2	0.25	SQL, Workflow	DSN-003.1
DSN-004	DSN Service - File Generation (Legacy Compatible)	6	0.75	File IO, C#	DSN-003
DSN-004.1	Filename builder (legacy format) + time/alpha conversions	2	0.25	C# Strings/DateTime	DSN-003
DSN-004.2	Control record builder (PadRight, DOS date, env/server fields)	2	0.25	C# Strings, SQL	DSN-004.1
DSN-004.3	Write DSN file content rules (POS/ELG delimiter, uppercase, truncate)	2	0.25	File IO	DSN-004.2
DSN-005	DSN Service - SQL Logging + Cleanup	0	0	SQL	DSN-004
DSN-005.1	Log results to SQL (procBTE_DSN_Data_INSERT) + record counts	0	0	SQL	DSN-004
DSN-005.2	Cleanup on failure (delete generated DSN files)	0	0	File IO	DSN-004
CONVERSION TOTAL		30	3.75		

Section 2	Dev Testing					
DSN-T001	Dev Testing - Local + QA Validation (Developer Owned)	10	1.25	Dev Testing, SQL, File IO	DSN-004
DSN-T001.1	Run happy-path DSN job and validate file output formatting	3	0.38	Dev Testing	DSN-004
DSN-T001.2	Validate POS/ELG vs tab-delimited DSN types output	2	0.25	Dev Testing	DSN-T001.1
DSN-T001.3	Validate SQL logging (record count, file name, upload location)	2	0.25	SQL Validation	DSN-T001.1
DSN-T001.4	Test “no claims → CCA-only reprocess” workflow end-to-end	2	0.25	Workflow/SQL	DSN-T001.1
DSN-T001.5	Test failure cleanup (file delete on exception)	1	0.13	Dev Testing	DSN-T001.1
DEV TESTING TOTAL		10	1.25		

GRAND TOTAL		40	5		

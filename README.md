Overview

Right now, the CompareTool service is doing both listening and execution. The service listens for jobs, loads job details, reads system variables, processes input files, calls stored procedures, handles DB2 fetch, applies rules, and finally exports results. This design puts too much inside the service. It limits scaling because the service has to do everything end to end.

The new design moves all heavy execution into the Console. The Service will only listen for jobs and enqueue them. The Console will take the job and run the full pipeline. This separation gives us scaling flexibility, better error handling, and makes the system easier to monitor.

⸻

High Level Flow
	1.	UI/API Upload – User uploads input file, job entry is created in SQL.
	2.	Service Listener – Picks up the request, enqueues into WorkQueue, marks job as pending.
	3.	Console Worker – Console picks up job from queue and runs the full execution:
	•	Initialize (config, variables, working folder)
	•	Load job details from SQL
	•	Load input file into staging
	•	Call DB2 and merge results
	•	Run SQL rules and validations
	•	Export results (Excel/TSV)
	•	Mark job completed or failed
 UI/API Upload
       │
       ▼
 ┌─────────────┐
 │   Service   │  (Enqueue only)
 └─────────────┘
       │
       ▼
 ┌──────────────────────────┐
 │         Console          │
 │  - Initialize Config     │
 │  - Get Job Details       │
 │  - Load Input File       │
 │  - DB2 Fetch & Merge     │
 │  - Run Rules/Validations │
 │  - Export Results        │
 │  - Mark Completion       │
 └──────────────────────────┘
Data
	•	JobHeader: Keeps metadata like JobId, FileName, CompareType, Status.
	•	WorkQueue: Tracks job execution (pending, running, completed, failed).
	•	Staging Tables: Temporary area for input file data.
	•	Result Tables: Store processed data and QC results.
	•	Logs: Step by step execution details, errors, row counts.

⸻

Error Handling
	•	Transient errors (e.g., DB2 timeout) → retry logic.
	•	Permanent errors (e.g., bad file format) → fail fast, mark job as error.
	•	Poison jobs (too many retries) → stopped and flagged.
	•	Logs capture job id, step, duration, and error details.

⸻

Scaling
	•	Service stays thin. Only needs 1–2 replicas.
	•	Console can be scaled out. Each worker handles one job at a time. Add more workers = more throughput.
	•	Database (SQL Server) remains central for staging, results, and rules.

Testing
	•	Testing is manual.
	•	Jobs will be run with sample input files covering different compare types.
	•	Output Excel/TSV will be validated against expected results.
	•	Manual checks include row counts, DB2 merge correctness, rule validations, and export formatting.
	•	Error scenarios (invalid file, missing columns, DB2 failure) will also be manually tested.

Conclusion

By moving all job execution into the Console, we decouple listening from processing. The Service stays light, and the Console becomes the execution engine. This lets us scale horizontally without changing business logic, and makes CompareTool more reliable under higher loads.
 

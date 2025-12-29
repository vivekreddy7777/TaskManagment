DSN Creation – Use Cases

Overview

The DSN creation process is responsible for preparing, splitting (if required), and generating DSN files based on eligible claim data.
The process is executed via C# services and relies on a combination of database procedures, linked server calls, and configuration-driven rules.

The following use cases describe the DSN lifecycle from request intake through file generation and validation.

⸻

Use Case 1 – Retrieve DSN Request Details

Description

Retrieves DSN-related request information and prepares the request context used by downstream steps.

Inputs
	•	Request number
	•	Environment (Production / UAT / Development)

Process
	•	Fetch request details from the Main_Request table
	•	Retrieve DSN-specific details using the DSN request lookup procedure
	•	Lookup RACF ID from QCore (if applicable)
	•	Resolve DSN bundle associated with the request
	•	Generate AuditId if not already present
	•	Store resolved values for subsequent processing

Outputs
	•	Request context populated with:
	•	Request number
	•	DSN bundle
	•	AuditId
	•	RACF ID
	•	Environment

Failure Scenario
	•	Request not found or invalid → request marked as FAILED
	•	AuditId generation fails → processing stops and error is logged

⸻

Use Case 2 – Select Mainframe User Account

Description

Determines the appropriate mainframe user account to use for DSN processing.

Inputs
	•	Request context
	•	System request indicator

Process
	•	If the request is system-driven, select a mainframe account from the configured account pool
	•	Account selection is performed using round-robin logic
	•	Selected account is associated with the request

Outputs
	•	Request updated with selected mainframe user account

Failure Scenario
	•	No available accounts → request marked as FAILED

⸻

Use Case 3 – Retrieve Previous Module Information

Description

Retrieves outputs from prior processing modules required for DSN creation.

Inputs
	•	Request number

Process
	•	Lookup the prior module record (Claim Creation)
	•	Retrieve WorkManager_ID associated with the request
	•	Attach WorkManager_ID to the request context

Outputs
	•	Request context updated with WorkManager_ID

Failure Scenario
	•	Prior module record missing → request marked as FAILED

⸻

Use Case 4 – Calculate Eligible Claims

Description

Identifies eligible (“good”) claims that should be included in the DSN.

Inputs
	•	Request context
	•	Claim data source

Process
	•	Apply non-fallout rules to identify valid claims
	•	Calculate total eligible claim count
	•	Determine whether DSN splitting is required based on claim limit

Outputs
	•	Total eligible claim count
	•	Indicator for split requirement

Failure Scenario
	•	Claim query fails → request marked as FAILED

⸻

Use Case 5 – Cancel DSN for No Eligible Claims

Description

Handles scenarios where no eligible claims are found.

Inputs
	•	Eligible claim count = 0

Process
	•	Mark DSN request as cancelled due to no claims
	•	Reprocess the request as CCA-Only
	•	Log the cancellation reason

Outputs
	•	DSN request cancelled
	•	Reprocessing triggered

Failure Scenario
	•	Status update fails → error logged and request flagged

⸻

Use Case 6 – Split DSN into Multiple Parts

Description

Splits DSN processing into multiple parts when claim volume exceeds the configured threshold.

Inputs
	•	Total eligible claim count
	•	Claim limit threshold

Process
	•	Calculate number of required DSN parts
	•	Create additional DSN requests for each part
	•	Create framework request for each DSN part
	•	Update DSN requests with framework IDs
	•	Record total number of parts

Outputs
	•	Multiple DSN requests created
	•	Framework IDs linked to each part

Failure Scenario
	•	Partial split failure → request marked as FAILED

⸻

Use Case 7 – Load Claims into DSN Tables

Description

Loads claim data into DSN working tables used for file generation.

Inputs
	•	Request context
	•	Eligible claim dataset

Process
	•	Execute DSN load stored procedure
	•	Populate DSN tables using linked server calls
	•	Capture row counts for auditing

Outputs
	•	DSN tables populated with claim data

Failure Scenario
	•	Stored procedure or linked server failure → request marked as FAILED

⸻

Use Case 8 – Create DSN Control Records

Description

Creates control records required for each DSN file.

Inputs
	•	Request context
	•	DSN file type configuration

Process
	•	Retrieve DSN file definitions from configuration
	•	Generate control record for each DSN file using standard format
	•	Insert control records into DSN data tables

Outputs
	•	Control records created for all required DSN files

Failure Scenario
	•	Missing configuration → request marked as FAILED

⸻

Use Case 9 – Generate DSN Files

Description

Generates DSN output files based on populated DSN tables.

Inputs
	•	DSN tables
	•	DSN file definitions

Process
	•	Extract records per DSN file type
	•	Format output according to file specification
	•	Write files to output location
	•	Record file metadata in DSN data table

Outputs
	•	DSN files generated
	•	File metadata stored in database

Failure Scenario
	•	File generation or IO error → request marked as FAILED

⸻

Use Case 10 – Validate Generated Files

Description

Validates that DSN files contain expected record counts.

Inputs
	•	Generated files
	•	Expected record counts

Process
	•	Compare expected vs actual record counts
	•	Verify required files are not empty
	•	Ensure final DSN file contains valid records

Outputs
	•	Validation status recorded

Failure Scenario
	•	Validation failure → request cancelled or marked FAILED

⸻

Use Case 11 – Complete DSN Processing

Description

Finalizes DSN processing and updates request status.

Inputs
	•	Successful completion of all prior steps

Process
	•	Update DSN request status to COMPLETED
	•	Record completion timestamp
	•	Log final processing summary

Outputs
	•	DSN request completed

Failure Scenario
	•	Completion update fails → request flagged for manual review

⸻

Use Case 12 – Error Handling and Retry

Description

Handles processing errors and retry scenarios consistently.

Inputs
	•	Error details
	•	Retry configuration

Process
	•	Log error with step context
	•	Update request status to FAILED
	•	Retry eligible failures based on retry policy
	•	Prevent duplicate processing during retry

Outputs
	•	Error logged
	•	Retry scheduled if applicable

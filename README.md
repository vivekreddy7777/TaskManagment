Story 1: Run DSN Service as a Worker

As the developer,
I want a .NET 8 worker service that runs continuously and polls for DSN requests,
so DSN jobs are processed automatically without manual execution.

Acceptance
	•	Service starts and stops cleanly
	•	Poll interval is configurable
	•	Logs when a DSN job begins and ends

⸻

Story 2: Generate DSN Files

As the developer,
I want the service to generate DSN files using the existing legacy rules,
so downstream systems receive files in the expected format.

Acceptance
	•	Filenames match legacy format
	•	Control record is written as the first line
	•	POS/ELG files are fixed-width (no delimiter)
	•	Other DSN files are tab-delimited
	•	Data is upper-case and truncated to record length

⸻

Story 3: Log DSN Results and Handle No-Claims

As the developer,
I want the service to log DSN results to SQL and handle no-claim scenarios correctly,
so DSN processing is auditable and workflows remain accurate.

Acceptance
	•	DSN file details are inserted into SQL
	•	Record counts are logged correctly
	•	When no claims exist, DSN is canceled and reprocessed as CCA-only
	•	No partial or orphaned files remain

SELECT *
FROM Quality_Control.MCR_GridTracker
WHERE sr IS NULL
  AND planYear >= 2024
  AND GridType <> 'Draft'
  AND (
        -- Scenario 1: Failed Validation + ErrorMsg like '%An error%'
        (CopayGridStatus = 'Failed Validation'
         AND ErrorMsg LIKE '%An error%')

        OR

        -- Scenario 2: Not Failed Validation + Processed = 1 + SSLoaded = 0
        (CopayGridStatus <> 'Failed Validation'
         AND Processed = 1
         AND SSLoaded = 0)

        OR

        -- Scenario 3: Processed = 1 + ProcessedOn IS NULL
        (Processed = 1
         AND ProcessedOn IS NULL)

        OR

        -- Scenario 4: SSLoaded = 0 and not draft
        (SSLoaded = 0)
      );

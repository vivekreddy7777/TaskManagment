CREATE PROCEDURE Quality_Control.proc_MCR_GridTracker_CheckAndNotify
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @body     NVARCHAR(MAX) = N'';
    DECLARE @subject  NVARCHAR(300);
    DECLARE @Case1 BIT = 0, 
            @Case2 BIT = 0, 
            @Case3 BIT = 0, 
            @Case4 BIT = 0;

    ----------------------------------------------------------------
    -- 1. Pull all rows that match ANY of the 4 conditions
    ----------------------------------------------------------------
    ;WITH BadRows AS
    (
        SELECT 
            m.sr,
            m.PlanYear,
            m.GridType,
            m.CopayGridStatus,
            m.ErrorMsg,
            m.Processed,
            m.SSLoaded,
            m.ProcessedOn,

            -- Case 1: Failed Validation + ErrorMsg like '%An error%'
            Case1 = CASE WHEN (m.CopayGridStatus = 'Failed Validation'
                               AND m.ErrorMsg LIKE '%An error%')
                         THEN 1 ELSE 0 END,

            -- Case 2: Not Failed Validation + Processed = 1 + SSLoaded = 0
            Case2 = CASE WHEN (m.CopayGridStatus <> 'Failed Validation'
                               AND m.Processed = 1
                               AND m.SSLoaded = 0)
                         THEN 1 ELSE 0 END,

            -- Case 3: Processed = 1 + ProcessedOn IS NULL
            Case3 = CASE WHEN (m.Processed = 1
                               AND m.ProcessedOn IS NULL)
                         THEN 1 ELSE 0 END,

            -- Case 4: SSLoaded = 0
            Case4 = CASE WHEN (m.SSLoaded = 0)
                         THEN 1 ELSE 0 END
        FROM Quality_Control.MCR_GridTracker m
        WHERE 
              m.sr IS NULL
          AND m.PlanYear >= 2024
          AND m.GridType <> 'Draft'
          AND (
                (m.CopayGridStatus = 'Failed Validation'
                 AND m.ErrorMsg LIKE '%An error%')
             OR (m.CopayGridStatus <> 'Failed Validation'
                 AND m.Processed = 1
                 AND m.SSLoaded = 0)
             OR (m.Processed = 1
                 AND m.ProcessedOn IS NULL)
             OR (m.SSLoaded = 0)
          )
    )
    SELECT *
    INTO #BadRows
    FROM BadRows;

    -- If nothing matched, exit quietly
    IF NOT EXISTS (SELECT 1 FROM #BadRows)
        RETURN;

    ----------------------------------------------------------------
    -- 2. Figure out which cases are present overall
    ----------------------------------------------------------------
    SELECT 
        @Case1 = MAX(Case1),
        @Case2 = MAX(Case2),
        @Case3 = MAX(Case3),
        @Case4 = MAX(Case4)
    FROM #BadRows;

    ----------------------------------------------------------------
    -- 3. Build dynamic subject: which cases failed
    ----------------------------------------------------------------
    SET @subject = N'MCR GridTracker – Failed Cases: ';

    IF @Case1 = 1 SET @subject += N'Case1 (Failed Validation), ';
    IF @Case2 = 1 SET @subject += N'Case2 (Processed=1 & SSLoaded=0), ';
    IF @Case3 = 1 SET @subject += N'Case3 (ProcessedOn NULL), ';
    IF @Case4 = 1 SET @subject += N'Case4 (SSLoaded=0), ';

    -- Trim trailing comma + space
    IF RIGHT(@subject, 2) = ', '
        SET @subject = LEFT(@subject, LEN(@subject) - 2);

    ----------------------------------------------------------------
    -- 4. Build body text
    ----------------------------------------------------------------
    SET @body = 
        N'The following MCR_GridTracker rows match one or more status rules.' + CHAR(13) + CHAR(10) +
        N'Cases:' + CHAR(13) + CHAR(10) +
        CASE WHEN @Case1 = 1 THEN N'  - Case1: CopayGridStatus = ''Failed Validation'' AND ErrorMsg LIKE ''%An error%''' + CHAR(13) + CHAR(10) ELSE N'' END +
        CASE WHEN @Case2 = 1 THEN N'  - Case2: CopayGridStatus <> ''Failed Validation'' AND Processed = 1 AND SSLoaded = 0' + CHAR(13) + CHAR(10) ELSE N'' END +
        CASE WHEN @Case3 = 1 THEN N'  - Case3: Processed = 1 AND ProcessedOn IS NULL' + CHAR(13) + CHAR(10) ELSE N'' END +
        CASE WHEN @Case4 = 1 THEN N'  - Case4: SSLoaded = 0' + CHAR(13) + CHAR(10) ELSE N'' END +
        CHAR(13) + CHAR(10) +
        N'Rows:' + CHAR(13) + CHAR(10) +
        N'SR | PlanYear | GridType | CopayGridStatus | Processed | SSLoaded | ProcessedOn | ErrorMsg' + CHAR(13) + CHAR(10) +
        REPLICATE('-', 120) + CHAR(13) + CHAR(10);

    SELECT 
        @body = @body +
            ISNULL(CAST(sr AS VARCHAR(20)), 'NULL') + ' | ' +
            ISNULL(CAST(PlanYear AS VARCHAR(10)), 'NULL') + ' | ' +
            ISNULL(GridType, 'NULL') + ' | ' +
            ISNULL(CopayGridStatus, 'NULL') + ' | ' +
            ISNULL(CAST(Processed AS VARCHAR(1)), 'NULL') + ' | ' +
            ISNULL(CAST(SSLoaded AS VARCHAR(1)), 'NULL') + ' | ' +
            ISNULL(CONVERT(VARCHAR(19), ProcessedOn, 120), 'NULL') + ' | ' +
            ISNULL(ErrorMsg, 'NULL') +
            CHAR(13) + CHAR(10)
    FROM #BadRows;

    ----------------------------------------------------------------
    -- 5. Call your existing email SP (same pattern as C#)
    ----------------------------------------------------------------
    EXEC dbo.procBTE_System_Emails_INSERT
         @Debug            = 0,
         @ProcResult       = 0,
         @CallingProcedure = N'proc_MCR_GridTracker_CheckAndNotify',
         @MailTo           = N'Q-Core@Express-Scripts.com',  -- change if needed
         @MailCC           = N'',
         @MailSubject      = @subject,
         @MailBody         = @body,
         @Procedure        = N'CompareTool';                 -- or 'MedicareSync', etc.
END;
GO

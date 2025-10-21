IF @ReportType IN ('DAW PENALTY')
BEGIN
    SET @SQLSelect =
    'SELECT DISTINCT
         OneABL.BPLID                                            AS [BPL_ID],
         OneABL.CarrierNumber                                    AS [Carrier],
         OneABL.FormularyID                                      AS [Formulary ID],
         dbo.fnTranslateValue(''' + @PROGRAM_ID + ''',
                              OneABL.FormularyID, ''FormularyID'', '''') AS [Formulary_Desc],
         CONVERT(varchar(10), TRY_CONVERT(date, OneABL.BPLEffectiveDateOfChange), 120)
                                                                 AS [BPL Effective Date Of Change],
         CASE WHEN UPPER(LTRIM(RTRIM(OneABL.DAW1Penalty))) IN (''YES'',''Y'',''1'')
              THEN ''Yes'' ELSE ''No'' END                       AS [DAW1 PENALTY],
         CASE WHEN UPPER(LTRIM(RTRIM(OneABL.DAW2Penalty))) IN (''YES'',''Y'',''1'')
              THEN ''Yes'' ELSE ''No'' END                       AS [DAW2 PENALTY]';

    SET @SQLFrom =
    ' FROM dbo.BTE_CI_IntentDoc_OneABL_BPLHeader AS OneABL';

    DECLARE @SQL nvarchar(max) =
        @SQLSelect + @SQLFrom + ISNULL(@SQLWhere, '') + ISNULL(@SQLOrder, '');

    -- PRINT @SQL;  -- uncomment to debug the final SQL
    EXEC(@SQL);
END

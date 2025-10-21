IF @ReportType IN ('DAW PENALTY')
BEGIN
    SET @SQLSelect = 
    'SELECT DISTINCT
         LEFT(Details.Carrier,3) + Details.Entity                  AS [BPL_ID],
         Details.Carrier                                           AS [Carrier],
         Details.XML_FORMULARYID                                   AS [Formulary ID],
         dbo.fnTranslateValue(''' + @PROGRAM_ID + ''',
                              BPLHeader.FormularyID, ''FormularyID'', '''') AS [Formulary_Desc],
         FORMAT(Details.CHANGE_EFFECTIVE_DT, ''yyyy-MM-dd'')       AS [BPL Effective Date Of Change],
         CASE WHEN UPPER(LTRIM(RTRIM(OneABL.DAW1Penalty))) IN (''YES'',''Y'',''1'') 
              THEN ''Yes'' ELSE ''No'' END AS [DAW1 PENALTY],
         CASE WHEN UPPER(LTRIM(RTRIM(OneABL.DAW2Penalty))) IN (''YES'',''Y'',''1'') 
              THEN ''Yes'' ELSE ''No'' END AS [DAW2 PENALTY]';

    SET @SQLFrom =
    ' FROM   @Intake      AS Intake
       INNER JOIN @Detail*    AS Details   ON Intake.Intake_ID     = Details.Intake_ID
       INNER JOIN @BtaSupp    AS BtaSupp   ON Details.Intake_ID    = BtaSupp.Intake_ID
       INNER JOIN @Header     AS Header    ON BtaSupp.Document_ID  = Header.Document_ID
       INNER JOIN @BPLHeader  AS BPLHeader ON Header.Document_ID   = BPLHeader.Document_ID
       LEFT  JOIN BTE_CI_IntentDoc_OneABL_BPLHeader AS OneABL
              ON OneABL.Document_ID = BPLHeader.Document_ID';

    DECLARE @SQL nvarchar(max) = @SQLSelect + @SQLFrom
                                + ISNULL(@SQLWhere, '') 
                                + ISNULL(@SQLOrder, '');
    EXEC(@SQL);
END

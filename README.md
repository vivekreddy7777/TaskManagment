// grabs CCRCDX ranges into #CCRCDX exactly like the SP (via OPENQUERY)
private void PullCrcdxRanges()
{
    const string create = @"
IF OBJECT_ID('tempdb..#CCRCDX') IS NOT NULL DROP TABLE #CCRCDX;
CREATE TABLE #CCRCDX
(
    DCX_RVRSD_PTNT_AGN_START VARCHAR(11),
    DCX_RVRSD_PTNT_AGN_END   VARCHAR(10),
    TABLE_ID                 VARCHAR(3)
);";
    using (var cmd = new SqlCommand(create, _sql)) cmd.ExecuteNonQuery();

    const string linkedServer = /* put your linked server name here */ "DB2_LINK";

    string insert = $@"
INSERT INTO #CCRCDX (DCX_RVRSD_PTNT_AGN_START, DCX_RVRSD_PTNT_AGN_END, TABLE_ID)
SELECT DCX_RVRSD_PTNT_AGN_START,
       DCX_RVRSD_PTNT_AGN_END,
       LEFT(DCX_TABLE_ID, 1)
FROM OPENQUERY({linkedServer},
    'SELECT DCX_RVRSD_PTNT_AGN_START, DCX_RVRSD_PTNT_AGN_END, DCX_TABLE_ID
       FROM DB0.@RL+CCRO.CCRCDX00
      FOR FETCH ONLY WITH UR')";
    using (var cmd = new SqlCommand(insert, _sql)) cmd.ExecuteNonQuery();
}

// grabs POSIDX ranges into #POSIDX exactly like the SP (via OPENQUERY)
private void PullPosnixRanges()
{
    const string create = @"
IF OBJECT_ID('tempdb..#POSIDX') IS NOT NULL DROP TABLE #POSIDX;
CREATE TABLE #POSIDX
(
    INX_DSP_PHCY_START VARCHAR(10),
    INX_DSP_PHCY_END   VARCHAR(10),
    TABLE_ID           VARCHAR(3),
    PROCESSED          BIT DEFAULT 0
);";
    using (var cmd = new SqlCommand(create, _sql)) cmd.ExecuteNonQuery();

    const string linkedServer = /* put your linked server name here */ "DB2_LINK";

    string insert = $@"
INSERT INTO #POSIDX (INX_DSP_PHCY_START, INX_DSP_PHCY_END, TABLE_ID)
SELECT LEFT(REPLACE(INX_DSP_PHCY_START, CHAR(0), ''), 10),
       LEFT(INX_DSP_PHCY_END, 10),
       LEFT(INX_TABLE_ID, 1)
FROM OPENQUERY({linkedServer},
    'SELECT INX_DSP_PHCY_START, INX_DSP_PHCY_END, INX_TABLE_ID
       FROM DB0.@RL+POS0.POSIDX00
      FOR FETCH ONLY WITH UR')";
    using (var cmd = new SqlCommand(insert, _sql)) cmd.ExecuteNonQuery();
}

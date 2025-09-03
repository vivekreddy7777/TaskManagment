private void MapDb2IdsIntoAddtnl1()
{
    var sqlText = $@"
UPDATE t
SET t.RVRSD_PNT_AGN = d.RVSVD_PINT_ACN
FROM [{Db}].[tblCompareClaimsDetails_Addtnl_1_IDs] AS t
JOIN #DB2_MEMBER_INFO d
  ON d.MEMBER_ID  = t.MEMBER_ID
 AND d.PERSON_NBR = t.PERSON_NBR
WHERE t.[BulkInsertKey] = @key
  AND t.[WM_ID]        = @wm;";

    using var cmd = new SqlCommand(sqlText, _sql);
    cmd.Parameters.AddWithValue("@key", _bulkInsertKey);
    cmd.Parameters.AddWithValue("@wm",  _wmId);
    cmd.ExecuteNonQuery();
}

--------------------------------------------
private void InsertEntityRowsIntoLocalIds()
{
    var sqlText = $@"
INSERT INTO [{Db}].[tblCompareClaimsDetails_Addtnl_1_IDs]
        ([WM_ID], [BulkInsertKey], [IM_CLAIM_ID], [RVRSD_PNT_AGN], [MEMBER_ID])
SELECT  @wm,   @key,            [CLAIM_ID],     [RVRSD_PNT_AGN],  [MEMBER_ID]
FROM   [{_tempTableName}] AS et;";

    using var cmd = new SqlCommand(sqlText, _sql);
    cmd.Parameters.AddWithValue("@key", _bulkInsertKey);
    cmd.Parameters.AddWithValue("@wm",  _wmId);
    cmd.ExecuteNonQuery();
}
--------------------------------------------
private void LoadPosnixRanges()
{
    // a) local temp
    const string create = @"
IF OBJECT_ID('tempdb..#POSNIX') IS NOT NULL DROP TABLE #POSNIX;
CREATE TABLE #POSNIX(
    INX_DSP_PHCY_START varchar(10),
    INX_DSP_PHCY_END   varchar(10),
    TABLE_ID           varchar(1),
    Processed          bit DEFAULT 0
);";
    using (var cmd = new SqlCommand(create, _sql)) cmd.ExecuteNonQuery();

    // b) pull from DB2  (swap to your real DB2 table/view name)
    const string db2Sql = @"
SELECT
    RTRIM(REPLACE(INX_DSP_PHCY_START, CHAR(0), '')) AS INX_DSP_PHCY_START,
    RTRIM(REPLACE(INX_DSP_PHCY_END,   CHAR(0), '')) AS INX_DSP_PHCY_END,
    LEFT(INX_TABLE_ID, 1)                            AS TABLE_ID
FROM POS0.POSIDXDB
WITH UR";

    var rows = new DataTable();
    using (var db2Cmd = _db2.CreateCommand())
    {
        db2Cmd.CommandText = db2Sql;
        using var da = new DB2DataAdapter((DB2Command)db2Cmd);
        da.Fill(rows);
    }

    // c) bulk to #POSNIX
    using var bulk = new SqlBulkCopy(_sql) { DestinationTableName = "#POSNIX" };
    bulk.ColumnMappings.Add("INX_DSP_PHCY_START", "INX_DSP_PHCY_START");
    bulk.ColumnMappings.Add("INX_DSP_PHCY_END",   "INX_DSP_PHCY_END");
    bulk.ColumnMappings.Add("TABLE_ID",           "TABLE_ID");
    bulk.WriteToServer(rows);
}
------------------------------------------
private void LoadCcrdxRanges()
{
    // a) local temp
    const string create = @"
IF OBJECT_ID('tempdb..#CCRDX') IS NOT NULL DROP TABLE #CCRDX;
CREATE TABLE #CCRDX(
    DCX_RVRSD_PNT_AGN_START varchar(11),
    DCX_RVRSD_PNT_AGN_END   varchar(10),
    TABLE_ID                varchar(1)
);";
    using (var cmd = new SqlCommand(create, _sql)) cmd.ExecuteNonQuery();

    // b) pull from DB2 (swap to your real CCR0/CCRDCX00 path)
    const string db2Sql = @"
SELECT
    DCX_RVRSD_PNT_AGN_START,
    DCX_RVRSD_PNT_AGN_END,
    LEFT(DCX_TABLE_ID, 1) AS TABLE_ID
FROM CCR0.CCRDCX00
WITH UR";

    var rows = new DataTable();
    using (var db2Cmd = _db2.CreateCommand())
    {
        db2Cmd.CommandText = db2Sql;
        using var da = new DB2DataAdapter((DB2Command)db2Cmd);
        da.Fill(rows);
    }

    // c) bulk to #CCRDX
    using var bulk = new SqlBulkCopy(_sql) { DestinationTableName = "#CCRDX" };
    bulk.ColumnMappings.Add("DCX_RVRSD_PNT_AGN_START", "DCX_RVRSD_PNT_AGN_START");
    bulk.ColumnMappings.Add("DCX_RVRSD_PNT_AGN_END",   "DCX_RVRSD_PNT_AGN_END");
    bulk.ColumnMappings.Add("TABLE_ID",                "TABLE_ID");
    bulk.WriteToServer(rows);
}
-------------------------------------------
private void UpdateAddtnl1FromEntityAndRanges()
{
    // (a) copy over simple fields from entity temp to Addtnl_1_IDs
    var updateA = $@"
UPDATE ID
SET  ID.CompareCode   = ISNULL(ET.COMPARE_CODE, 'MARN')
    ,ID.CLAIM_DOS     = ET.CLAIM_DOS
    ,ID.PHARMACY_ID   = ET.PHARMACY_ID
    ,ID.ClaimResponse = ET.ClaimResponse
FROM [{Db}].[tblCompareClaimsDetails_Addtnl_1_IDs] ID
JOIN [{_tempTableName}] ET
  ON ID.IM_CLAIM_ID = ET.CLAIM_ID
WHERE ID.[BulkInsertKey] = @key
  AND ID.[WM_ID]        = @wm;";

    using (var cmd = new SqlCommand(updateA, _sql))
    {
        cmd.Parameters.AddWithValue("@key", _bulkInsertKey);
        cmd.Parameters.AddWithValue("@wm",  _wmId);
        cmd.ExecuteNonQuery();
    }

    // (b) set POS_DB and CCR_DB from the temp range tables
    var updateB = $@"
UPDATE T
SET POS_DB =
(
    SELECT TOP 1 P.TABLE_ID
    FROM #POSNIX P
    WHERE TRY_CAST(T.PHARMACY_ID AS bigint) BETWEEN
          TRY_CAST(P.INX_DSP_PHCY_START AS bigint)
      AND TRY_CAST(P.INX_DSP_PHCY_END   AS bigint)
),
CCR_DB =
(
    SELECT TOP 1 C.TABLE_ID
    FROM #CCRDX C
    WHERE TRY_CAST(T.RVRSD_PNT_AGN AS bigint) BETWEEN
          TRY_CAST(C.DCX_RVRSD_PNT_AGN_START AS bigint)
      AND TRY_CAST(C.DCX_RVRSD_PNT_AGN_END   AS bigint)
)
FROM [{Db}].[tblCompareClaimsDetails_Addtnl_1_IDs] T
WHERE T.[BulkInsertKey] = @key
  AND T.[WM_ID]        = @wm
  AND TRY_CONVERT(datetime, T.CLAIM_DOS, 121) > '1900-01-01';";

    using (var cmd = new SqlCommand(updateB, _sql))
    {
        cmd.Parameters.AddWithValue("@key", _bulkInsertKey);
        cmd.Parameters.AddWithValue("@wm",  _wmId);
        cmd.ExecuteNonQuery();
    }
}

// helper: validate and bracket a multipart identifier like dbo.MyTable_123
private static string SafeIdent(string name)
{
    if (string.IsNullOrWhiteSpace(name))
        throw new InvalidOperationException("Temp table name is blank.");

    // allow letters, digits, underscore, and dot (for schema.table)
    foreach (char c in name)
        if (!(char.IsLetterOrDigit(c) || c == '_' || c == '.'))
            throw new InvalidOperationException($"Bad table name: {name}");

    // bracket each part to survive underscores/numbers/reserved words
    return string.Join(".", name.Split('.').Select(p => $"[{p}]"));
}

private void StageClaimRows()
{
    // _tempTableName, _sql (SqlConnection), _wmId, _bulkInsertKey are existing members
    var src = SafeIdent(_tempTableName);   // e.g. [dbo].[tempCompare_tblCompareTemp_BI_202509011750_VM]

    string sql = $@"
IF OBJECT_ID('tempdb..#EntityTbl') IS NOT NULL DROP TABLE #EntityTbl;
CREATE TABLE #EntityTbl
(
    WM_ID            int,
    CLAIM_ID         varchar(30),
    COMPARE_CODE     varchar(20),
    CLAIM_DOS        varchar(10),
    PHARMACY_ID      varchar(20),
    ClaimResponse    varchar(10),
    MEMBER_ID        varchar(30),
    PERSON_NBR       varchar(3),
    RVSVD_PNT_ACN    bigint NULL
);

INSERT INTO #EntityTbl (WM_ID, CLAIM_ID, COMPARE_CODE, CLAIM_DOS, PHARMACY_ID, ClaimResponse, MEMBER_ID, PERSON_NBR)
SELECT  @wm,
        CLM.CLAIM_ID,
        CLM.COMPARE_CODE,
        CLM.CLAIM_DOS,
        CLM.PHARMACY_ID,
        CLM.CLAIM_RESPONSE,
        CLM.MEMBER_ID,
        CLM.PERSON_NBR
FROM {src} AS CLM
WHERE CLM.BulkInsertKey = @key
  AND CLM.MEMBER_ID IS NOT NULL;

-- seed Addtnl_1_IDs from temp table (matches SP step)
INSERT INTO dbo.tblCompareClaimsDetails_Addtnl_1_IDs
    (WM_ID, BulkInsertKey, IM_CLAIM_ID, MEMBER_ID, PERSON_NBR, RVSVD_PNT_ACN)
SELECT  @wm, @key, CLAIM_ID, MEMBER_ID, PERSON_NBR, NULL
FROM #EntityTbl;";

    using var cmd = new SqlCommand(sql, _sql);
    cmd.Parameters.AddWithValue("@wm",  _wmId);
    cmd.Parameters.AddWithValue("@key", _bulkInsertKey);
    cmd.ExecuteNonQuery();
}

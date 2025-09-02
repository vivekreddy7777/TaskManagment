string database = "Quality_Control";

// 2) build a *safe* 3-part identifier for the temp table
// QUOTENAME will add brackets and fail on bad characters if any.
string src = $"{database}.dbo.{SqlClientSqlString.QuoteIdentifier(tempTableName)}";

// 3) create the staging temp table (#EntityTbl)
var createStageSql = @"
IF OBJECT_ID('tempdb..#EntityTbl') IS NOT NULL DROP TABLE #EntityTbl;
CREATE TABLE #EntityTbl
(
    WM_ID          INT,
    CLAIM_ID       BIGINT,
    COMPARE_CODE   VARCHAR(50),
    CLAIM_DOS      DATETIME,
    PHARMACY_ID    VARCHAR(50),
    ClaimResponse  VARCHAR(50),
    MEMBER_ID      VARCHAR(50),
    PERSON_NBR     INT,
    RVSVD_PINT_ACN BIGINT NULL
);";
using (var cmd = new SqlCommand(createStageSql, conn))
{
    cmd.ExecuteNonQuery();
}

// 4) INSERT into #EntityTbl from the **dynamic** temp table (same columns/aliases as SP)
string insertEntitySql = $@"
INSERT INTO #EntityTbl
    (WM_ID, CLAIM_ID, COMPARE_CODE, CLAIM_DOS, PHARMACY_ID, ClaimResponse, MEMBER_ID, PERSON_NBR, RVSVD_PINT_ACN)
SELECT
    @wm,                              -- WM_ID
    [Pharmacy Claim ID]     AS CLAIM_ID,
    ClaimType               AS COMPARE_CODE,
    [DOS]                   AS CLAIM_DOS,
    [Pharmacy #]            AS PHARMACY_ID,
    [Paid/Rejected]         AS ClaimResponse,
    [Member #]              AS MEMBER_ID,
    [Person NO]             AS PERSON_NBR,
    NULL                    AS RVSVD_PINT_ACN
FROM {src}
WHERE [MEMBER #] IS NOT NULL;";

using (var cmd = new SqlCommand(insertEntitySql, conn))
{
    cmd.Parameters.Add("@wm", SqlDbType.Int).Value = wmId;
    if (debug) Console.WriteLine($"[SQL] insertEntitySql from {src}");
    cmd.ExecuteNonQuery();
}

// 5) seed the Addtnl_1_IDs table exactly like the SP does right after building EntityTbl
var seedAddtnlSql = @"
INSERT INTO dbo.tblCompareClaimsDetails_Addtnl_1_IDs
    (WM_ID, BulkInsertKey, IM_CLAIM_ID, MEMBER_ID, PERSON_NBR, RVSVD_PINT_ACN)
SELECT
    @wm,
    @key,
    CLAIM_ID, MEMBER_ID, PERSON_NBR,
    NULL
FROM #EntityTbl;";

using (var cmd = new SqlCommand(seedAddtnlSql, conn))
{
    cmd.Parameters.Add("@wm",  SqlDbType.Int).Value = wmId;
    cmd.Parameters.Add("@key", SqlDbType.VarChar, 50).Value = bulkInsertKey ?? (object)DBNull.Value;
    if (debug) Console.WriteLine("[SQL] seedAddtnlSql");
    cmd.ExecuteNonQuery();
}

// 6) load CLAIM_MEMBER_INFO (matches SP “Get CLAIM_MEMBER_INFOs”)
string insertMemberSql = $@"
INSERT INTO dbo.CLAIM_MEMBER_INFO (MEMBER_ID, PERSON_NBR, PROCESSED)
SELECT DISTINCT
    [Member #], [Person NO], 0
FROM {src}
WHERE [MEMBER #] IS NOT NULL;";
using (var cmd = new SqlCommand(insertMemberSql, conn))
{
    if (debug) Console.WriteLine("[SQL] insertMemberSql");
    cmd.ExecuteNonQuery();
}

// 7) (optional) drop the temp table (it will auto-drop at session end)
using (var cmd = new SqlCommand("DROP TABLE IF EXISTS #EntityTbl;", conn))
{
    cmd.ExecuteNonQuery();
}

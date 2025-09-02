// 1) fully-qualify the temp import table
var database = "Quality_Control";
var schema   = "dbo";
var src = $"[{database}].[{schema}].[{_tempTableName}]";

// 2) create #EntityTbl (same as you have)
string createStageSql = @"
IF OBJECT_ID('tempdb..#EntityTbl') IS NOT NULL DROP TABLE #EntityTbl;
CREATE TABLE #EntityTbl
(
  WM_ID INT,
  CLAIM_ID BIGINT,
  COMPARE_CODE NVARCHAR(50),
  CLAIM_DOS DATETIME,
  PHARMACY_ID VARCHAR(50),
  ClaimResponse NVARCHAR(50),
  MEMBER_ID NVARCHAR(50),
  PERSON_NBR NVARCHAR(50),
  RVSVD_PINT_ACN BIGINT NULL
);";
using (var cmd = new SqlCommand(createStageSql, _sql)) cmd.ExecuteNonQuery();

// 3) INSERT using COALESCE fallback instead of ClaimType
string insertEntitySql = $@"
INSERT INTO #EntityTbl
(WM_ID, CLAIM_ID, COMPARE_CODE, CLAIM_DOS, PHARMACY_ID,
 ClaimResponse, MEMBER_ID, PERSON_NBR, RVSVD_PINT_ACN)
SELECT
  @wm,
  CLM.[PHARMACY CLAIM ID] AS CLAIM_ID,
  COALESCE(
      NULLIF(LTRIM(RTRIM(CLM.[ClaimType])), ''),         -- if present in some files
      NULLIF(LTRIM(RTRIM(CLM.[REC TYPE])), ''),          -- fallback 1 (exists in your table)
      NULLIF(LTRIM(RTRIM(CLM.[MDS Claim Status])), ''),  -- fallback 2 (exists in your table)
      NULL                                               -- else null
  ) AS COMPARE_CODE,
  CLM.[DOS]                 AS CLAIM_DOS,
  CLM.[PHARMACY #]          AS PHARMACY_ID,
  CLM.[Paid/Rejected]       AS ClaimResponse,
  CLM.[MEMBER #]            AS MEMBER_ID,
  CLM.[PERSON NO]           AS PERSON_NBR,
  NULL
FROM {src} AS CLM
WHERE CLM.[MEMBER #] IS NOT NULL
  AND CLM.[BulkInsertKey] = @key;";

using (var cmd = new SqlCommand(insertEntitySql, _sql))
{
    cmd.Parameters.AddWithValue("@wm",  _wmId);
    cmd.Parameters.AddWithValue("@key", _bulkInsertKey ?? (object)DBNull.Value);
    Console.WriteLine($"[SQL] insertEntitySql from {src}");
    cmd.ExecuteNonQuery();
}

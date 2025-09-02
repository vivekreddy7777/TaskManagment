string insertEntitySql = $@"
INSERT INTO #EntityTbl
(WM_ID, CLAIM_ID, COMPARE_CODE, CLAIM_DOS, PHARMACY_ID,
 ClaimResponse, MEMBER_ID, PERSON_NBR, RVSVD_PINT_ACN)
SELECT
  @wm,
  CLM.[PHARMACY CLAIM ID] AS CLAIM_ID,
  COALESCE(
      NULLIF(LTRIM(RTRIM(CLM.[REC TYPE])), ''),
      NULLIF(LTRIM(RTRIM(CLM.[MDS Claim Status])), ''),
      NULL
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

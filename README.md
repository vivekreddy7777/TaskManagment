SELECT DISTINCT
    [PHARMACY CLAIM ID]        AS CLAIM_ID,
    [ClaimType]                AS COMPARE_CODE,
    [DOS]                      AS CLAIM_DOS,
    [PHARMACY #]               AS PHARMACY_ID,
    [Paid/Rejected]            AS ClaimResponse,
    [MEMBER #]                 AS MEMBER_ID,
    [PERSON NO]                AS PERSON_NBR,
    NULL                       AS RVSVD_PTNT_ACN
FROM {src}
WHERE [MEMBER #] IS NOT NULL;";

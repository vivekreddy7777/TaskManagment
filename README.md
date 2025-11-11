private async Task StageEntityTableAsync(CancellationToken ct)
{
    if (string.IsNullOrWhiteSpace(_tempTableName))
        throw new InvalidOperationException("TempTableName is not set before StageEntityTableAsync.");

    _log.LogInformation("[PREP] Stage entity table from {TempTable}", _tempTableName);

    var sql = $@"
IF OBJECT_ID('dbo.EntityTbl', 'U') IS NOT NULL
    DROP TABLE dbo.EntityTbl;

CREATE TABLE dbo.EntityTbl
(
    CLAIM_ID        VARCHAR(50),
    COMPARE_CODE    VARCHAR(20),
    CLAIM_DOS       VARCHAR(20),
    PHARMACY_ID     VARCHAR(20),
    CLAIM_RESPONSE  VARCHAR(10),
    MEMBER_ID       VARCHAR(20),
    PERSON_NBR      VARCHAR(20),
    RVRVSD_PTNT_ACN VARCHAR(20)
);

INSERT INTO dbo.EntityTbl
(
    CLAIM_ID,
    COMPARE_CODE,
    CLAIM_DOS,
    PHARMACY_ID,
    CLAIM_RESPONSE,
    MEMBER_ID,
    PERSON_NBR,
    RVRVSD_PTNT_ACN
)
SELECT DISTINCT
    CLAIM_ID,
    COMPARE_CODE,
    CLAIM_DOS,
    PHARMACY_ID,
    ClaimResponse,
    MEMBER_ID,
    PERSON_NBR,
    RVRVSD_PTNT_ACN
FROM [Quality_Control].[dbo].[{_tempTableName}];
";

    await _context.Database.ExecuteSqlRawAsync(sql, ct);

    _log.LogInformation("[PREP] Stage entity table completed.");
}

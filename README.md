DECLARE @RulesOrClause NVARCHAR(MAX);

;WITH RulesRaw AS
(
    SELECT
        RuleBody =
            LTRIM(
                CASE
                    WHEN LEFT(LTRIM(priorityRuleSQLWhere), 5) = 'WHERE'
                        THEN SUBSTRING(LTRIM(priorityRuleSQLWhere), 6, LEN(LTRIM(priorityRuleSQLWhere)))
                    ELSE LTRIM(priorityRuleSQLWhere)
                END
            )
    FROM dbo.tbl_AutoAssign_Priority_Rules
    WHERE priorityRuleSQLWhere IS NOT NULL
      AND LTRIM(RTRIM(priorityRuleSQLWhere)) <> ''
),
Rules AS
(
    SELECT
        RuleBody =
            CASE
                WHEN LEFT(LTRIM(RuleBody), 2) = 'OR'
                    THEN LTRIM(SUBSTRING(LTRIM(RuleBody), 3, LEN(LTRIM(RuleBody))))
                WHEN LEFT(LTRIM(RuleBody), 3) = 'AND'
                    THEN LTRIM(SUBSTRING(LTRIM(RuleBody), 4, LEN(LTRIM(RuleBody))))
                ELSE RuleBody
            END
    FROM RulesRaw
)
SELECT
    @RulesOrClause = STRING_AGG(RuleBody, ' OR ')
FROM Rules;

IF @RulesOrClause IS NOT NULL AND LEN(@RulesOrClause) > 0
BEGIN
    SET @SQLWhere = @SQLWhere + CHAR(10) + ' AND ( ' + @RulesOrClause + ' )';
END

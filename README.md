DECLARE @RulesOrClause NVARCHAR(MAX);

;WITH Rules AS
(
    SELECT
        RuleBody =
            CASE
                WHEN LEFT(LTRIM(priorityRuleSQLWhere), 5) = 'WHERE'
                    THEN LTRIM(SUBSTRING(LTRIM(priorityRuleSQLWhere), 6, LEN(LTRIM(priorityRuleSQLWhere))))
                ELSE LTRIM(priorityRuleSQLWhere)
            END
    FROM dbo.tbl_AutoAssign_Priority_Rules
    WHERE priorityRuleSQLWhere IS NOT NULL
      AND LTRIM(RTRIM(priorityRuleSQLWhere)) <> ''
)
SELECT @RulesOrClause =
    STRING_AGG('(' + RuleBody + ')', ' OR ')
FROM Rules;

IF @RulesOrClause IS NOT NULL AND LEN(@RulesOrClause) > 0
BEGIN
    SET @SQLWhere = @SQLWhere + CHAR(10) + ' AND ( ' + @RulesOrClause + ' )';
END

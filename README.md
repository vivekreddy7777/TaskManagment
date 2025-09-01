private static async Task LogProcSignatureAsync(SqlConnection conn, ILogger log)
{
    const string sql = @"
SELECT DB_NAME() AS db,
       OBJECT_SCHEMA_NAME(p.object_id) AS [schema],
       OBJECT_NAME(p.object_id)        AS proc,
       p.parameter_id,
       p.name                           AS param,
       p.is_output,
       t.name                           AS type_name,
       p.max_length
FROM sys.parameters p
JOIN sys.types t ON t.user_type_id = p.user_type_id
WHERE p.object_id = OBJECT_ID(N'[Quality_Control].[dbo].[procCompareTool_ClaimBulkInsert1]')
ORDER BY p.parameter_id;";

    using var cmd = new SqlCommand(sql, conn);
    using var r = await cmd.ExecuteReaderAsync();
    var lines = new List<string>();
    while (await r.ReadAsync())
        lines.Add($"{r["db"]}.{r["schema"]}.{r["proc"]} {r["param"],-18} out={r["is_output"]} {r["type_name"]}({r["max_length"]})");
    log.LogInformation("Proc signature:\n" + string.Join("\n", lines));
}

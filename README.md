public async Task CallCompareTool(CancellationToken ct = default)
{
    using var scope = _scopeFactory.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();
    var conn = dbContext.Database.GetDbConnection();

    var mustOpen = conn.State == ConnectionState.Closed;
    if (mustOpen) await conn.OpenAsync(ct);

    try
    {
        // 1) Call first stored procedure
        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandText = "procCompareTool_ClaimBulkInsert";
            cmd.CommandType = CommandType.StoredProcedure;

            Add(cmd, "@Password",     DbType.String, Constants.Password);
            Add(cmd, "@QAM_ID",       DbType.Int64,  RequestorID);
            Add(cmd, "@Compare_ID",   DbType.Int64,  Compare_ID);
            Add(cmd, "@FileName",     DbType.String, FileName);
            Add(cmd, "@GridType",     DbType.String, GridType);
            Add(cmd, "@Additional",   DbType.String, Additional);
            Add(cmd, "@OverrideType", DbType.String, OverrideType);
            Add(cmd, "@Region",       DbType.String, Region);

            Add(cmd, "@ValidationErrors", DbType.Boolean, null, ParameterDirection.Output);
            Add(cmd, "@BulkInsertKey",    DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@valTableName",     DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@valTempTableName", DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@tempTableName",    DbType.String,  null, ParameterDirection.Output);

            await cmd.ExecuteNonQueryAsync(ct);

            ValidationErrors = GetOut(cmd, "@ValidationErrors", false);
            BulkInsertKey    = GetOut(cmd, "@BulkInsertKey",    string.Empty);
            valTableName     = GetOut(cmd, "@valTableName",     string.Empty);
            valTempTableName = GetOut(cmd, "@valTempTableName", string.Empty);
            tempTableName    = GetOut(cmd, "@tempTableName",    string.Empty);
        }

        // 2) Call your console EXE in between
        var process = new Process
        {
            StartInfo = new ProcessStartInfo
            {
                FileName = "YourConsoleApp.exe",              // path to console app
                Arguments = $"{BulkInsertKey} {valTableName}",// pass args if needed
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                UseShellExecute = false,
                CreateNoWindow = true
            }
        };

        process.Start();
        string output = await process.StandardOutput.ReadToEndAsync();
        string error  = await process.StandardError.ReadToEndAsync();
        await process.WaitForExitAsync(ct);

        _logger.LogInformation("Console output: {0}", output);
        if (!string.IsNullOrWhiteSpace(error))
            _logger.LogError("Console error: {0}", error);

        // 3) Continue with the second SP
        using (var next = conn.CreateCommand())
        {
            next.CommandText = "procCompareTool_ClaimBulkInsert2";
            next.CommandType = CommandType.StoredProcedure;

            Add(next, "@Password",     DbType.String, Constants.Password);
            Add(next, "@BulkInsertKey",DbType.String, BulkInsertKey);
            Add(next, "@QAM_ID",       DbType.Int64,  RequestorID);
            Add(next, "@Compare_ID",   DbType.Int64,  Compare_ID);
            Add(next, "@FileName",     DbType.String, FileName);
            Add(next, "@GridType",     DbType.String, GridType);
            Add(next, "@Additional",   DbType.String, Additional);
            Add(next, "@OverrideType", DbType.String, OverrideType);
            Add(next, "@Region",       DbType.String, Region);

            await next.ExecuteNonQueryAsync(ct);
        }
    }
    finally
    {
        if (mustOpen) await conn.CloseAsync();
    }
}

static void Add(DbCommand c, string name, DbType type, object value,
                ParameterDirection dir = ParameterDirection.Input)
{
    var p = c.CreateParameter();
    p.ParameterName = name;
    p.DbType        = type;
    p.Direction     = dir;
    p.Value         = value ?? DBNull.Value;
    c.Parameters.Add(p);
}

static T GetOut<T>(DbCommand c, string name, T fallback = default!)
{
    if (!c.Parameters.Contains(name)) return fallback;
    var v = c.Parameters[name].Value;
    if (v == null || v == DBNull.Value) return fallback;
    return (T)Convert.ChangeType(v, typeof(T));
}

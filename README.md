public async Task CallCompareTool(CancellationToken ct = default)
{
    // locals we’ll pass to the console and second SP
    string bulkInsertKey, requestorId, fileName, gridType, additional,
           overrideType, region, tableName, valTableName, valTempTableName,
           tempTableName, environment;

    using var scope = _scopeFactory.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();
    var conn = dbContext.Database.GetDbConnection();

    var mustOpen = conn.State == ConnectionState.Closed;
    if (mustOpen) await conn.OpenAsync(ct);

    try
    {
        // -----------------------------
        // 1) Call first stored procedure
        // -----------------------------
        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandType = CommandType.StoredProcedure;
            cmd.CommandText = "procCompareTool_ClaimBulkInsert";

            // IN params (adjust the value sources as needed)
            Add(cmd, "@Password",   DbType.String, Constants.Password);
            Add(cmd, "@WM_ID",      DbType.Int64,  RequestorID);
            Add(cmd, "@Compare_ID", DbType.Int64,  Compare_ID);
            Add(cmd, "@FileName",   DbType.String, FileName);
            Add(cmd, "@GridType",   DbType.String, GridType);
            Add(cmd, "@Additional", DbType.String, Additional);
            Add(cmd, "@OverrideType", DbType.String, OverrideType);
            Add(cmd, "@Region",     DbType.String, Region);

            // OUT params (console needs these too)
            Add(cmd, "@ValidationErrors", DbType.Boolean, null, ParameterDirection.Output);
            Add(cmd, "@BulkInsertKey",   DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@tableName",       DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@valTableName",    DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@valTempTableName",DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@tempTableName",   DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@Environment",     DbType.String,  null, ParameterDirection.Output);

            await cmd.ExecuteNonQueryAsync(ct);

            // Normalize all to strings the console expects
            bool validationErrors = ToBool(GetOut(cmd, "@ValidationErrors"));
            bulkInsertKey   = ToStr(GetOut(cmd, "@BulkInsertKey"));
            tableName       = ToStr(GetOut(cmd, "@tableName"));
            valTableName    = ToStr(GetOut(cmd, "@valTableName"));
            valTempTableName= ToStr(GetOut(cmd, "@valTempTableName"));
            tempTableName   = ToStr(GetOut(cmd, "@tempTableName"));
            environment     = ToStr(GetOut(cmd, "@Environment"));

            // These inputs also go to console (as strings)
            requestorId = Convert.ToString(RequestorID);
            fileName    = ToStr(FileName);
            gridType    = Convert.ToString(GridType);
            additional  = ToStr(Additional);
            overrideType= ToStr(OverrideType);
            region      = ToStr(Region);

            // Optional guard – console requires 12 args
            if (new[]{
                    bulkInsertKey, requestorId, fileName, gridType, additional,
                    overrideType, region, tableName, valTableName,
                    valTempTableName, tempTableName, environment
                }.Any(string.IsNullOrWhiteSpace))
            {
                throw new InvalidOperationException("One or more required arguments for console call are missing.");
            }

            // -------------------------------------
            // 2) Launch the console with 12 args
            // -------------------------------------
            var psi = new ProcessStartInfo
            {
                FileName = _consoleExePath,          // e.g. \\server\share\BTA_CompareTool_Console.exe
                UseShellExecute = false,
                RedirectStandardOutput = true,
                RedirectStandardError  = true,
                CreateNoWindow = true
            };

            // Add each arg to avoid quoting issues
            psi.ArgumentList.Add(bulkInsertKey);
            psi.ArgumentList.Add(requestorId);      // wmId
            psi.ArgumentList.Add(fileName);
            psi.ArgumentList.Add(gridType);
            psi.ArgumentList.Add(additional);
            psi.ArgumentList.Add(overrideType);
            psi.ArgumentList.Add(region);
            psi.ArgumentList.Add(tableName);
            psi.ArgumentList.Add(valTableName);
            psi.ArgumentList.Add(valTempTableName);
            psi.ArgumentList.Add(tempTableName);
            psi.ArgumentList.Add(environment);

            using var process = Process.Start(psi)!;
            string stdout = await process.StandardOutput.ReadToEndAsync();
            string stderr = await process.StandardError.ReadToEndAsync();
            await process.WaitForExitAsync(ct);

            _logger.LogInformation("Console exited {Code}. out: {Out}. err: {Err}",
                process.ExitCode, stdout, stderr);

            if (process.ExitCode != 0)
                throw new Exception($"Console failed with code {process.ExitCode}. {stderr}");

            if (validationErrors)
                _logger.LogWarning("ValidationErrors=true from first SP.");
        }

        // -----------------------------
        // 3) Call the second procedure
        // -----------------------------
        using (var next = conn.CreateCommand())
        {
            next.CommandType = CommandType.StoredProcedure;
            next.CommandText = "procCompareTool_ClaimBulkInsert2";

            Add(next, "@Password",     DbType.String, Constants.Password);
            Add(next, "@BulkInsertKey",DbType.String, bulkInsertKey);
            Add(next, "@WM_ID",        DbType.Int64,  RequestorID);
            Add(next, "@Compare_ID",   DbType.Int64,  Compare_ID);
            Add(next, "@FileName",     DbType.String, FileName);
            Add(next, "@GridType",     DbType.String, GridType);
            Add(next, "@Additional",   DbType.String, Additional);
            Add(next, "@OverrideType", DbType.String, OverrideType);
            Add(next, "@Region",       DbType.String, Region);

            await next.ExecuteNonQueryAsync(ct);
        }
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error in CallCompareTool");
        throw;
    }
    finally
    {
        if (mustOpen) await conn.CloseAsync();
    }

    // -------------- helpers --------------
    static void Add(DbCommand c, string name, DbType type, object value,
                    ParameterDirection dir = ParameterDirection.Input)
    {
        var p = c.CreateParameter();
        p.ParameterName = name;
        p.DbType = type;
        p.Direction = dir;
        p.Value = value ?? DBNull.Value;
        c.Parameters.Add(p);
    }

    static object GetOut(DbCommand c, string name)
        => c.Parameters.Contains(name) ? c.Parameters[name].Value : DBNull.Value;

    static string ToStr(object v)
        => v == null || v == DBNull.Value ? string.Empty : Convert.ToString(v);

    static bool ToBool(object v)
    {
        if (v == null || v == DBNull.Value) return false;
        if (v is bool b) return b;
        var s = Convert.ToString(v);
        return bool.TryParse(s, out var parsed) ? parsed : (s == "1");
    }
}

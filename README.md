using System;
using System.Data;
using System.Diagnostics;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

public async Task CallCompareTool(CancellationToken ct = default)
{
    using var scope   = _scopeFactory.CreateScope();
    var    dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();
    using var conn   = dbContext.Database.GetDbConnection();

    var mustOpen = conn.State == ConnectionState.Closed;
    if (mustOpen) await conn.OpenAsync(ct);

    // We'll need these across the whole method
    string bulkInsertKey, requestorId, fileName, gridType, additional,
           overrideType, region, tableName, valTableName, valTempTableName,
           tempTableName, environment;

    try
    {
        // ------------------------------------------------------------------
        // 1) FIRST STORED PROCEDURE
        // ------------------------------------------------------------------
        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandType = CommandType.StoredProcedure;
            cmd.CommandText = "procCompareTool_ClaimBulkInsert";

            // INPUTS
            Add(cmd, "@Password",   DbType.String, Constants.Password);
            Add(cmd, "@WM_ID",      DbType.Int64,  RequestorID);
            Add(cmd, "@Compare_ID", DbType.Int64,  Compare_ID);
            Add(cmd, "@FileName",   DbType.String, FileName);
            Add(cmd, "@GridType",   DbType.String, GridType);
            Add(cmd, "@Additional", DbType.String, Additional);
            Add(cmd, "@OverrideType", DbType.String, OverrideType);
            Add(cmd, "@Region",     DbType.String, Region);

            // OUTPUTS
            Add(cmd, "@ValidationErrors", DbType.Boolean, null, ParameterDirection.Output);
            Add(cmd, "@BulkInsertKey",    DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@TableName",        DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@ValTableName",     DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@ValTempTableName", DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@TempTableName",    DbType.String,  null, ParameterDirection.Output);
            Add(cmd, "@Environment",      DbType.String,  null, ParameterDirection.Output);

            await cmd.ExecuteNonQueryAsync(ct);

            // Pull OUT params and normalize
            bool   validationErrors = ToBool (GetOut(cmd, "@ValidationErrors"), false);
            bulkInsertKey           = ToStr  (GetOut(cmd, "@BulkInsertKey"));
            tableName               = ToStr  (GetOut(cmd, "@TableName"));
            valTableName            = ToStr  (GetOut(cmd, "@ValTableName"));
            valTempTableName        = ToStr  (GetOut(cmd, "@ValTempTableName"));
            tempTableName           = ToStr  (GetOut(cmd, "@TempTableName"));
            environment             = ToStr  (GetOut(cmd, "@Environment"));

            // Also normalize inputs we’ll pass to console as strings
            requestorId = RequestorID.ToString();
            fileName    = ToStr(FileName);
            gridType    = ToStr(GridType);
            additional  = ToStr(Additional);
            overrideType= ToStr(OverrideType);
            region      = ToStr(Region);

            if (validationErrors)
                _logger.LogWarning("Validation errors flagged by first SP.");
        }

        // ------------------------------------------------------------------
        // 2) CALL THE CONSOLE (12 ARGS)
        // ------------------------------------------------------------------
        var argsList = new[]
        {
            bulkInsertKey, requestorId, fileName, gridType, additional,
            overrideType, region, tableName, valTableName,
            valTempTableName, tempTableName, environment
        };

        // Optional guard — console expects 12 args, none blank
        if (argsList.Length != 12 || argsList.Any(string.IsNullOrWhiteSpace))
            throw new InvalidOperationException("One or more required arguments for console call are missing/blank.");

        var psi = new ProcessStartInfo
        {
            FileName = _consoleExePath,           // e.g. \\server\share\BTA_CompareTool_Console.exe
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError  = true,
            CreateNoWindow = true
        };

        foreach (var arg in argsList)
            psi.ArgumentList.Add(arg);

        using (var process = Process.Start(psi)!)
        {
            string stdout = await process.StandardOutput.ReadToEndAsync();
            string stderr = await process.StandardError.ReadToEndAsync();
            await process.WaitForExitAsync(ct);

            _logger.LogInformation("Console exited {Code}. OUT: {Out} ERR: {Err}",
                process.ExitCode, stdout, stderr);

            if (process.ExitCode != 0)
                throw new Exception($"Console failed with code {process.ExitCode}. {stderr}");
        }

        // ------------------------------------------------------------------
        // 3) SECOND STORED PROCEDURE
        // ------------------------------------------------------------------
        using (var next = conn.CreateCommand())
        {
            next.CommandType = CommandType.StoredProcedure;
            next.CommandText = "procCompareTool_ClaimBulkInsert2";

            Add(next, "@Password",     DbType.String, Constants.Password);
            Add(next, "@BulkInsertKey",DbType.String, bulkInsertKey);
            Add(next, "@WM_ID",        DbType.Int64,  RequestorID);
            Add(next, "@Compare_ID",   DbType.Int64,  Compare_ID);
            Add(next, "@FileName",     DbType.String, fileName);
            Add(next, "@GridType",     DbType.String, gridType);
            Add(next, "@Additional",   DbType.String, additional);
            Add(next, "@OverrideType", DbType.String, overrideType);
            Add(next, "@Region",       DbType.String, region);

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
}

/* ------------------------ helpers ------------------------ */

static void Add(IDbCommand c, string name, DbType type, object value, ParameterDirection dir = ParameterDirection.Input)
{
    var p = c.CreateParameter();
    p.ParameterName = name;
    p.DbType        = type;
    p.Direction     = dir;
    p.Value         = value ?? DBNull.Value;
    c.Parameters.Add(p);
}

static object GetOut(IDbCommand c, string name)
{
    if (!c.Parameters.Contains(name)) return DBNull.Value;
    var v = ((IDataParameter)c.Parameters[name]).Value;
    return v ?? DBNull.Value;
}

static string ToStr(object v) =>
    v == null || v == DBNull.Value ? string.Empty : Convert.ToString(v)!;

static bool ToBool(object v, bool fallback = false)
{
    if (v == null || v == DBNull.Value) return fallback;
    if (v is bool b) return b;
    if (bool.TryParse(Convert.ToString(v), out var parsed)) return parsed;
    if (v is int i) return i != 0;
    return fallback;
}

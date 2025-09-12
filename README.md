
using System;
using System.Data;
using System.Data.Common;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

public async Task ExportResults(CancellationToken ct = default)
{
    _logger.LogInformation("ExportResults started");

    using var scope = _scopeFactory.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();

    var conn = dbContext.Database.GetDbConnection();
    var mustOpen = conn.State != ConnectionState.Open;
    if (mustOpen) await conn.OpenAsync(ct);

    try
    {
        // 1) INNER PROC (execute via cmd + parameters)
        string reportName;
        string execSummary; // human-readable record for outer logging proc

        try
        {
            (reportName, execSummary) = await ExecuteInnerExportAsync(conn, ct);
            _logger.LogInformation("Inner export proc executed successfully (branch={Branch})",
                reportName);
        }
        catch (SqlException ex)
        {
            _logger.LogError(ex, "SQL error in inner export proc. Number={Number}, Message={Message}", ex.Number, ex.Message);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error in inner export proc: {Message}", ex.Message);
            throw;
        }

        // 2) OUTER PROC (via cmd + parameters, same order as screenshots)
        try
        {
            await ExecuteOuterLogAsync(conn, reportName, execSummary, ct);
            _logger.LogInformation("Outer logging proc executed successfully.");
        }
        catch (SqlException ex)
        {
            _logger.LogError(ex, "SQL error in outer log proc. Number={Number}, Message={Message}", ex.Number, ex.Message);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error in outer log proc: {Message}", ex.Message);
            throw;
        }

        // If you have a follow-up, call it here (ensure it's a method or delegate)
        // FollowUp?.Invoke();
    }
    catch (SqlException ex)
    {
        _logger.LogError(ex, "SQL error in ExportResults. Number={Number}, Message={Message}", ex.Number, ex.Message);
        throw;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unexpected error in ExportResults: {Message}", ex.Message);
        throw;
    }
    finally
    {
        if (mustOpen) await conn.CloseAsync();
        _logger.LogInformation("ExportResults End");
    }

    // ===== helpers =====

    async Task<(string reportName, string executedSummary)> ExecuteInnerExportAsync(DbConnection connection, CancellationToken token)
    {
        using var cmd = connection.CreateCommand();
        cmd.CommandType = CommandType.StoredProcedure;

        if (ValidationErrors)
        {
            cmd.CommandText = "dbo.procCompareTool_ClaimExport_Errors";

            var pWm = cmd.CreateParameter();
            pWm.ParameterName = "@WM_ID";
            pWm.DbType = DbType.Int32;
            pWm.Value = Compare_ID;
            cmd.Parameters.Add(pWm);

            var pBulk = cmd.CreateParameter();
            pBulk.ParameterName = "@BulkInsertKey";
            pBulk.DbType = DbType.String;
            pBulk.Value = (object?)BulkInsertKey ?? DBNull.Value;
            cmd.Parameters.Add(pBulk);

            // debug params
            foreach (DbParameter p in cmd.Parameters)
                _logger.LogDebug("Inner Param: {Name}={Value} ({Type})", p.ParameterName, p.Value ?? "NULL", p.DbType);

            await cmd.ExecuteNonQueryAsync(token);

            // readable audit text for outer proc
            var summary = $"EXEC dbo.procCompareTool_ClaimExport_Errors @WM_ID={Compare_ID}, @BulkInsertKey='{BulkInsertKey}'";
            return ("CompareErrors", summary);
        }
        else
        {
            cmd.CommandText = "dbo.procCompareTool_ClaimExport_ALTERNATE";

            var pWm = cmd.CreateParameter();
            pWm.ParameterName = "@WM_ID";
            pWm.DbType = DbType.Int32;
            pWm.Value = Compare_ID;
            cmd.Parameters.Add(pWm);

            var pType = cmd.CreateParameter();
            pType.ParameterName = "@Type";
            pType.DbType = DbType.String;
            pType.Value = (object?)ExportRequestType ?? DBNull.Value;
            cmd.Parameters.Add(pType);

            foreach (DbParameter p in cmd.Parameters)
                _logger.LogDebug("Inner Param: {Name}={Value} ({Type})", p.ParameterName, p.Value ?? "NULL", p.DbType);

            await cmd.ExecuteNonQueryAsync(token);

            var summary = $"EXEC dbo.procCompareTool_ClaimExport_ALTERNATE @WM_ID={Compare_ID}, @Type='{ExportRequestType}'";
            return ("CompareAlternate", summary);
        }
    }

    async Task ExecuteOuterLogAsync(DbConnection connection, string reportName, string execSummary, CancellationToken token)
    {
        using var cmdLog = connection.CreateCommand();
        cmdLog.CommandType = CommandType.StoredProcedure;
        cmdLog.CommandText = "dbo.procBTE_Export_MainRequest_INSERT";

        DbParameter Add(string name, DbType type, object? value, int size = 0, ParameterDirection dir = ParameterDirection.Input)
        {
            var p = cmdLog.CreateParameter();
            p.ParameterName = name;
            p.DbType = type;
            p.Direction = dir;
            if (size > 0 && p is System.Data.Common.DbParameter dp) dp.Size = size;
            p.Value = value ?? DBNull.Value;
            cmdLog.Parameters.Add(p);
            _logger.LogDebug("Outer Param: {Name}={Value} ({Type}, {Dir})", name, p.Value ?? "NULL", p.DbType, p.Direction);
            return p;
        }

        // SAME ORDER AS YOUR SCREENSHOTS
        Add("@Password",        DbType.String,  Constants.Password, 50);
        Add("@UserId",          DbType.Int32,   RequestorID);
        Add("@Report_Name",     DbType.String,  reportName, 100);
        Add("@ExecOutput",      DbType.Boolean, ExecOutput);
        Add("@PasswordProtect", DbType.Boolean, PasswordProtect);
        Add("@FileName_Attr",   DbType.String,  FileNameAttr + "_" + Constants.CentralTime.ToString("yyyyMMddHmmssFFF"), 260);
        Add("@ExecSQL",         DbType.String,  execSummary, int.MaxValue);
        Add("@DataSource",      DbType.String,  QCoreDb, 50);

        var pOut = Add("@ProcResult", DbType.Int32, null, 0, ParameterDirection.Output);

        await cmdLog.ExecuteNonQueryAsync(token);

        _logger.LogInformation("procBTE_Export_MainRequest_INSERT finished. ProcResult={Result}",
            pOut.Value ?? -1);
    }
}

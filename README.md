
public async Task ExportResults(CancellationToken ct = default)
{
    _logger.LogInformation("ExportResults started");
    using var scope = _scopeFactory.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();

    // Reuse EF connection
    var conn = dbContext.Database.GetDbConnection();
    var mustOpen = conn.State != ConnectionState.Open;
    if (mustOpen) await conn.OpenAsync(ct);

    try
    {
        string reportName;
        string innerProcName;

        // -------------------------------
        // 1) INNER PROC (Errors/Alternate)
        // -------------------------------
        var swInner = Stopwatch.StartNew();

        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandType = CommandType.StoredProcedure;

            if (ValidationErrors)
            {
                reportName   = "CompareErrors";
                innerProcName = "dbo.procCompareTool_ClaimExport_Errors";
                cmd.CommandText = innerProcName;

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

                _logger.LogDebug("{Proc} params: @WM_ID={WM}, @BulkInsertKey={Bulk}",
                    innerProcName, Compare_ID, BulkInsertKey);
            }
            else
            {
                reportName   = "CompareAlternate";
                innerProcName = "dbo.procCompareTool_ClaimExport_ALTERNATE";
                cmd.CommandText = innerProcName;

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

                _logger.LogDebug("{Proc} params: @WM_ID={WM}, @Type={Type}",
                    innerProcName, Compare_ID, ExportRequestType);
            }

            await cmd.ExecuteNonQueryAsync(ct);
        }

        swInner.Stop();
        _logger.LogInformation("Inner export proc {Proc} completed in {Ms} ms.",
            ValidationErrors ? "procCompareTool_ClaimExport_Errors" : "procCompareTool_ClaimExport_ALTERNATE",
            swInner.ElapsedMilliseconds);

        // For audit trail in the outer proc, store a readable summary of what ran
        var executedSummary = ValidationErrors
            ? $"EXEC dbo.procCompareTool_ClaimExport_Errors @WM_ID={Compare_ID}, @BulkInsertKey='{BulkInsertKey}'"
            : $"EXEC dbo.procCompareTool_ClaimExport_ALTERNATE @WM_ID={Compare_ID}, @Type='{ExportRequestType}'";

        // ----------------------------------------------------------
        // 2) OUTER PROC: procBTE_Export_MainRequest_INSERT (same order)
        // ----------------------------------------------------------
        var swOuter = Stopwatch.StartNew();

        using (var cmdLog = conn.CreateCommand())
        {
            cmdLog.CommandType = CommandType.StoredProcedure;
            cmdLog.CommandText = "dbo.procBTE_Export_MainRequest_INSERT";

            // helper to reduce boilerplate
            IDbDataParameter Add(string name, DbType type, object? value, int size = 0, ParameterDirection dir = ParameterDirection.Input)
            {
                var p = cmdLog.CreateParameter();
                p.ParameterName = name;
                p.DbType = type;
                p.Direction = dir;
                if (size > 0 && p is System.Data.Common.DbParameter dp) dp.Size = size;
                p.Value = value ?? DBNull.Value;
                cmdLog.Parameters.Add(p);
                _logger.LogDebug("LogProc Param {Name}={Value} ({Type}, {Dir})", name, p.Value, p.DbType, p.Direction);
                return p;
            }

            // SAME ORDER AS YOUR SCREENSHOT:
            Add("@Password",        DbType.String,  Constants.Password, 50);
            Add("@UserId",          DbType.Int32,   RequestorID);
            Add("@Report_Name",     DbType.String,  reportName, 100);
            Add("@ExecOutput",      DbType.Boolean, ExecOutput);
            Add("@PasswordProtect", DbType.Boolean, PasswordProtect);
            Add("@FileName_Attr",   DbType.String,  FileNameAttr + "_" + Constants.CentralTime.ToString("yyyyMMddHmmssFFF"), 260);
            Add("@ExecSQL",         DbType.String,  executedSummary, int.MaxValue); // for audit trail
            Add("@DataSource",      DbType.String,  QCoreDb, 50);

            var pOut = Add("@ProcResult", DbType.Int32, null, 0, ParameterDirection.Output);

            await cmdLog.ExecuteNonQueryAsync(ct);

            _logger.LogInformation("Outer proc {Proc} completed in {Ms} ms. ProcResult={Result}",
                cmdLog.CommandText, swOuter.ElapsedMilliseconds, pOut.Value ?? -1);
        }

        swOuter.Stop();

        // Continue with your flow
        Run();
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
}

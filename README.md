public async Task ExportResults()
{
    _logger.LogInformation("ExportResults started");

    try
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<DBServerContext>();
        LogStatus("Call Export Service");

        // Keep a human-readable summary of what we executed for audit/logging
        string executedInnerProc;
        string reportName;

        using (var conn = new SqlConnection(Constants.ConnectionString))
        {
            await conn.OpenAsync();

            // ------------------------------------------------------------
            // 1) Execute the INNER export proc with parameters (no ExecSQL text)
            // ------------------------------------------------------------
            using (var cmd = conn.CreateCommand())
            {
                cmd.CommandType = CommandType.StoredProcedure;

                if (ValidationErrors)
                {
                    reportName = "CompareErrors";
                    cmd.CommandText = "dbo.procCompareTool_ClaimExport_Errors";

                    cmd.Parameters.Add(new SqlParameter("@WM_ID", SqlDbType.Int) { Value = Compare_ID });
                    cmd.Parameters.Add(new SqlParameter("@BulkInsertKey", SqlDbType.VarChar, 100)
                    {
                        Value = (object?)BulkInsertKey ?? DBNull.Value
                    });

                    executedInnerProc = $"{cmd.CommandText} @WM_ID={Compare_ID}, @BulkInsertKey='{BulkInsertKey}'";
                    _logger.LogDebug("Executing {Proc} with @WM_ID={WM_ID}, @BulkInsertKey={BulkInsertKey}",
                        cmd.CommandText, Compare_ID, BulkInsertKey);
                }
                else
                {
                    reportName = "CompareAlternate";
                    cmd.CommandText = "dbo.procCompareTool_ClaimExport_ALTERNATE";

                    cmd.Parameters.Add(new SqlParameter("@WM_ID", SqlDbType.Int) { Value = Compare_ID });
                    cmd.Parameters.Add(new SqlParameter("@Type", SqlDbType.VarChar, 50)
                    {
                        Value = (object?)ExportRequestType ?? DBNull.Value
                    });

                    executedInnerProc = $"{cmd.CommandText} @WM_ID={Compare_ID}, @Type='{ExportRequestType}'";
                    _logger.LogDebug("Executing {Proc} with @WM_ID={WM_ID}, @Type={Type}",
                        cmd.CommandText, Compare_ID, ExportRequestType);
                }

                var swInner = Stopwatch.StartNew();
                await cmd.ExecuteNonQueryAsync();
                swInner.Stop();
                _logger.LogInformation("Inner export proc {Proc} completed in {Ms} ms.",
                    cmd.CommandText, swInner.ElapsedMilliseconds);
            }

            // ------------------------------------------------------------
            // 2) Log/queue the export via the OUTER proc (same order as screenshot)
            //    Password, UserId, Report_Name, ExecOutput, PasswordProtect,
            //    FileName_Attr, ExecSQL, DataSource, ProcResult (OUTPUT)
            // ------------------------------------------------------------
            using (var cmdLog = conn.CreateCommand())
            {
                cmdLog.CommandType = CommandType.StoredProcedure;
                cmdLog.CommandText = "dbo.procBTE_Export_MainRequest_INSERT";

                // helper to add and debug-log
                void Add(string name, SqlDbType type, object? value, int size = 0, ParameterDirection dir = ParameterDirection.Input)
                {
                    var p = size > 0 ? new SqlParameter(name, type, size) : new SqlParameter(name, type);
                    p.Direction = dir;
                    p.Value = value ?? DBNull.Value;
                    cmdLog.Parameters.Add(p);
                    _logger.LogDebug("SP Param: {Name} = {Value} (Type={Type}, Dir={Dir})",
                        name, p.Value, p.SqlDbType, p.Direction);
                }

                // SAME ORDER AS YOUR SCREENSHOT
                Add("@Password",        SqlDbType.VarChar,  Constants.Password, 50);
                Add("@UserId",          SqlDbType.Int,      RequestorID);
                Add("@Report_Name",     SqlDbType.NVarChar, reportName, 100);
                Add("@ExecOutput",      SqlDbType.Bit,      ExecOutput);
                Add("@PasswordProtect", SqlDbType.Bit,      PasswordProtect);
                Add("@FileName_Attr",   SqlDbType.NVarChar, FileNameAttr + "_" + Constants.CentralTime.ToString("yyyyMMddHmmssFFF"), 260);

                // For auditing, store exactly what we executed above
                Add("@ExecSQL",         SqlDbType.NVarChar, executedInnerProc, -1);
                Add("@DataSource",      SqlDbType.VarChar,  QCoreDb, 50);

                var procResult = new SqlParameter("@ProcResult", SqlDbType.Int)
                {
                    Direction = ParameterDirection.Output
                };
                cmdLog.Parameters.Add(procResult);

                var swLog = Stopwatch.StartNew();
                await cmdLog.ExecuteNonQueryAsync();
                swLog.Stop();

                _logger.LogInformation(
                    "Logged export via {Proc} in {Ms} ms. ProcResult={Result}",
                    cmdLog.CommandText, swLog.ElapsedMilliseconds, procResult.Value ?? -1);
            }
        }

        // Continue original flow
        Run();
    }
    catch (SqlException sqlEx)
    {
        _logger.LogError(sqlEx, "SQL error in ExportResults. Number={Number}, Message={Message}",
            sqlEx.Number, sqlEx.Message);
        throw;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unexpected error in ExportResults: {Message}", ex.Message);
        throw;
    }
    finally
    {
        _logger.LogInformation("ExportResults End");
    }
}

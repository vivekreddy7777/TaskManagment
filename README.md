public async Task ExportResults(CancellationToken ct = default)
{
    _logger.LogInformation("ExportResults started");

    using var scope = _scopeFactory.CreateScope();

    // Core DB (outer proc)
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();
    var conn = dbContext.Database.GetDbConnection();
    if (conn.State != ConnectionState.Open)
        await conn.OpenAsync(ct);

    // QC DB (inner procs)
    var qcdbContext = scope.ServiceProvider.GetRequiredService<QCDBServerContext>();
    var qconn = qcdbContext.Database.GetDbConnection();
    if (qconn.State != ConnectionState.Open)
        await qconn.OpenAsync(ct);

    try
    {
        string reportName;
        string execSummary;

        // -------------------------
        // Inner proc on QC DB
        // -------------------------
        using (var cmd = (SqlCommand)qconn.CreateCommand())
        {
            cmd.CommandType = CommandType.StoredProcedure;

            if (ValidationErrors)
            {
                reportName = "CompareErrors";
                cmd.CommandText = "dbo.procCompareTool_ClaimExport_Errors";

                cmd.Parameters.Add(new SqlParameter("@WM_ID", SqlDbType.VarChar, 50)
                { Value = (object)Compare_ID ?? DBNull.Value });

                cmd.Parameters.Add(new SqlParameter("@BulkInsertKey", SqlDbType.VarChar, 100)
                { Value = (object)BulkInsertKey ?? DBNull.Value });

                await cmd.ExecuteNonQueryAsync(ct);

                execSummary = $"EXEC dbo.procCompareTool_ClaimExport_Errors @WM_ID='{Compare_ID}', @BulkInsertKey='{BulkInsertKey}'";
                _logger.LogInformation("QC inner proc executed: {Proc}", cmd.CommandText);
            }
            else
            {
                reportName = "CompareAlternate";
                cmd.CommandText = "dbo.procCompareTool_ClaimExport_ALTERNATE";

                cmd.Parameters.Add(new SqlParameter("@WM_ID", SqlDbType.VarChar, 50)
                { Value = (object)Compare_ID ?? DBNull.Value });

                cmd.Parameters.Add(new SqlParameter("@Type", SqlDbType.VarChar, 50)
                { Value = (object)ExportRequestType ?? DBNull.Value });

                await cmd.ExecuteNonQueryAsync(ct);

                execSummary = $"EXEC dbo.procCompareTool_ClaimExport_ALTERNATE @WM_ID='{Compare_ID}', @Type='{ExportRequestType}'";
                _logger.LogInformation("QC inner proc executed: {Proc}", cmd.CommandText);
            }
        }

        // -------------------------
        // Outer proc on Core DB
        // -------------------------
        using (var cmdLog = (SqlCommand)conn.CreateCommand())
        {
            cmdLog.CommandType = CommandType.StoredProcedure;
            cmdLog.CommandText = "dbo.procBTE_Export_MainRequest_INSERT";

            cmdLog.Parameters.Add(new SqlParameter("@Password", SqlDbType.VarChar, 50)
            { Value = (object)Constants.Password ?? DBNull.Value });

            cmdLog.Parameters.Add(new SqlParameter("@UserId", SqlDbType.Int)
            { Value = RequestorID });

            cmdLog.Parameters.Add(new SqlParameter("@Report_Name", SqlDbType.VarChar, 100)
            { Value = (object)reportName ?? DBNull.Value });

            cmdLog.Parameters.Add(new SqlParameter("@ExecOutput", SqlDbType.Bit)
            { Value = ExecOutput });

            cmdLog.Parameters.Add(new SqlParameter("@PasswordProtect", SqlDbType.Bit)
            { Value = PasswordProtect });

            cmdLog.Parameters.Add(new SqlParameter("@FileName_Attr", SqlDbType.VarChar, 260)
            { Value = (object)(FileNameAttr + "_" + Constants.CentralTime.ToString("yyyyMMddHmmssFFF")) ?? DBNull.Value });

            cmdLog.Parameters.Add(new SqlParameter("@ExecSQL", SqlDbType.VarChar, -1) // -1 = VARCHAR(MAX)
            { Value = (object)execSummary ?? DBNull.Value });

            cmdLog.Parameters.Add(new SqlParameter("@DataSource", SqlDbType.VarChar, 50)
            { Value = (object)QCoreDb ?? DBNull.Value });

            var outParam = new SqlParameter("@ProcResult", SqlDbType.Int)
            { Direction = ParameterDirection.Output };
            cmdLog.Parameters.Add(outParam);

            await cmdLog.ExecuteNonQueryAsync(ct);

            _logger.LogInformation("Core outer proc executed. ProcResult={Result}", outParam.Value ?? -1);
        }
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
        if (qconn.State == ConnectionState.Open) await qconn.CloseAsync();
        if (conn.State == ConnectionState.Open) await conn.CloseAsync();
        _logger.LogInformation("ExportResults End");
    }
}

using System;
using System.Data;
using System.Data.SqlClient;

public void ExportResults()
{
    _logger.LogInformation("ExportResults started");

    // 1) open QC connection for the inner export
    using (var qconn = new SqlConnection(Constants.QCConnectionString))   // <-- QC DB
    // 2) open Core connection for the outer logging insert
    using (var conn  = new SqlConnection(Constants.ConnectionString))     // <-- Core/App DB
    {
        try
        {
            qconn.Open();
            conn.Open();

            string reportName;
            string execSummary; // human-readable text for logging only

            // -------------------------------
            // INNER PROC on qconn (QC)
            // -------------------------------
            using (var cmd = new SqlCommand())
            {
                cmd.Connection  = qconn;
                cmd.CommandType = CommandType.StoredProcedure;

                if (ValidationErrors)
                {
                    reportName     = "CompareErrors";
                    cmd.CommandText = "dbo.procCompareTool_ClaimExport_Errors";

                    // @WM_ID looks alphanumeric -> NVARCHAR
                    cmd.Parameters.Add(new SqlParameter("@WM_ID", SqlDbType.NVarChar, 50)
                    { Value = (object)Compare_ID ?? DBNull.Value });

                    cmd.Parameters.Add(new SqlParameter("@BulkInsertKey", SqlDbType.NVarChar, 100)
                    { Value = (object)BulkInsertKey ?? DBNull.Value });

                    _logger.LogDebug("QC exec {Proc}: @WM_ID={WM_ID}, @BulkInsertKey={BulkKey}",
                        cmd.CommandText, Compare_ID, BulkInsertKey);

                    cmd.ExecuteNonQuery();

                    execSummary = string.Format(
                        "EXEC dbo.procCompareTool_ClaimExport_Errors @WM_ID='{0}', @BulkInsertKey='{1}'",
                        Compare_ID, BulkInsertKey);
                }
                else
                {
                    reportName     = "CompareAlternate";
                    cmd.CommandText = "dbo.procCompareTool_ClaimExport_ALTERNATE";

                    cmd.Parameters.Add(new SqlParameter("@WM_ID", SqlDbType.NVarChar, 50)
                    { Value = (object)Compare_ID ?? DBNull.Value });

                    cmd.Parameters.Add(new SqlParameter("@Type", SqlDbType.NVarChar, 50)
                    { Value = (object)ExportRequestType ?? DBNull.Value });

                    _logger.LogDebug("QC exec {Proc}: @WM_ID={WM_ID}, @Type={Type}",
                        cmd.CommandText, Compare_ID, ExportRequestType);

                    cmd.ExecuteNonQuery();

                    execSummary = string.Format(
                        "EXEC dbo.procCompareTool_ClaimExport_ALTERNATE @WM_ID='{0}', @Type='{1}'",
                        Compare_ID, ExportRequestType);
                }

                _logger.LogInformation("Inner export (QC) succeeded. Report={Report}", reportName);

                // -------------------------------
                // OUTER PROC on conn (Core/App)
                // -------------------------------
                using (var cmdLog = new SqlCommand("dbo.procBTE_Export_MainRequest_INSERT", conn))
                {
                    cmdLog.CommandType = CommandType.StoredProcedure;

                    // keep parameter order same as your original code/screenshots
                    cmdLog.Parameters.Add(new SqlParameter("@Password", SqlDbType.VarChar, 50)
                    { Value = (object)Constants.Password ?? DBNull.Value });

                    cmdLog.Parameters.Add(new SqlParameter("@UserId", SqlDbType.Int)
                    { Value = RequestorID });

                    cmdLog.Parameters.Add(new SqlParameter("@Report_Name", SqlDbType.NVarChar, 100)
                    { Value = (object)reportName ?? DBNull.Value });

                    cmdLog.Parameters.Add(new SqlParameter("@ExecOutput", SqlDbType.Bit)
                    { Value = ExecOutput });

                    cmdLog.Parameters.Add(new SqlParameter("@PasswordProtect", SqlDbType.Bit)
                    { Value = PasswordProtect });

                    cmdLog.Parameters.Add(new SqlParameter("@FileName_Attr", SqlDbType.NVarChar, 260)
                    { Value = (object)(FileNameAttr + "_" + Constants.CentralTime.ToString("yyyyMMddHmmssFFF")) ?? DBNull.Value });

                    // store the readable inner call text for auditing
                    cmdLog.Parameters.Add(new SqlParameter("@ExecSQL", SqlDbType.NVarChar, -1)
                    { Value = (object)execSummary ?? DBNull.Value });

                    cmdLog.Parameters.Add(new SqlParameter("@DataSource", SqlDbType.VarChar, 50)
                    { Value = (object)QCoreDb ?? DBNull.Value });

                    var outParam = new SqlParameter("@ProcResult", SqlDbType.Int)
                    { Direction = ParameterDirection.Output };
                    cmdLog.Parameters.Add(outParam);

                    foreach (SqlParameter p in cmdLog.Parameters)
                        _logger.LogDebug("Core param {Name}={Value} (Type={Type}, Dir={Dir})",
                            p.ParameterName, p.Value ?? "NULL", p.SqlDbType, p.Direction);

                    cmdLog.ExecuteNonQuery();

                    _logger.LogInformation("Logged export via procBTE_Export_MainRequest_INSERT. ProcResult={Result}",
                        outParam.Value ?? -1);
                }
            }
        }
        catch (SqlException ex)
        {
            _logger.LogError(ex, "SQL error in ExportResults. Number={Number}, Message={Message}",
                ex.Number, ex.Message);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error in ExportResults: {Message}", ex.Message);
            throw;
        }
        finally
        {
            if (qconn.State == ConnectionState.Open) qconn.Close();
            if (conn.State  == ConnectionState.Open) conn.Close();
            _logger.LogInformation("ExportResults End");
        }
    }
}

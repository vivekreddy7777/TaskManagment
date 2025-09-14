using (var cmdLog = connection.CreateCommand())
{
    cmdLog.CommandType = CommandType.StoredProcedure;
    cmdLog.CommandText = "dbo.procBTE_Export_MainRequest_INSERT";

    DbParameter Add(string name, DbType type, object? value,
        int size = 0, ParameterDirection dir = ParameterDirection.Input)
    {
        var p = cmdLog.CreateParameter();
        p.ParameterName = name;
        p.DbType = type;
        p.Direction = dir;
        if (size > 0 && p is System.Data.Common.DbParameter dp) dp.Size = size;
        p.Value = value ?? DBNull.Value;
        cmdLog.Parameters.Add(p);

        _logger.LogDebug("Outer Param: {Name}={Value} ({Type}, {Dir})",
            name, p.Value ?? "NULL", p.DbType, p.Direction);

        return p;
    }

    // Required / input parameters
    Add("@Password", DbType.String, Constants.Password, 20);
    Add("@Debug", DbType.Boolean, false);
    Add("@PrintSQL", DbType.Boolean, false);
    Add("@UserId", DbType.String, RequestorID, 50);
    Add("@Report_Name", DbType.String, reportName, 50);
    Add("@ExcelOutput", DbType.Boolean, ExcelOutput);
    Add("@PasswordProtect", DbType.Boolean, PasswordProtect);
    Add("@ExcelTabName", DbType.String, "Output", 50);
    Add("@FileName_Attr", DbType.String,
        FileName_Attr + "_" + Constants.CentralTime.ToString("yyyyMMddHHmmssFFF"), 260);
    Add("@ExecSQL", DbType.String, execSummary, int.MaxValue);
    Add("@DataSource", DbType.String, _CoreDb, 50);
    Add("@Priority", DbType.Int32, null);
    Add("@RunDate", DbType.String, DateTime.Now.ToString("yyyy-MM-dd"), 30);
    Add("@IsDLreportemail", DbType.Boolean, false);
    Add("@ReportEmail", DbType.String, "", 260);

    // Output parameters
    var procResult = Add("@ProcResult", DbType.String, null, int.MaxValue, ParameterDirection.Output);
    var requestNumber = Add("@Request_Number", DbType.Int64, null, 0, ParameterDirection.Output);

    await cmdLog.ExecuteNonQueryAsync(token);

    _logger.LogInformation("BTE Export Result: ProcResult={Proc}, Request_Number={Req}",
        procResult.Value, requestNumber.Value);
}

protected async Task<DataTable> GetMainRequests(string procName, CancellationToken ct = default)
{
    using var scope     = _scopeFactory.CreateScope();
    var dbContext       = scope.ServiceProvider.GetRequiredService<DBServerContext>();
    var conn            = (SqlConnection)dbContext.Database.GetDbConnection();

    var mustClose = conn.State != ConnectionState.Open;
    if (mustClose) await conn.OpenAsync(ct);

    try
    {
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = procName;                    // e.g. "procBTE_CompareTool_MainRequest_SELECT"
        cmd.CommandType = CommandType.StoredProcedure;

        // --- parameters (typed; avoids AddWithValue pitfalls) ---
        cmd.Parameters.Add("@Password",       SqlDbType.NVarChar, 50).Value = Constants.Password;
        cmd.Parameters.Add("@Request_Number", SqlDbType.BigInt).Value       = Request_Number;
        cmd.Parameters.Add("@UserID",         SqlDbType.NVarChar, 50).Value = UserID;

        // Optional OUTPUT param (only if your proc defines it)
        var procResultParam = cmd.Parameters.Add("@ProcResult", SqlDbType.Int);
        procResultParam.Direction = ParameterDirection.Output;

        // --- execute and load to DataTable ---
        await using var reader = await cmd.ExecuteReaderAsync(ct);
        var table = new DataTable();
        table.Load(reader);

        // Example: log the output value if present
        if (procResultParam.Value != DBNull.Value)
        {
            var procResult = Convert.ToInt32(procResultParam.Value);
            _logger.LogInformation("GetMainRequests: ProcResult={ProcResult}, Rows={Rows}", procResult, table.Rows.Count);
        }

        return table;
    }
    finally
    {
        if (mustClose) await conn.CloseAsync();
    }
}

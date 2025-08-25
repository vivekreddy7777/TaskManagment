private void SetTraceOutput()
{
    // Delete any previous log file for this module
    var logFilePath = Path.Combine("C:\\Logs", $"{_module}_{DateTime.Now:yyyyMMdd}.log");
    if (File.Exists(logFilePath))
        File.Delete(logFilePath);

    // Create new log file
    using var writer = File.AppendText(logFilePath);
    writer.WriteLine($"[{DateTime.Now}] Logging started for module {_module}");

    // Keep reference if you want to write manually later
    _logger.LogInformation("Logging started, file: {path}", logFilePath);
}
private async Task<bool> OkToStartAsync(SqlConnection conn, string module, string table, CancellationToken ct)
{
    using var cmd = new SqlCommand("FuncBTE_ServiceBroker_OkToStart", conn)
    {
        CommandType = CommandType.StoredProcedure
    };
    cmd.Parameters.AddWithValue("@Module", module);
    cmd.Parameters.AddWithValue("@ModuleName", table);

    var dt = new DataTable();
    using (var da = new SqlDataAdapter(cmd))
    {
        da.Fill(dt);
    }

    if (dt.Rows.Count == 0)
        return true;

    var reason = dt.Rows[0]["Reason"]?.ToString();
    if (!string.IsNullOrEmpty(reason))
    {
        _logger.LogWarning("Not ready to start: {reason}", reason);
        return false;
    }

    return true;
}

private async Task DebugDb2WhichColumnAsync(string sql, CancellationToken ct)
{
    await using var conn = new DB2Connection(_db2Connection.ConnectionString);
    await conn.OpenAsync(ct);

    await using var cmd = conn.CreateCommand();
    cmd.CommandText = sql;

    await using var reader = await cmd.ExecuteReaderAsync(ct);

    if (!await reader.ReadAsync(ct))
    {
        _log.LogWarning("[DB2 DEBUG] No rows returned.");
        return;
    }

    for (int i = 0; i < reader.FieldCount; i++)
    {
        var name = reader.GetName(i);
        var db2Type = reader.GetDataTypeName(i);

        try
        {
            // Reproduce EF’s failing call path
            var dec = reader.GetDecimal(i);

            _log.LogInformation("[DB2 DEBUG] OK DECIMAL Col[{Idx}] {Name} ({Type}) = {Val}",
                i, name, db2Type, dec);
        }
        catch (Exception ex)
        {
            // Log raw value too (without forcing decimal)
            object raw;
            try { raw = reader.GetValue(i); }
            catch { raw = "<unable to read>"; }

            _log.LogError(ex,
                "[DB2 DEBUG] FAIL GetDecimal at Col[{Idx}] {Name} ({Type}). Raw={Raw}",
                i, name, db2Type, raw == DBNull.Value ? "NULL" : raw);

            throw; // stop at first failing column
        }
    }
}

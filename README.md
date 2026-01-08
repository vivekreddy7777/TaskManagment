private async Task DebugDb2WhichColumnAsync(string sql, CancellationToken ct)
{
    await using var conn = new DB2Connection(_db2ConnectionString);
    await conn.OpenAsync(ct);

    await using var cmd = conn.CreateCommand();
    cmd.CommandText = sql;

    await using var reader = await cmd.ExecuteReaderAsync(ct);

    while (await reader.ReadAsync(ct))
    {
        for (int i = 0; i < reader.FieldCount; i++)
        {
            string colName = reader.GetName(i);
            string db2Type = reader.GetDataTypeName(i);

            try
            {
                // IMPORTANT: GetValue reads the raw value without forcing decimal conversion
                object val = reader.GetValue(i);

                // Optional: if you want to reproduce exact failing path, try decimal conversion:
                // var d = reader.GetDecimal(i);

                _log.LogDebug("OK col[{Idx}] {Name} ({Type}) = {Val}",
                    i, colName, db2Type, val == DBNull.Value ? "NULL" : val);
            }
            catch (Exception ex)
            {
                _log.LogError(ex,
                    "FAIL col[{Idx}] {Name} ({Type})",
                    i, colName, db2Type);

                throw; // stop at first failing column
            }
        }

        break; // one row is enough to identify the column
    }
}
